# ECS相关概念

如果你熟悉[实体-组件-系统（ECS）](https://en.wikipedia.org/wiki/Entity%E2%80%93component%E2%80%93system)的相关概念，你可能会发现这和Unity现存的 __GameObject__/__Component__ 有命名上的冲突。下面是一系列Unity中ECS相关概念的映射关系：

### Entity → Entity

首先Unity并没有实际存在的 __Entity__ ，所以这个结构只是以其概念命名。Entity像是超轻量级的GameObject，因为它们自己什么行为都不做，也不存储任何数据（甚至是名字）。

你可以给Entity添加Component，就像给GameObject添加Component一样

### Component → ComponentData

这里介绍一个高性能的组件类型

```
struct MyComponent: IComponentData
{} 
```

__EntityManager__ 管理内存并且尽可能保证在遍历一组Component的时候能线性地访问内存。而且对于每个Entity除了struct自身没有额外开销。

为了同现有的组件类型（比如 __MonoBehaviours__ ）区分，其名字也说明了它只存储数据。 __ComponentData__ 可以在Entity上添加和删除。

### System → ComponentSystem

Unity中有许多 "System"，同"Component"一样，这个名字是一个涵盖性术语。 __ComponentSystem__ 定义了你游戏的行为，而且可以操作某些类型的数据：传统的GameObject和Component，或者纯ECS ComponentData和Entity数据。
