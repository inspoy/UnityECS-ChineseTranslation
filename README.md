![Unity](https://unity3d.com/files/images/ogimg.jpg?1)
> 本文由[Inspoy](http://inspoy.cc)出于个人兴趣进行翻译，水平有限难免出现翻译错误，语句不通顺，语序错乱崩坏。。而且也许不能及时更新到最新的官方版本，最重要的是，随时可能弃坑←_←

* 对应官方版本：Release 6

# 实体-组件-系统（ECS）用户手册

* [ECS原理](content/ecs_principles_and_vision.md)
* [ECS适合你吗?](content/is_ecs_for_you.md)
* [ECS相关概念](content/ecs_concepts.md)
* [ECS是如何工作的](content/getting_started.md)
* [ECS特性详解](content/ecs_in_detail.md)
* [更多信息](#further-information)
* [ECS当前状态](#status-of-ecs)

## Job system概览

* [Job system如何工作](content/job_system.md)
* [底层概览 - 创建容器&自定义Job类型](content/custom_job_types.md)
* [如何针对Burst编译器进行优化](content/burst_optimization.md)
* [从一个Job安排另一个Job - why not?](content/scheduling_a_job_from_a_job.md)

## 教程

* [教程: 一个用ECS实现的双摇杆打飞机](content/two_stick_shooter.md)

## 简单的例子

* [RotationExample.unity](content/rotation_example.md): 改变位于一个运动的球体里的Entity的Component的循环。

## 社区

* [Performance by Default](http://unity3d.com/performance-by-default)

## 更多信息

![Unity at GDC 2018](https://blogs.unity3d.com/wp-content/uploads/2018/03/Unity-GDC-Google-Desktop-Profile-Cover.jpg)

### Unity at GDC 2018

* [Keynote: The future of Unity (Entity Component System & Performance) by Joachim Ante](https://www.youtube.com/watch?v=3Mq9EH8RT_U)

* [Evolving Unity by Joachim Ante [FULL VIDEO]](https://www.youtube.com/watch?v=aFFLEiDr3T0)

* [Unity Job System and Entity Component System by Tim Johansson [FULL VIDEO]](https://www.youtube.com/watch?v=kwnb9Clh2Is)

* [Democratizing Data-Oriented Design: A Data-Oriented Approach to Using Component Systems by Mike Acton [FULL VIDEO]](https://www.youtube.com/watch?v=p65Yt20pw0g)

* [C# Sharp to Machine Code by Andreas Fredriksson [FULL VIDEO]](https://www.youtube.com/watch?v=NF6kcNS6U80)

* [ECS for Small Things by Vladimir Vukicevic [FULL VIDEO]](https://www.youtube.com/watch?v=EWVU6cFdmr0)

* [Exclusive: Unity takes a principled step into triple-A performance at GDC](https://www.mcvuk.com/development/exclusive-unity-takes-a-principled-step-into-triple-a-performance-at-gdc)

  

![Unity at Unite Austin 2017](https://blogs.unity3d.com/wp-content/uploads/2017/09/Unite_Austin_Blog_Post.jpg)

### Unity at Unite Austin 2017

* [Keynote: Performance demo ft. Nordeus by Joachim Ante](http://www.youtube.com/watch?v=0969LalB7vw)
* [Unity GitHub repository of Nordeus demo](https://github.com/Unity-Technologies/UniteAustinTechnicalPresentation)
* [Writing high performance C# scripts by Joachim Ante [FULL VIDEO]](http://www.youtube.com/watch?v=tGmnZdY5Y-E)



---

## ECS当前状态

Entity 迭代：
* 我们已经实现了多种方式（foreach vs 数组，注入 vs API）。现在我们暴露了所有可能的方式，以便用户可以先尽量尝试然后向我们反馈你们更喜欢哪一种。之后我们会删除其他的方式，只保留一个最好的。

使用ECS的Job API：
* 我们相信可以让它变得更简洁。接下来要尝试的是Async / Await，看看是否有一些又快又简单的模式。

我们的目标是，让Entity可以像GameObject那样可以在编辑器里编辑。场景中要么都是Entity，要么都是GameObject。然而我们现在并没有工具可以不通过GameObject来编辑Entity。所以之后我们想实现：
* 在Hierarchy和Inspector窗口中显示和编辑Entity。
* 让Entity支持保存、打开场景，保存为Prefab。
