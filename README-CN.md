# DragonECS - C# 实体组件系统框架
DragonECS 是一个 ECS 框架，旨在最大限度地提高动态实体修改的便捷性、模块化、可扩展性和性能。它使用纯 C# 开发，没有依赖项和代码生成功能。其灵感源自 LeoEcs Lite。
## 基本概念
### Entity
实体是数据所依附的对象。它们以标识符的形式实现，标识符有两种类型：

- int - 一次性标识符，在一个 tick 内使用。不建议存储 int 类型的标识符，请使用 entlong 类型；
- entlong - 长期标识符，包含用于明确标识的完整信息；
```csharp
// 在世界上创建一个新的实体。
int entityID = _world.NewEntity();

// 删除实体。
_world.DelEntity(entityID);

// 将一个实体的组件复制到另一个实体。
_world.CopyEntity(entityID, otherEntityID);

// 实体克隆。
int newEntityID = _world.CloneEntity(entityID);
```
- 与 entlong 合作
```csharp
// 将 int 转换为 entlong。
entlong entity = _world.GetEntityLong(entityID);
// 或者
entlong entity = (_world, entityID);

// 检查实体是否仍然存活。
if (entity.IsAlive) { }

// 将 entlong 转换为 int。如果实体不存在，则会引发异常。
int entityID = entity.ID;
// 或者
var (entityID, world) = entity;
 
// 将 entlong 转换为 int。如果实体仍然存在，则返回 true 及其 int 标识符。
if (entity.TryGetID(out int entityID)) { }
```
>实体不能脱离组件而存在，最后一个组件被删除后或者 tick 结束时，空实体将被立即自动删除。

### Component
组件是附加到实体的数据。
```csharp
// IEcsComponent 组件存储在常规存储中。
struct Health : IEcsComponent
{
    public float health;
    public int armor;
}
// 具有 IEcsTagComponent 的组件存储在标签优化存储中。
struct PlayerTag : IEcsTagComponent {}
```
### System
系统是主要逻辑，实体的行为在此定义。它们以用户类的形式存在，并至少实现一个流程接口。主要流程包括：
```csharp
class SomeSystem : IEcsPreInit, IEcsInit, IEcsRun, IEcsDestroy
{
    // 在执行 EcsPipeline.Init() 时和执行 IEcsInit.Init() 之前将被调用一次。
    public void PreInit () { }
    
    // 在执行 EcsPipeline.Init() 时以及执行 IEcsPreInit.PreInit() 之后将被调用一次。
    public void Init ()  { }
    
    // 将在 EcsPipeline.Run() 执行期间调用一次。
    public void Run () { }
    
    // 将在 EcsPipeline.Destroy() 执行期间调用一次。
    public void Destroy () { }
}
```

## 框架概念
### 管道
容器和系统引擎。负责配置系统调用队列，提供系统间消息传递机制和依赖注入机制。以 EcsPipeline 类的形式实现。
#### 建造
Builder 负责构建管道。系统被添加到 Builder 中，最后构建管道。示例：
```csharp
EcsPipeline pipeline = EcsPipeline.New() // 创建管道构建器
    // 将 System1 添加到系统队列
    .Add(new System1())
    //将 System2 添加到 System1 之后的队列
    .Add(new System2())
    //将 System3 添加到 System2 之后的队列，但作为单个实例。
    .AddUnique(new System3())
    // 完成管道构建并返回其实例
    .Build(); 
pipeline.Init(); // 初始化管道。
```
```csharp
class SomeSystem : IEcsRun, IEcsPipelineMember
{
    // 获取系统所属管道的实例
    public EcsPipeline Pipeline { get ; set; }

    public void Run () { }
}
```
> 对于同时构造和初始化，有一个方法 Builder.BuildAndInit();
### 依赖注入
框架为系统实现了依赖注入。这是一个从管道初始化开始，并注入传递给Builder的数据的过程。
```csharp
class SomeDataA { /* ... */ }
class SomeDataB : SomeDataA { /* ... */ }

// ...
SomeDataB _someDataB = new SomeDataB();
EcsPipeline pipeline = EcsPipeline.New()
    // ...
    // 将 _someDataB 注入实现 IEcsInject<SomeDataB> 的系统。
    .Inject(_someDataB) 
    // 将实现 IEcsInject<SomeDataA> 的系统添加到注入树，
    // 现在这些系统也将接收_someDataB。
    .Injector.AddNode<SomeDataA>()
    // ...
    .Add(new SomeSystem())
    // ...
    .BuildAndInit();

// ...
// 对于注入，使用 IEcsInject<T> 接口及其 Inject(T obj) 方法。
class SomeSystem : IEcsInject<SomeDataA>, IEcsRun
{
    SomeDataA _someDataA
    // obj 将是 SomeDataA 类型的实例。？？？
    public void Inject(SomeDataA obj) => _someDataA = obj;

    public void Run () 
    {
        _someDataA.DoSomething();
    }
}
```
> 使用内置依赖注入是可选的。
> 有一个扩展可以简化注入语法——自动依赖注入。
### 模块
实现共同功能的系统组可以组合成模块，并且可以简单地将模块添加到管道中。
```csharp
using DCFApixels.DragonECS;

class Module1 : IEcsModule 
{
    public void Import(EcsPipeline.Builder b) 
    {
        b.Add(new System1());
        b.Add(new System2());
        b.AddModule(new Module2());
        // ...
    }
}
```
```csharp
EcsPipeline pipeline = EcsPipeline.New()
    // ...
    .AddModule(new Module1())
    // ...
    .BuildAndInit();
```
### 排序
有两种方法可以控制管道中系统的排列，无论它们的添加顺序如何：层和排序顺序。
#### 图层
该层定义了在管道中插入系统的位置。例如，如果您希望将系统插入到管道末尾，则可以将此系统添加到 EcsConsts.END_LAYER 层。
```csharp
const string SOME_LAYER = nameof(SOME_LAYER);
EcsPipeline pipeline = EcsPipeline.New()
    // ...
    // 在结束层之前插入一个新层 EcsConsts.END_LAYER ？？？
    .Layers.Insert(EcsConsts.END_LAYER, SOME_LAYER)
    // SomeSystem 系统将被插入到 SOME_LAYER 层。
    .Add(New SomeSystem(), SOME_LAYER) 
    // ...
    .BuildAndInit();
```
内置图层按以下顺序排列：
- EcsConst.PRE_BEGIN_LAYER
- EcsConst.BEGIN_LAYER
- EcsConst.BASIC_LAYER (默认情况下，系统会添加到此处)
- EcsConst.END_LAYER
- EcsConst.POST_END_LAYER

#### 排序顺序
要对同一层内的系统进行排序，请使用 int 类型的排序顺序值。默认情况下，添加的系统 sortOrder 为 0。
```csharp
EcsPipeline pipeline = EcsPipeline.New()
    // ...
    // SomeSystem 系统将被插入到 EcsConsts.BEGIN_LAYER 层
    // 并且位于 sortOrder 小于 10 的系统之后。
    .Add(New SomeSystem(), EcsConsts.BEGIN_LAYER, 10)
    // ...
    .BuildAndInit();
```
### 流程
进程是实现了通用接口（例如 IEcsRun）的系统队列。运行器用于启动进程。内置进程会自动启动。此外，还可以实现用户进程。
#### 嵌入式流程
- IEcsPreInit、IEcsInit、IEcsRun、IEcsDestroy - EcsPipeline 生命周期进程。
- IEcsInject<T> - 依赖注入系统进程。
- IOnInitInjectionComplete - 也是一个依赖注入系统进程，但表示初始化注入已完成。
#### 