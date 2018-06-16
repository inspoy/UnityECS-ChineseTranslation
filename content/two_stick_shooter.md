# 双摇杆打飞机的ECS教程

在下面这篇文章中，我们打算尽可能的使用Unity ECS和Job System来开发这个教程案例。
我们选用的游戏是一个简单的双摇杆打飞机，这用传统方式来实现起来是非常简单的。

## 场景设置

在这篇教程中，我们会尽可能使用ECS来制作。
我们这里需要的场景几乎就是空的，只包含了少数几个 __GameObjects__ ，如下：

* 摄像机
* 光源
* 用于保存我们生成ECS Entity需要的参数的模板对象
* 几个用于开始游戏和显示血量的UI对象

教程工程位于 *TwoStickShooter/Pure*。

## 启动入口

使用ECS时应该如何启动你的游戏？首先，你需要在任何update发生前先往System里插入一些初始的Entity。

一个简单的答案是，在项目开始运行时执行一些代码就可以了。在这个项目中，有一个包含了两个方法的类 __TwoStickBootstrap__ 。第一个方法最先初始化，创建了我们和ECS交互所需要的核心 __EntityManager__ 。

总的来说，这个启动入口实现了：

* 创建了一个EntityManager，我们用于创建和修改Entity和它们的Components的一个关键的ECS抽象。
* 创建了一些原型（Archetype），你可以把它想成一种描述了这种Entity应当包含哪些Components的蓝图。这一步是可选的，只不过Archetype可以避免在生成Entity时的内存重分配和对象移动，因为它们将立即使用正确的内存布局来创建。
* 从上述我们在场景中读取我们设置好的相关参数。

### 场景数据

纯ECS数据现在还不太支持在编辑器中编辑，所以在过渡期间我们采用了两种方法来配置我们的游戏：

1. 对于资产引用，我们在场景中创建了一些添加有 __IComponentData__ 类型的原型（Prototype）GameObject。这是我们为了自定义主角和子弹的外观而采取的方法。一旦完成为止，我们就可以抛弃这些原型对象了。
2. 对于“各项配置”，用Unity传统的在GameObject上添加组件的方式来实现是非常方便地，因为这样你可以在运行时去在Inspector面板中微调这些数值。这个样例工程就用了一个 __TwoStickExampleSettings__ 组件来达成目的，我们把这个组件添加到一个叫做 __Settings__ 的空GameObject上。这样我们就能获取到它并全局保留，而且当Settings的值发生变化时也能及时收到更新。

### 原型（Archetype）

鉴于我们这个游戏非常小，我们就把所有需要的的实体原型都声明在启动入口里了。要创建一个原型，只需要把所有Entity需要的 __ComponentType__ 都列出来就行了。

我们先看下其中一个原型，__PlayerArchetype__，用于创建，呃，玩家。

```c#
PlayerArchetype = entityManager.CreateArchetype(
    typeof(Position2D), typeof(Heading2D), typeof(PlayerInput),
    typeof(Faction), typeof(Health), typeof(TransformMatrix));
```

玩家原型`PlayerArchType`包含如下Component：

* __Position2D__ 和 __Heading2D__ - 这些内置的ECS组件允许玩家可以被正确地设置位置，以及使用内建的2D到3D变换来为接下来的渲染做准备。
* __PlayerInput__ 会在每帧用填充玩家的[输入](https://docs.unity3d.com/ScriptReference/Input.html)信息。
* __Faction__ 描述了玩家所属的“阵营”。之后我们要攻击的敌对方也会用到这个。
* __Health__ 只是一个简单的计数器。
* 最后，我添加了另一个内置的组件 __TransformMatrix__ ，它包含了一个[4x4矩阵](https://docs.unity3d.com/ScriptReference/Matrix4x4.html)来被 __InstanceRender__ System读取并进行渲染。

你可以把ECS中玩家所控制的东西视为一个由这些Component组成的Entity。

其他的原型设置都是类似的。

下一个初始化方法会在场景被加载后执行，因为这个方法需要访问场景中的一些蓝图对象。

### 从场景中提取配置

一旦场景被加载，我们的 __InitializewithScene__ 方法就会被调用。这里我们从场景中提取了一些包括一个我们能在运行时改变的Settings对象在内的一些对象。

### 开始新游戏

要开始游戏，我们要往World中添加一个玩家Entity。下面的代码实现了这一点：

```c#
public static void NewGame()
{
    // 获取ECS的EntityManager
    var entityManager = World.Active.GetOrCreateManager<EntityManager>();
    
    // 创建一个基于PlayerArchetype的实体，它会有我们列出的所有组件类型的默认值。
    Entity player = entityManager.CreateEntity(PlayerArchetype);
    
    // 我们能修改一些Component，更像是这样：
    entityManager.SetComponentData(player, new Position2D { Value = new float2(0.0f, 0.0f) });
    entityManager.SetComponentData(player, new Heading2D  { Value = new float2(0.0f, 1.0f) });
    entityManager.SetComponentData(player, new Faction { Value = Faction.Player });
    entityManager.SetComponentData(player, new Health { Value = Settings.playerInitialHealth });
    
    // 最后我们添加一个SharedComponentData用于指定外观渲染。
    entityManager.AddSharedComponentData(player, PlayerLook);
}
```

## Systems!

为了渲染一帧，我们需要做一些数据的变换：

* 我们要对玩家的输入进行采样（__PlayerInputSystem__）。
* 我们要移动玩家并且允许其进行射击（__PlayerMoveSystem__）。
* 我们要在适当的时机生成新的敌人（__EnemySpawnSystem__）。
* 敌人也需要进行移动（__EnemyMoveSystem__）。
* 敌人也需要进行射击（__EnemyShootSystem__）。
* 我们要根据玩家和敌人的动作来生成新的子弹（__ShotSpawnSystem__）。
* 我们要清掉那些已经超时的子弹（__ShotDestroySystem__）。
* 我们要让子弹产生伤害（__ShotDamageSystem__）。
* 我们要清掉把那些已经没有生命值的Entity（__RemoveDeadSystem__）。
* 我们要把某些数据推送到UI对象上（__UpdatePlayerHUD__）。

### 玩家的输入，移动和射击

值得一提的是，在编写ECS风格代码中应该优先以多人游戏考虑：我们总是拥有一批玩家。

PlayerInputSystem就是一个负责从Unity输入API获取输入信息，并把这些输入插入到叫做PlayerInput的组件中。同时更新用于计算玩家射击CD的 **fire cooldown**。

PlayerMoveSystem处理从PlayerInputSystem中获得的基础移动和射击信息。这是相对很直接的，除了当玩家射击时创建子弹的方法。在创建子弹时并不是直接生成一个子弹实体，而是创建了一个叫做 __ShotSpawnData__ 的组件，并将该组件交由被另一个System来执行相关的操作。这叫做关注点分离，它解决了几个问题：

1. PlayerMoveSystem不需要知道生成一个子弹需要哪些组件。
2. ShotSpawnSystem（同时为玩家和敌人生成子弹）不需要知道这些子弹为什么，是怎么发射出来的。
3. 我们可以稍后在被明确定义的时间点上，在World中一并生成这些子弹。

这种操作实现了类似传统组件架构中的延迟事件。

### 敌人的生成，移动和射击

如果敌人不会向你还击，那这个游戏就太没挑战性了，所以很自然的，就会有一些System来做这些事情。

对于敌人相关的System，最有趣的一个就是EnemySpawnSystem了。它需要持续追踪生成敌人的时刻，但我们不想把这些状态放到System里。ECS的设计原则之一是应该记录组件的状态并稍后可以重建场景。存储一些状态变量用于帧与帧之间是违反这一原则的。

作为替代，EnemySpawnSystem把它的状态存储在了一个单例Entity上的组件中。

The EnemySpawnSystem instead stores its state in a singleton component, attached to a singleton Entity. We create the Entity and the initial values for this component in a setup function __EnemySpawner.SetupComponentData()__. Here we also initialize a random seed and store that along with the rest of the data, so that games will predictably spawn enemies in the same pattern every time, regardless of frame rate or if something fancy like state replay is happening.

Inside the EnemySpawnSystem, due to **ref return** not being implemented yet, we have to take a copy of our system's state from the singular __State__ array, and then when we've  modified it for the next frame, we store it back into the component.

This may look like a lot of boilerplate (and it is) but it's also kind interesting to think about this in a different way. What if we renamed State to "Wave" and updated more than one of them at a time, orchestrated by some other system? We would get multiple simultaneous "Waves" spawning and updating in concert. ECS makes these sort of transformations much easier and cleaner than if we had used global data attached to the system. 

One quirk is that we have to put off actually spawning an Entity until we've completed the above step of storing back our component state, because touching the EntityManager will
immediately invalidate all injected arrays (including the one where our state is kept!). Our
solution to this is [command buffers](https://docs.unity3d.com/ScriptReference/Rendering.CommandBuffer.html) 
(via __EntityCommandBuffer__ - but command buffers don't yet support __ISharedComponentData__, which is needed here to set the rendered look.)

Enemies move automatically using the stock __MoveForward__ component, so that's taken care of.

We need them to shoot however, and EnemyShootSystem does just that. It creates entities with ShotSpawnData data on them that will be converted to shots later; together with any player shots.

Finally we also need a way to get rid of enemies that go offscreen. __EnemyRemovalSystem__
goes through all enemy positions and kills offscreen enemies by setting their health to -1.

### Handling shots

ShotSpawnSystem deals with creating actual shots from the requests dropped into the ECS by players and enemies. This is a simple straightforward affair that just loops over all ShotSpawnData and converts them into shots.

More interesting is ShotDamageSystem, which intersects bullets and targets and deals damage. This uses 4 injected groups:

* Players
* Shots fired by players
* Enemies
* Shots fired by enemies

This way it can kick off two jobs:

* Players vs enemy shots
* Enemies vs player shots

It uses a very simplistic point against circle collision test.

We also need to get rid of shots that didn't hit anything and just fly off. When their time to
live goes to zero, we let ShotDestroySystem remove them.

### Final pieces

We need something that culls dead objects from the world, and RemoveDeadSystem does just
that.

Finally, we want to display some data about the player's health on the screen and UpdatePlayerHUD accomplishes this task.
