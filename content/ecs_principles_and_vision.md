# 实体-组件-系统（ECS）原理

ECS建立于一系列原则之上。这些原则为我们要实现的目标提供了良好的基础。一些原则非常清晰地反映在代码里，而另一些只是我们为自己设定的目标。

## 默认情况下的性能

我们希望能有一个简单的方法来在各个平台上生成高效的机器码。

我们的性能甚至可以和用C++手写的，高度优化过的[SIMD](https://en.wikipedia.org/wiki/SIMD)函数相提并论。

我们使用一系列方式包括编译技术（Burst），容器（Unity.Collections），数据布局（ECS）来使编写的代码一开始就是高效的。

* 数据布局和迭代 - ECS在默认情况下在批量迭代实体时保证数据布局在内存中是线性的。而这就是ECS带来的性能提升的关键因素。
* C# job system 使编写安全的多线程代码变得很简单。C# Job调试器会检测所有的线程竞争条件。
* Burst是我们专为C# jobs准备的编译器。我们让C# job代码遵循一些模式，就能生成更高效的机器码。代码在各个平台上的编译和优化都使用了SIMD指令集。

拿实例化的性能为例。根据理论上的限制，实例化10万个实体（每个320字节的内存分配）需要9毫秒，而用ECS来实现也只需要10毫秒。所以我们的性能已经非常接近理论上限了。

我们在Unite奥斯汀上展示了一个包含10万个独立实体还能跑到60fps的战争模拟的Demo。所有的业务代码都跑在多个CPU核心上。
[See ECS performance demo [Video]](https://www.youtube.com/watch?v=0969LalB7vw)

## 简单

编写高性能([performant](https://en.wiktionary.org/wiki/performant))必须十分容易。我们坚信，编写高性能的代码就像写__MonoBehaviour.Update__一样容易。

> Note: 为了设定正确的预期，我们认为仍然有一些方法可以实现这一目标。

## 用同一种方式写代码

我们想找到一种可以同时编写游戏代码，编辑器代码，资产管线代码和引擎代码的方式。我们相信这样可以给用户提供一个更简单，更强大的工具。

物理系统就是一个绝佳的例子。目前物理相关的代码对开发者是不可见的，很多开发者都想基于他们游戏的需求来微调一下这些物理代码。如果物理引擎的代码
 If physics engine code was written the same way as game code using ECS, it would make it easy to plug your own simulation code between existing physics simulation stages or take full control.

Another example, lets imagine you want to make a heavily moddable game.

If our import pipeline is implemented as a set of __ComponentSystems__. And we have some FBX import pipeline code that is by default used in the asset pipeline to import and postprocess an FBX file. (Mesh is baked out and FBX import code used in the editor.)

Then it would be easy to configure the Package Manager that the same FBX import and postprocessing code could be used in a deployed game for the purposes of modding.

We believe this will, at the foundation level, make Unity significantly more flexible than it is today.

## Networking

We want to define one simple way of writing all game code. When following this approach, your game can use one of three network architectures depending on what type of game you create.

We are focused on providing best of class network engine support for hosted games. Using the recently acquired [Multiplay.com](http://Multiplay.com) service we offer a simple pipeline to host said games.

* FPS - Simulation on the server
* RTS - Deterministic lock step simulation
* Arcade games - GGPO

> Note: To set expectations right, we are not yet shipping any networking code on top of Entity Component System. It is work in progress.

## Determinism

Our build pipeline must be [deterministic](https://en.wikipedia.org/wiki/Deterministic_algorithm). Users can choose if all simulation code should run deterministically.

You should always get the same results with the same inputs, no matter what device is being used. This is important for networking, replay features and even advanced debugging tools.

To do this we will leverage our Burst compiler to produce exact floating point math between different platforms. Imagine a linux server & iOS device running the same floating point math code. This is useful for many scenarios particularly for connected games, but also debugging, replay etc. 

> Note: Floating point math discrepancies is a problem that Unity decided to tackle head on. This issue has been known about for some time, but so far there has not been a need great enough to encourage people to solve it. For some insight into this problem, including some of the workarounds needed to avoid solving it, consider reading [Floating-Point Determinism by Bruce Dawson](https://randomascii.wordpress.com/2013/07/16/floating-point-determinism/).

## Sandbox

Unity is a sandbox, safe and simple.

We provide great error messages when API's are used incorrectly, we never put ourselves in a position where incorrect usage results in a crash and that is by design (as opposed to a bug we can quickly fix).

A good example of sandbox behaviour is that our C# job system guarantees that none of your C# job code has race conditions. We deterministically check all possible race conditions through a combination of static code analysis & runtime checks. We give you well written error messages about any race conditions right away. So you can trust that your code works and feel safe that even developers who write multithreaded game code for the first time will do it right.

## Tiny

We want Unity to be usable for all content from < 50kb executables + content, to gigabyte sized games. We want Unity to load in less than 1 second for small content.

## Iteration time

We aim to keep iteration time for any common operations in a large project folder below 500ms.

As an example we are working on rewriting the C# compiler to be fully incremental with the goal of:

> When changing a single .cs file in a large project. The combined compile and hot reload time should be less than 500ms.

## Our code comes with full unit test coverage

We believe in shipping robust code from the start. We use unit tests to prove that our code works correctly when it is written and committed by the developer. Tests are shipped as part of the packages.

## Evolution

We are aware that we are proposing a rather large change in how to write code. From MonoBehaviour.Update to ComponentSystem & using jobs.

We believe that ultimately the only thing that convinces a game developer is trying it and seeing the result with your own eyes, on your own game. 

Thus it is important that applying the ECS approach on an existing project should be easy and quick to do. Our goal is that within 30 minutes a user can, in a large project, change some code from MonoBehaviour.Update to ComponentSystem and have a successful experience optimizing his game code.

## Packages

We want the majority of our engine code to be written in C# and deployed in a Package. All source code is available to all Unity Pro customers.

We want a rapid feedback loop with customers, given that we can push code and get feedback on something quickly in a package without destabilizing other parts.

Previously most of our engine code was written in C++, which creates a disconnect with how our customers write code and how programmers at Unity write code. Due to the Burst compiler tech & ECS, we can achieve better than C++ with C# code and as a result we can all write code exactly the same way.

## Collaboration

We believe Unity users and Unity developers are all on the same team. Our purpose is to help all Unity users create the best game experiences faster, in higher quality, and with great performance. 

We believe every feature we develop must be developed with real scenarios and real production feedback early on. The Package Manager facilitates that.

For those in the community that want to contribute engine code, we aim to make that easy by working directly on the same code repositories that contributors can commit to as well. Through well defined principles and full test coverage of all features, we hope to keep the quality of contributions high as well. 

The source code repositories will be available for all Unity Pro Customers.

## Transparency

We believe in transparency. We develop our features in the open, we actively communicate on both forum and blogs. We reserve time so each developer can spend time with customers and understand our users pain points.
