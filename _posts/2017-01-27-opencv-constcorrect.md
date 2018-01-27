---
layout: post
image: images/thomas-kelley-64329.jpg  # Photo by Thomas Kelley on Unsplash
title: "OpenCV and const-correctness"
excerpt: "Make cv::Mat const again!"
tags: 
  - OpenCV
  - const-correctness
  - C++11
---


Lately I've started getting curious about functional programming (FP). It seems to offer benefits, which are too neat to ignore: 
* simpler and more concise code, 
* concurrency almost "for free", 
* easier unit-testing, 
* looser coupling between components. 

On the other hand, employing FP can seem unusual at first and requires certain discipline, especially if you use languages not specifically tailored for FP.

Purity / lack of side effects, which are usually associated with FP, are exciting concepts, but unfortunately they are often impossible or impractical to fully embrace in **real™️** applications, either due to inefficiency or due to hardware/software limitations (API calls which mutate hidden state, etc). But immutability, which is intimately connected with purity, seems to be easier to apply selectively.

Unfortunately, in many languages (C++ in particular) data is mutable by default, so we have to explicitly express our intent to keep particular objects unchanged throughout their lifetime. This practice is often referred to as **const correctness**. Basically you just sprinkle your code with `const`, and maybe even put some effort into making as much objects as possible constant (of course you should try to stay reasonable and find balance in such kind of desicions). Sounds easy, but once you start actually doing it, you end up with following benefits:
* catching occasional unintended data mutation (or even worse - logic errors due to insufficient planning)
* getting rid of race conditions and need of synchronisation (no need to sync state that doesn't change)
* potentially more optimised code (compiler will have more insight into what's going to happen)
* "self-documenting" function signatures (no need to explicitly mark input and output arguments)

Here're some great reads in case you're interested in efficient use of FP in C++, and lets move on:
* [Const Correctness, C++ FAQ](https://isocpp.org/wiki/faq/const-correctness)
* [The Functional Revolution in C++](https://bartoszmilewski.com/2014/06/09/the-functional-revolution-in-c/)
* [Efficient Pure Functional Programming in C++ Using Move Semantics](https://blog.knatten.org/2012/11/02/efficient-pure-functional-programming-in-c-using-move-semantics/)

While having few pieces of non-const correct code might be fine if properly isolated and tested, recently I've stumbled across something that I found really **disturbing**. Turns out that refcounting mechanism behind cv::Mat allows long-ranging side effects which I personally didn't expect (probably many people didn't either). 

I can't say that we haven't been warned about it, here's what [tutorial for an older OpenCV version](https://docs.opencv.org/2.4/doc/tutorials/core/mat_the_basic_image_container/mat_the_basic_image_container.html#mat) has to say about it:
> The idea is that each Mat object has its own header, however the matrix may be shared between two instance of them by having their matrix pointers point to the same address. Moreover, the copy operators will only copy the headers and the pointer to the large matrix, not the data itself. \<code snippet cut\> All the above objects, in the end, point to the same single data matrix. Their headers are different, however, and making a modification using any of them will affect all the other ones as well.

I would say that this approach is an abuse of a type system, and that it adds a great weapon to the "shoot-yourself-in-a-leg" arsenal. Simple snippet to illustrate:

```c++
#include <opencv2/opencv.hpp>

// developer which has access only to a header 
// might think that cv::Mat outer won't be changed.
void corrupt(const cv::Mat &outer) {
  // you'd expect a copy here after which inner would be decoupled from outer
  // (unless you're an opencv developer).
  // in reality they become "entangled" due to shared content, 
  // only headers are independent.
  cv::Mat inner = outer;
  
  // safe way to write this would be to clone outer to get inner above,
  // but it incurs performance penalty which it not necessary if outer
  // is a temporary object.
  // developer which works on this function has no way of knowing 
  // whether user of this function intends to use outer after calling it.
  inner = cv::Scalar(0, 0, 0);
}

int main() {
  // here we expect that cv::Mat declared as const won't be changed after loading
  const cv::Mat img = cv::imread("img.jpg");

  // preview before 
  cv::imshow("img", img);
  
  // nasty function which modifies const object passed by const reference
  corrupt(img);
  
  // preview after
  cv::imshow("img after corrupt(img)", img);
  cv::waitKey();

  return 0;
}
```

It's not the first time such behaviour raises eyebrows: [Stack Overflow: Is cv::Mat class flawed by design?](https://stackoverflow.com/questions/13713625/is-cvmat-class-flawed-by-design), but OpenCV dev's responses usually involve straw man "cloning all the way would decrease performance, and that's not what most people want" and suggest explicitly cloning when necessary, seemingly unaware of other possible solutions (providing a Mat class which is essentially immutable, but otherwise compatible with cv::Mat interface). I would argue that abusing language type system can hardly be justified by performance, which would make no sense if it compromises program correctness.

But enough ranting, time to do something about it. Unfortunately, when I became fully aware of this situation with cv::Mat - too much code was influenced, so instead of coming up with something completely new my goal was fixing this behaviour with minimal changes.

Core concept was wrapping cv::Mat with a thin wrapper which clones the matrix **both** at construction and at access. Something like that:

```c++
class ConstMat {
  const cv::Mat mat;
  
  public:
  // remove explicit if you're fine with automatic casting from cv::Mat 
  // to ConstMat that involves clone
  explicit ConstMat(const cv::Mat &in) : mat(in.clone()) {
  }
  
  cv::Mat clone() const {
    return mat.clone();
  }
};
```

You might be wondering how is it better than using clone on cv::Mat directly like suggested. In fact - it's not dramatically different, but now you cannot forget to call clone() before modification, and as a result - you can be absolutely sure that ConstMat instance will never be modified once constructed (unless someone is really motivated to do so of course). Additional benefit - when you copy ConstMat to ConstMat - default copy constructor will be used, which uses default refcounting OpenCV mechanism without clones. We can afford that, since both instances are immutable. But we ain't done yet.

Many methods or functions that operate on cv::Mat actually don't make any changes to underlying data. This means that we can call them directly on internal data, without worrying about possible mutation and without making unneeded clones. You can implement only a subset of methods/functions that you need. 

```c++
class ConstMat {
  ...
  public:
  ...
  // wrap methods that we're "allowed" to call
  cv::Size size() const {
    return mat.size();
  }
  
  bool empty() const {
    return mat.empty();
  }
  
  int type() const {
    return mat.type();
  }
  ...
  // same for functions that don't modify cv::Mat
  // we could've used `friend` and retain the original call syntax
  // but I find this approach (return value) slightly more convenient 
  cv::Mat cvtColor(int code) const {
    cv::Mat out;
    cv::cvtColor(mat, out, code);
    return out;
  }
  
  cv::Mat resize(cv::Size dsize, double fx = 0, double fy = 0) const {
    cv::Mat out;
    cv::resize(mat, out, dsize, fx, fy);
    return out;
  }
  ...
};
```

Above I've made a point about temporary data, cloning which will be an unneeded overhead. Can we somehow make use of that? Move semantics to the rescue!

```c++
class ConstMat {
  ...
  public:
  ...
  // don't clone temporaries, move instead
  ConstMat(cv::Mat &&in) : mat(std::move(in)) {
  }
  ...
};

```

Now we don't need to call clone() everywhere we want to make an assignment. Usage example:
```c++
cv::Mat img1 = cv::imread("file.jpg");
const auto cimg1 = ConstMat(std::move(img1));

// or more concise
const auto cimg2 = ConstMat(cv::imread("file.jpg"));
```

There's a problem with this approach though, can you spot it?

It's about the refcounting. If you have multiple matrices which point to the same data, moving from one would leave moved matrix point to the same data as the first one! 
Example to demonstrate:

```c++
cv::Mat img1 = cv::imread("file.jpg");
cv::Mat img2 = img1;  // several matrices share same data now

// try to steal data from img to contruct cimg
const auto cimg = ConstMat(std::move(img1));

// img2 "sneaked" into cimg's mat, and can now mess with it
img2 = cv::Scalar(0, 0, 0);

cv::imshow("img2", img2);
cv::imshow("cimg", cimg.clone());

// both images are corrupted now

cv::waitKey();
```

That's really not what we want. Luckily OpenCV exposes refcount in cv::Mat, so we can relatively easily fix this:

```c++
class ConstMat {
  ...
  cv::Mat checkRefcountMoveOrClone(cv::Mat &&in) {
#if (CV_MAJOR_VERSION > 2) || ((CV_MAJOR_VERSION == 2) && (CV_MINOR_VERSION > 4))
    // this works starting from 2.5
    const int refCount = in.u ? (in.u->refcount) : 0;
#else
    // this worked until 2.4.9
    const int refCount = in.refcount ? *in.refcount : 0;
#endif

    // move matrices with unique data and clone ones with shared
    if (refCount < 2) {
      return std::move(in);
    } else {
      return in.clone();
    }
  }

 public:
  ...
  // I prefer to let automatic casting happen here, but feel free
  // to add explicit if you want more control
  ConstMat(cv::Mat &&in) : mat(checkRefcountMoveOrClone(std::move(in))) { 
  }
};
```

Try running previous snippet again - now only `img2` will be corrupted, and `cimg` will be let intact. Nice!
Now we have a thin wrapper around cv::Mat, that prevents modification of its content, and at the same time doesn't force you to clone it every time. You can achieve same result using cv::Mat "responsibly", but effort to do so grows faster than the size of your codebase, so I find using this wrapper easier in the long term.

