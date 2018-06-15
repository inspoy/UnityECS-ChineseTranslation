# ECS入门

## 我们要解决什么问题？

当使用 __GameObject__/__MonoBehaviour__ 开发游戏时，很容易编写代码，然而最终却难以阅读，维护和优化。这是许多因素造成的：[面向对象编程](https://en.wikipedia.org/wiki/Object-oriented_programming)，Mono编译的非最佳的机器码，[垃圾回收（GC）](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science))和[单线程](https://en.wikipedia.org/wiki/Thread_(computing)#Single_threading)代码。

## 让ECS来拯救

[实体-组件-系统（ECS）](https://en.wikipedia.org/wiki/Entity%E2%80%93component%E2%80%93system)是一种编写代码的新方式，专注于你要真正解决的问题：构成你游戏的数据和行为。

In addition to being a better way of approaching game programming for design reasons, using ECS puts you in an ideal position to leverage Unity's job system and Burst compiler, letting you take full advantage of today's multicore processors.

We have exposed Unity's native job system so that users can gain the benefits of [multithreaded](https://en.wikipedia.org/wiki/Thread_(computing)) batch processing from within their ECS C# scripts. The native job system has built in safety features for detecting [race conditions](https://en.wikipedia.org/wiki/Race_condition).

However we need to introduce a new way of thinking and coding to take full advantage of the job system.

## What is ECS?

### MonoBehavior - 亲切的老朋友

MonoBehaviour 同时包含数据和行为，以下脚本会在每帧更新 __Transform__ 组件的旋转信息：

```C#
using UnityEngine;

class Rotator : MonoBehaviour
{
    // 数据 - 可在Inspector中进行编辑
    public float speed;
    
    // 行为 - 读取speed的值并改变Transform的旋转信息
    void Update()
    {
        transform.rotation *= Quaternion.AngleAxis(Time.deltaTime * speed, Vector3.up);
    }
}
```

然而MonoBehaviour继承了好多其他的类；每个类都包含了他们自己的一系列数据，即使上面的脚本中完全没有用到，于是我们就无故浪费了好多内存。所以我们得想想哪些是我们真正需要的数据，然后优化一下代码。

### ComponentSystem - 踏进新时代

在新的模型中，Component只包含数据。

而 __ComponentSystem__ 则包含行为，一个ComponentSystem负责更新所有匹配一系列Component的GameObject。

```C#
using Unity.Entities;
using UnityEngine;

class Rotator : MonoBehaviour
{
    // 数据 - 可在Inspector中进行编辑
    public float Speed;
}

class RotatorSystem : ComponentSystem
{
    struct Group
    {
        // 定义了这个ComponentSystem处理的GameObject需要拥有的Component。
        public Transform Transform;
        public Rotator   Rotator;
    }
    
    override protected void OnUpdate()
    {
        // 我们马上就看到了第一个优化
        // 我们知道所有rotator用的deltaTime都是一样的
        // 于是我们把deltaTime缓存为一个本地变量来获得更好的性能
        float deltaTime = Time.deltaTime;
        
        // ComponentSystem.GetEntities<Group> 
        // 这个方法让我们可以高效地遍历每个同时有Transform和Rotator的GameObject
        // (我们在上边的结构体里定义了)
        foreach (var e in GetEntities<Group>())
        {
            e.Transform.rotation *= Quaternion.AngleAxis(e.Rotator.Speed * deltaTime, Vector3.up);
        }
    }
}
```

## 混合ECS: 将ComponentSystem和现存的GameObject & Component一起使用

现在还有许多基于MonoBehaviour/GameObject写的代码，我们想

There is a lot of existing code based on MonoBehaviour, GameObject and friends. We want to make it easy to work with existing GameObjects and existing components. But make it easy to transition one piece at a time to the ComponentSystem style approach.

In the example above you can see that we simply iterate over all components that contain both Rotator and Transform components.

### How does the Component System know about Rotator and Transform?

In order to iterate over components like in the Rotator example, those entities have to be known to the __EntityManager__.

ECS ships with the __GameObjectEntity__ component. On __OnEnable__, the GameObjectEntity component creates an entity with all components on the GameObject. As a result the full GameObject and all its components are now iterable by ComponentSystems.

> Thus for the time being you must add a GameObjectEntity component on each GameObject that you want to be visible / iterable from the ComponentSystem.

### How does the ComponentSystem get created?
Unity automatically creates a default world on startup and populates it with all Component Systems in the project. Thus if you have a game object with the the necessary components and a __GameObjectEntity__, the System will automatically start executing with those components.

### 这对我的游戏意味着什么？

这意味着你可以一个一个地把 __MonoBehaviour.Update__ 中的行为转为ComponentSystem中实现。实际上你可以把所有的数据留在MonoBehaviour里，实际上这也是转换为ECS风格代码的一个非常简单的开始。

So your scene data remains in GameObjects & components. You continue to use __[GameObject.Instantiate](https://docs.unity3d.com/ScriptReference/Object.Instantiate.html)__ to create instances etc.

You simply move the contents of your MonoBehaviour.Update into a __ComponentSystem.OnUpdate__ method. The data is kept in the same MonoBehaviour or other components.

#### 好处：

+ 实现数据和行为的分离 - 你得到了更简洁的代码
+ System一次性对很多对象批量操作，避免了逐对象虚拟调用，在批处理中就很容易进行优化了（参考上面的deltaTime优化）。
+ 你可以继续使用Inspector面板和其他编辑器工具

#### 不足：

- 实例化时间不会有提升
- 加载时间也不会有提升
- 数据都是随机访问的，不能保证内存被线性访问
- 没有多核优化
- 没有[SIMD](https://en.wikipedia.org/wiki/SIMD)


So using ComponentSystem, GameObject and MonoBehaviour is a great first step to writing ECS code. It gives you some quick performance improvements, but it does not tap into the full range of performance benefits available.

## 纯ECS: 一切为了性能 - IComponentData & Jobs

One motivation to use ECS is because you want your game to have optimal performance. By optimal performance we mean that if you were to hand write all of your code using SIMD intrinsics (custom data layouts for each loop) then you would end up with similar performance to what you get when writing simple ECS code.

C# Job System并不支持托管类型，只支持结构体和 __NativeContainers__ 。所以只有 __IComponentData__ 可以在C# Job中被安全地访问。

EntityManger对于ComponentData尽可能保证[线性内存布局](https://en.wikipedia.org/wiki/Flat_memory_model)。用IComponentData实现C# Jobs是实现高性能的很重要的组成部分。

下面是一个完整的例子：

```cs
using System;
using Unity.Entities;

// RotationSpeed组件只是简单的储存了速度值
[Serializable]
public struct RotationSpeed : IComponentData
{
    public float Value;
}

// 目前要把ComponentData添加到GameObject必须使用这个Wrapper
// 将来我们想把这个Wrapper自动化
public class RotationSpeedComponent : ComponentDataWrapper<RotationSpeed> { } 
```

```cs
using Unity.Collections;
using Unity.Entities;
using Unity.Jobs;
using Unity.Burst;
using Unity.Mathematics;
using Unity.Transforms;
using UnityEngine;

// 使用IJobProcessComponentData来遍历匹配所需组件类型的所有Entity
// 对Entity的处理是并行的，主线程只负责调度
public class RotationSpeedSystem : JobComponentSystem
{
    // IJobProcessComponentData是用来遍历所有拥有指定Compoenent的Enity的简单方法
    // 它也比IJobParallelFor用起来更方便
    [BurstCompile]
    struct RotationSpeedRotation : IJobProcessComponentData<Rotation, RotationSpeed>
    {
        public float dt;

        public void Execute(ref Rotation rotation, [ReadOnly]ref RotationSpeed speed)
        {
            rotation.Value = math.mul(math.normalize(rotation.Value), math.axisAngle(math.up(), speed.Value * dt));
        }
    }

    // 我们继承JobComponentSystem，这样System就能自动提供给我们需要的依赖关系了
    // IJobProcessComponentData声明了它要对RotationSpeed读取，对Rotation写入。
    // 这样声明之后，JobSystem就可以给我们Job之间的依赖了，包括之前已经安排好的要写入Rotation或RotationSpeed的Job
    // 我们同样需要返回这个依赖关系，以便我们安排的Job可以被下一个需要它的System所注册
    // 这种方法意味着：
    // * 主线程没有任何等待，只是根据依赖关系对Job进行调度（Job当所有依赖项完成后才会开始执行）
    // * 依赖关系是自动计算出来的，所以我们可以写些模块化的多线程代码
    protected override JobHandle OnUpdate(JobHandle inputDeps)
    {
        var job = new RotationSpeedRotation() { dt = Time.deltaTime };
        return job.Schedule(this, 64, inputDeps);
    } 
}
```