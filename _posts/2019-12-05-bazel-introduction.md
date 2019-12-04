---
layout: post
title: "Dive into Bazel. Part I: Introduction."
comments: true
---

In this post I want to talk about how we build and test our software. It looks like a pretty obvious question and that's why sometimes it slips from our heads - just take your favourite build tool (gradle, maven, sbt, cmake) and start tinkering around with your project. But the question is - can we do better? What's wrong with existing tools? In this post I'll try to give an overview of bazel build tool, explain main ideas behind it and show how it can increase productivity of your team.

## What is Bazel? 

Bazel is a modern build and test tool, similar to the tools that you're used to. On the first glance it does exactly the same job, but the ideas behind the processes are pretty much different. 

Bazel (originally `blaze`) has been developed in Google and was open sourced several years ago. It was aimed to solve some problems that Google had with [their monorepository](https://cacm.acm.org/magazines/2016/7/204032-why-google-stores-billions-of-lines-of-code-in-a-single-repository/fulltext), here are some of them:

- maintaining large code base at scale,
- slow compilation time,
- slow execution time of unit tests,
- support for multiple languages,
- dependency management across services,
- universal tools for building and testing projects.

Even though I'm a huge fan of monorepos, we're not going to discuss here all advantages and trade-offs of this code structuring type. Instead, let's dive into more details about `bazel` itself :)

## What's wrong with other build tools?

First of all let's discuss why somebody might want to have a new build tool.

#### Slow compilation time 
As the company grows its codebase will grow as well. And obviously, the more code you have to build and maintain - the more time it will take. Given that most compilers (C++, Rust, Scala with sbt) are pretty slow, compilation time becomes very important factor for developers productivity and development velocity. One of the solutions is to use incremental compilation, but from my experience it's not reliable enough and could easily  produce some cryptic run-time bugs (like `java.lang.ClassNotFoundException`) which are gone once you rebuild everything from scratch. 

#### Non-reproducible builds
The most important thing is that almost all build tools depends on your system environment and configuration. If you build your project with JDK 13 and your remote colleague has JDK 8, most likely project will not compile. Same thing could happen with GCC version or any other global dependency that you may have installed on your host machine or CI server. Therefore, your builds are not reproducible and not deterministic. From the same code base you can get different results and service that works on your machine can suddenly start throwing exceptions on production. And ultimately it becomes harder to guarantee software quality, because it could lead to floating or tricky bugs that are hard to reproduce on local environment.

#### Different build tools and their complexity
Almost every programming language brings its own build and test tools and practices which is not bad by itself. But some of these tools might be complicated and even confusing. As an example we might think about standard build tool in Scala community called `sbt`. If you're familiar with this tool, then you probably know how hard is it to start using it for beginners and how difficult is it to maintain and evolve configuration. I'll not dive here into all details, but if you're interested in them - [there is good article](http://www.lihaoyi.com/post/SowhatswrongwithSBT.html) with deep analysis.

On the top of that, there is also a pretty common case in companies to have different tech stacks across teams, so every time when developer jumps to another project or wants to help other team, most probably he/she must learn a new toolchain that increases learning curve and entry threshold.

## Bazel to the rescue
Let's take a look on main concepts and ideas in `bazel`. When you install `bazel` for the first time, it doesn't know anything about how to build your project. To be able to compile and package your project you have to plug-in special `rules`. 

Rule in `bazel` is a special definition that describes the relationship between inputs and outputs, and the steps to build those outputs. If you familiar with functional programming concepts you can treat every rule as a pure function that takes inputs (e.g. code sources) and produce some outputs (e.g. binaries, but actually it could be whatever you want). 

Bazel community does an amazing work and [provides a lot of ready out of the box rules](https://github.com/bazelbuild) for building and testing apps for specific programming languages and even frameworks, so you can easily build Java, Scala, Rust, Haskell, Go, JavaScript, Protobuf & gRPC, etc. 

The "purity" of rules leads us to another beautiful concept and this is there `bazel` really starts to shine. And it's called hermetic builds.

#### Hermetic builds
In simple words hermetic build means that build process doesn’t depend on host environment, it doesn't have an access to env variables, installed software like compilers, SDKs, libraries, no assumptions are made that something already exists and properly configured on hosting machine (e.g. protobuf compiler or docker daemon). We have complete sandbox with deterministic actions that guaranties reproducibility. There is no more need for complicated host configurations and pain for newcomers - everything that your project needs for build is defined inside your build files. Sandboxing also helps to prevent build system bugs itself, which is common especially for incremental builds and complex build systems. 

But how does it work? `Bazel` knows all dependencies for every task in action graph and doesn’t have any implicit assumptions. It computes hashes for every input and output and can guaranty that every time we run build command the output will be exactly the same, matching bit to bit. 

This fact gives us an extreme power: first of all, we can cache all outputs and if inputs don't change - we can just skip building it again and put it into local or remote cache. As the codebase becomes very large, rebuilding it from scratch can take significant amount of time or even become impossible on local machines. However, with remote cache most part is already there - we only need to recompile targets that have changed. The same thing happens with unit tests - if covered code has not been changed, there is no need to test it again. Therefore we can benefit a lot from extremely fast local and CI builds (with our Scala monorepo we got more than 15x speed up!).  Moreover, if your company approaches Google-scale size and code base, you can also benefit from distributed builds.

Another interesting thing is that we can deliver exactly the same application version for development, QA and deployment. If it sounds like docker use-case for you - you're right, but `bazel` gives you an ability to achieve it without additional tools. Moreover, docker builds are not reproducible as well (more details about that in following parts).

If you use micro service architecture for your backend, combined with mono-repository it’s very easy to have a snapshot of your system for every development step. 

#### Dependency management
The good thing about `bazel` is that it supports concept of hermetic build so thoroughly that it’s almost impossible for us to create non-hermetic build. But of course, this requires some discipline that we have to follow in order to achieve it. 

First of all, we need to specify our third party dependencies such as Java artifacts in a bit specific way: not only do we need to specify location and versions of artifacts, but we also need to provide hash for them to make sure that nobody has republished different code with the same version. This is especially critical for dynamic languages such as JavaScript with `npm` or Python.

The second important thing - we need to declare all transitive dependencies explicitly in our build files for deterministic output. It requires more effort from developers but it solves a lot of problems so in my opinion it’s totally worth it. Of course nobody wants to do it manually, so Bazel developers provide us special tools to generate list of transitive dependencies for different languages ([bazel-deps for Scala as an example](https://github.com/johnynek/bazel-deps)).

To make our life a bit easier, `bazel` also contains a couple of auxiliary commands, so we can write special queries and track down dependencies for our targets. It uses special syntax for graph description so you can even visualise it using tool like `Graphviz`. `Graphviz` is extremely useful tool for tracking dependencies among services and protocol changes (in case if you use something like `gRPC`). 

## Pitfalls
Like any other technology of course `bazel` is not perfect. So here is a list of issues I’ve faced while I was working on my current project. 

First of all, `bazel` is relatively young. Therefore, sometimes you can face with lack of documentation or unspecified properties. Be ready to read source code of rules and ask community for help.

Second, you should get used to dependencies declaration and always keep in mind the main idea of `bazel`: ensure reproducible builds. Sometimes this can arise in unexpected places such as providing `bazel` with environment variables for staging or production deployments.

Third, major part of `bazel` rules are supported by community, so be prepared to contribute or spend extra time trying to achieve some unusual stuff. For example, most of the rules support compilation and testing, but there can be a problem to get test coverage reports. 

All these factors can prevent one from going a full `bazel` build setup, because languages are highly ingrained in their environments (Rust with Cargo, npm for JS) and it can be challenging to match dev experience with a universal build system.

## Summary
Overall, `bazel` is a great tool and its benefits for me definitely outweigh downsides. 

On my current project we use it for our mono-repository for almost one year and we were able to solve several major pain-points for us (like reproducible environments deployments, fast release cycles and so on).

So I would definitely recommend to play around with this tool and give it a try!

In the next part we'll take a closer look on a practical side with `bazel`: initial workspace configuration, build files, compilation and testing. 

Stay tuned!

## Useful links
- [Official bazel website](https://bazel.build/)
- [Bazel vision](https://docs.bazel.build/versions/master/bazel-vision.html)
- [Bazel ecosystem (GitHub)](https://github.com/bazelbuild)