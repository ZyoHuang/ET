# ET-Zyo 项目级 AI 工作指引

## 1. 文件用途与事实优先级

本文件适用于整个仓库，供后续 AI 或新成员在分析、修改、验证本项目时优先读取。目标是说明“代码应该放在哪里、为什么这样分层、如何安全地完成修改”，而不是替代 `README.md` 与 `Book/` 中的教程。

发生描述冲突时，按以下顺序判断事实：

1. 当前源码、`.csproj`、`.asmdef`、`Packages/manifest.json`、`ProjectSettings/ProjectVersion.txt`。
2. 本文件中的项目级约定。
3. `Book/` 与根 `README.md` 中的历史说明。

仓库由 ET 7.2 演进而来，部分旧文档仍写 Unity 2021/.NET 6 或旧版组件工厂 API；当前实际基线是 Unity `6000.0.43f1`、.NET `net7.0`、C# 10。不要为了匹配旧文档而回退当前工程配置。

## 2. 项目定位

这是一个 Unity 客户端与 C#/.NET 分布式游戏服务端共仓、共享逻辑的 ET 框架项目，当前整合了：

- ET 7.2 风格的 Entity/System、事件、Actor、配置和热重载体系。
- Unity 6 客户端与独立 .NET 7 服务端。
- HybridCLR 客户端热更新/IL2CPP 元数据补充流程。
- Unity.Mathematics，供双端共享逻辑使用一致的数据类型。
- UniTask 与 NPBehave 第三方源码/插件。当前 ET 主逻辑仍主要使用 `ETTask`，NPBehave 也主要以独立插件存在；不要在没有明确边界设计时把它们直接替换进 ET 核心调度。

项目的核心目标是：开发时可把所有服务 Scene 放进一个进程便于调试，发布时只修改启动配置即可拆成多个进程；客户端、机器人与服务端尽量共享不依赖 UnityEngine 的模型和逻辑。

## 3. 仓库地图与所有权

| 路径 | 职责 | 修改提示 |
| --- | --- | --- |
| `Unity/` | Unity 6 工程，是大部分共享源码的权威物理位置 | 用 Unity 打开此目录，不是仓库根目录 |
| `Unity/Assets/Scripts/Core/` | 最底层共享运行时：Entity、EventSystem、网络、计时器、对象池、日志、配置基类等 | 双端可用，不应依赖业务模块 |
| `Unity/Assets/Scripts/Loader/` | Unity 启动、平台桥接、动态装载 Model/Hotfix、HybridCLR 接入 | 仅放不热更的装载/平台代码 |
| `Unity/Assets/Scripts/Codes/Model/` | 纯数据模型、消息/配置声明、双端共享与 Client/Server 数据组件 | Entity 数据放这里，不放通常业务方法 |
| `Unity/Assets/Scripts/Codes/Hotfix/` | 可热重载业务逻辑、System、事件和消息处理器 | 不放持久实例字段；遵守热更分析器约束 |
| `Unity/Assets/Scripts/Codes/ModelView/` | 客户端 Unity 表现层数据，如 UI、GameObject、资源组件 | 可依赖 UnityEngine，仅客户端使用 |
| `Unity/Assets/Scripts/Codes/HotfixView/` | 客户端表现层逻辑与表现事件 | 可依赖 UnityEngine，仅客户端使用 |
| `Unity/Assets/Scripts/Codes/Model/Generate/` | Excel/Proto 生成的 Client、Server、ClientServer 代码 | 生成物，禁止把手工修改当作长期修复 |
| `Unity/Assets/Scripts/Empty/` | 生成 Unity 工程/程序集所需的占位脚本 | 注释已明确：不要删除或修改 |
| `Unity/Assets/Scripts/Editor/` | 构建、代码模式、服务器启动、配置/协议生成、导航导出等编辑器工具 | 仅 Editor 程序集 |
| `Unity/Assets/Config/Excel/` | 配置表源文件 | 修改配置的首选源头 |
| `Unity/Assets/Config/Proto/` | 网络协议源文件；文件名携带端类型与起始 opcode | 修改协议的首选源头 |
| `Unity/Assets/Bundles/` | 客户端配置、代码和资源包输入/产物 | 多数内容由工具生成或打包，不手改 DLL/bytes |
| `DotNet/` | 独立服务端的 App、Loader、Core、Model、Hotfix、ThirdParty 工程 | 多个项目通过 `<Compile Include>` 链接 Unity 下的共享源码 |
| `Share/Analyzer/` | ET Roslyn 分析器，许多架构约定会以编译错误强制执行 | 改规则前先判断是业务错误还是框架规则确需演进 |
| `Share/Tool/` | `Tool`：ExcelExporter 与 Proto2CS | 生成器本身及模板在这里 |
| `Share/Libs/` | KCP、Recast 等原生/共享依赖 | 谨慎改动第三方或平台代码 |
| `Config/` | 服务端运行时配置、导出 JSON/bytes、NLog、Recast 数据 | Excel 导出物优先从源表重生 |
| `Bin/` | .NET/工具构建输出与可运行 DLL | 构建产物，不作为源码修改 |
| `Book/` | ET 设计文档和历史运行说明 | 用来理解思想，版本事实以当前工程为准 |
| `Store/` | 可选扩展模块的介绍文档 | 不是当前核心运行链的一部分 |
| `Tools/` | rsync、Recast 导出器与 IDE 配置等辅助工具 | 包含第三方二进制，避免无关格式化/替换 |

不要手工维护 Unity 自动生成的 `.csproj`/`Unity.sln`。新增、移动 Unity 资产时保留对应 `.meta`；不要提交 `Library/`、`Temp/`、`Logs/`、`UserSettings/`、`obj/` 等本地生成目录。

## 4. 程序集与分层边界

### 4.1 逻辑分层

依赖方向应保持为：

```text
ThirdParty / Mathematics
          ↓
        Core
          ↓
        Loader
          ↓
        Model ─────→ ModelView (客户端表现数据)
          ↓                 ↓
        Hotfix ────→ HotfixView (客户端表现逻辑)
```

实际 `.asmdef` 会因装载需要有少量附加引用，但业务设计仍应遵守：底层不知道上层；数据层不知道具体业务流程；共享逻辑不依赖 UnityEngine；Server 不引用 `ET.Client`。

### 4.2 Model 与 Hotfix

- `Model` 保存 Entity、组件数据、消息契约、配置类型、枚举和生命周期接口。
- `Hotfix` 保存 ObjectSystem、扩展方法、消息处理器、事件处理器和可热重载流程。
- Entity 通常直接继承 `Entity`，声明数据和经过控制的属性；业务行为写进对应 `...System` 静态类或框架处理器。
- `ModelView/HotfixView` 是同样的二分，但只服务 Unity 表现层。

这种拆分使 Hotfix DLL 可以卸载重载，同时 Entity 数据仍由 Model DLL 保持。不要把运行时状态藏在 Hotfix 类的实例字段或静态字段里。

### 4.3 Client、Server、Share

每一层内部继续按运行端分目录：

- `Share/`：双端共享，不依赖仅客户端或仅服务端设施，通常使用 `namespace ET`。
- `Client/`：客户端/机器人客户端逻辑，通常使用 `namespace ET.Client`。
- `Server/`：服务端专用逻辑，使用 `namespace ET.Server`。
- `ModelView` 与 `HotfixView` 只存在客户端表现代码。

独立服务端仍编译一部分 Client 逻辑，以支持无 Unity 表现层的机器人；这不等于 Server 代码可以反向引用 `ET.Client`。分析器 `ET0022` 会阻止 Server 程序集内的不当客户端引用。

### 4.4 CodeMode 与 ENABLE_CODES

`Unity/Assets/Resources/GlobalConfig.asset` 当前 `CodeMode: 3`，即 `ClientServer`。

- `Client`：Unity 只编译/装载客户端、共享和表现代码。
- `Server`：面向 Unity Server/服务端组合，包含服务端、共享以及机器人所需 Client 代码。
- `ClientServer`：编辑器内同时装载客户端和服务端，便于 All-in-One 调试。
- 未定义 `ENABLE_CODES` 时，`ET/Build Tool` 将选定目录编译成 `Model.dll` 和 `Hotfix.dll`，Unity 再动态加载。
- 定义 `ENABLE_CODES` 时，`*.Codes.asmdef` 直接编译源码，便于 Editor 引用和调试；此模式强制要求 `ClientServer`，不需要也不能执行 `BuildModelAndHotfix`，正式打包前必须移除该宏。

## 5. 启动与运行链

### 5.1 Unity 入口

1. `Scenes/Init` 中的 `ET.Init` MonoBehaviour 注册主线程同步上下文和核心 Singleton。
2. Unity `CodeLoader` 在 `ENABLE_CODES` 模式直接发现程序集，否则加载 `Model.dll`/`Hotfix.dll`；IL2CPP 包中按需调用 HybridCLR。
3. Loader 反射执行 `ET.Entry.Start`，避免平台入口直接依赖 Model。
4. `Entry` 初始化 Mongo/Protobuf、网络服务、`Root` 和配置，然后依次发布 `EntryEvent1`、`EntryEvent2`、`EntryEvent3`。
5. `EntryEvent1` 安装共享消息、数值、AI、客户端 Scene 管理等组件；`EntryEvent2` 创建服务端设施和配置指定的服务 Scene；`EntryEvent3` 加载客户端资源并创建客户端 Scene。

### 5.2 独立 .NET 服务端入口

1. `DotNet/App/Program.cs` 先引用 `Entry.Init()` 防止发布裁剪 Model，再调用 `DotNet/Loader/Init.Start()`。
2. Loader 注册与 Unity 端同构的核心 Singleton，找到 Model，使用可回收 `AssemblyLoadContext` 装载 `Hotfix.dll`。
3. 同样反射执行 `ET.Entry.Start`。独立服务端没有 View 程序集，因此只会命中已装载的共享/服务端事件处理器。
4. `Program` 以单线程循环调用 `Game.Update/LateUpdate/FrameFinishUpdate`；ET 异步延续回到主线程同步上下文。

### 5.3 Scene 与部署拓扑

`Root -> Process Scene -> 业务 Scene -> Entity/Component` 构成主要运行时树。事件和消息常用 `SceneType` 限定作用域，新增处理器必须选择正确的 SceneType，不能用全局分发掩盖边界。

服务端根据 `Options.Process` 读取 `StartProcessConfig`，再从 `StartSceneConfig` 创建该进程拥有的 Realm、Gate、Location、Map、Robot、Router、Benchmark 等 Scene。`Localhost` 当前把所有 Scene 放在 Process 1 中；Release 或自定义表可把同样的 Scene 分散到多个进程。这是“代码组件化、拓扑配置化”的核心，不要在业务代码里写死某个服务一定处于某个进程。

## 6. 核心设计思想

### 6.1 树形 Entity/System，而非扁平 ECS

- 所有运行时状态以 Entity 为基础，Entity 既可作为父子树节点，也可作为 Component 挂载。
- `[ChildOf(typeof(Parent))]` 声明合法父子关系；`[ComponentOf(typeof(Owner))]` 声明合法组件关系。分析器会检查 `AddChild`/`AddComponent`。
- 使用父 Entity 的 `AddChild*`、`AddComponent*` 创建对象，不直接 `new EntityType()`；使用 `Dispose`/`RemoveChild`/`RemoveComponent` 结束生命周期。
- 父对象 Dispose 会递归清理 Children 和 Components，并触发 DestroySystem；异步逻辑不能假设 await 之后对象仍存活。
- 跨异步或长期保存 Entity 引用时优先使用 `EntityRef<T>`，它用 `InstanceId` 防止对象已释放或被对象池复用后继续误用。
- `Id` 是逻辑身份，`InstanceId` 表示当前运行时实例/位置；两者不能互换。

### 6.2 数据与逻辑分离

- Entity 类放数据，System 放逻辑。生命周期由 `AwakeSystem<T>`、`DestroySystem<T>`、`UpdateSystem<T>` 等驱动。
- System 常写成静态扩展类；需要访问 Entity 私有字段时给 System 标记 `[FriendOf(typeof(EntityType))]`。
- 不通过深继承复用业务能力；通过挂载/移除组件组合能力。
- 不在 Entity 中保存另一个 Entity 类型的裸字段或委托；使用 Child、Component、`EntityRef<T>`、事件或框架映射。

### 6.3 事件驱动解耦

- 状态变化由 `EventSystem` 发布事件，UI、表现、数值观察者、场景流程分别订阅，避免一个消息处理器直接操作多个模块。
- `[Event(SceneType.X)]`、`[MessageHandler(SceneType.X)]`、`[ActorMessageHandler(SceneType.X)]` 必须表达清楚运行域。
- Entry 分阶段事件也是模块安装机制。新增全局模块时，优先在对应 Entry/Scene 创建事件中安装组件，而不是给 Loader 增加业务依赖。

### 6.4 Actor 与 Actor Location

- 挂载 `MailBoxComponent` 的 Entity 才是可接收 Actor 消息的对象；Mailbox 保证相应消息调度语义并防止逻辑重入。
- 已知 `InstanceId` 时走普通 Actor，实例位置直接编码在 ID 中，成本较低。
- 只知道稳定 `Entity.Id`、且对象可能跨进程迁移时走 Actor Location；Location 服务维护 Id 到当前 InstanceId 的映射并处理重试/锁定。
- 请求/响应选择正确的 `IActorMessage/IActorRequest/IActorResponse` 或 Location 变体，处理器基类必须与消息接口匹配。

### 6.5 All-in-One 与共享逻辑

- 开发环境可把客户端和完整服务端放进同一个 Unity 进程，也可用一个 .NET 进程承载全部服务 Scene。
- 同进程 Actor 调用可以短路为本地调度；拆分部署后仍保持相同业务 API。
- 双端共享逻辑使用 Unity.Mathematics 等无 UnityEngine 依赖类型；Unity GameObject、UI、Animator、资源加载只进入 View 层。

### 6.6 异步与取消

- ET 业务异步默认使用 `ETTask`/`ETTask<T>`；禁止 `async void`。
- 调用 ETTask：异步调用链中使用 `await`，有意 fire-and-forget 时显式 `.Coroutine()`；同步方法里不能静默丢弃任务。
- 带取消令牌的异步函数必须透传同一个令牌，await 后检查取消状态，不给令牌参数默认值，也不传 `null`。以当前源码中的 `ETCancellationToken`/相关 API 为准。
- 行为机/AI 的行为是一段可中断协程；切换行为前取消上一个协程。共用函数，不要为了复用把行为节点拆成大量微小状态。

## 7. 主要业务/基础模块索引

| 模块 | 主要位置 | 作用 |
| --- | --- | --- |
| Entity/EventSystem | `Core/Module/Entity`, `Core/Module/EventSystem` | 对象树、生命周期 System、事件、Invoke、类型扫描 |
| Network/Message | `Core/Module/Network`, `Model/Share/Module/Message`, 各端 `Module/Message` | KCP/TCP 等传输基础、Session、消息分派、RPC |
| Actor | `Model/Server/Module/Actor`, `Hotfix/Server/Module/Actor` | 已知 InstanceId 的跨 Scene/进程消息 |
| ActorLocation | 对应 `Module/ActorLocation` | 稳定 Entity.Id 寻址、迁移与位置锁 |
| Scene | `Model/Share/Module/Scene`, `Hotfix/Share/Module/Scene`, Server `SceneFactory` | 客户端/服务端 Scene 生命周期与管理 |
| Unit/Move/Numeric/ObjectWait | `Model/Share/Module/*` 与对应 Hotfix | 游戏单位、移动、数值观察、异步事件等待 |
| AI | `Model/Share/Module/AI`, `Hotfix/Share/Module/AI` | ET 的优先级行为机；与独立 NPBehave 插件区分 |
| AOI | `Model/Server/Module/AOI`, `Hotfix/Server/Module/AOI` | 服务端九宫格兴趣管理与可见性事件 |
| Recast | Share Recast 模块、`Config/Recast`, `Tools/RecastNavExportor` | 双端/服务端导航数据与寻路 |
| DB/HTTP/Router | Server 对应 Module | Mongo 访问、HTTP 处理、软路由与节点管理 |
| RobotCase/Benchmark | Server Demo 与 Module | 端到端用例、机器人压测和网络基准 |
| Resource/UI/View | `ModelView/Client`, `HotfixView/Client` | AssetBundle、GameObject、UI 与表现同步 |
| Config | `Core/Module/Config`, `ConfigLoader`, 生成 Config 类 | 按端和 StartConfig 加载 protobuf bytes |

## 8. 强制编码规则与常用模式

`Share/Analyzer` 的 ET0001–ET0022 规则会把多项约定提升为编译错误。修改前先阅读同模块现有实现，至少遵守：

- Entity 类直接继承 `Entity`，避免 Entity 多层继承。
- Entity 的数据声明在 Model；方法通常放 Hotfix System。除非框架已有明确 `[EnableMethod]` 例外，不在 Entity 内新增业务方法。
- Hotfix 中普通类必须是静态类，或继承/带有框架 `BaseAttribute` 体系认可的处理器类型；Hotfix 类不得声明非 const 实例字段/属性。
- 非 const 静态字段必须标记 `[StaticField]`，并确认热重载/清理语义。优先把可变状态放 Entity/Singleton，而不是静态变量。
- 静态 System/Helper 之间保持单向依赖，分析器禁止静态类环依赖。
- 访问其他 Entity 的字段需要合法生命周期 System 或 `[FriendOf]`；优先通过公开属性/扩展方法维持边界。
- 新增 Child/Component 必须声明正确的 `[ChildOf]`/`[ComponentOf]`。
- 所有唯一 ID/opcode 必须在规定区间且不重复；不要手工改生成消息的 opcode。
- Server 目录不依赖 `ET.Client`；共享目录不依赖端专属命名空间。
- 使用项目既有四空格/制表风格和文件邻近风格，不做与任务无关的全仓格式化。

典型功能落位：

1. 在 Model 对应端目录定义 Entity/Component 数据，并标注父子或组件归属。
2. 在 Hotfix 同结构目录定义生命周期 System、扩展方法和处理器。
3. 涉及 Unity 对象时，把表现数据/逻辑分别放到 ModelView/HotfixView，并通过事件响应共享逻辑状态。
4. 在对应 Entry 事件、SceneFactory 或 Scene 创建事件中安装组件。
5. 跨模块用事件，跨 Scene/进程用普通消息或 Actor；不要持有对另一模块内部对象的隐式全局引用。

## 9. 配置、协议与生成物

### 9.1 Excel 配置

权威输入是 `Unity/Assets/Config/Excel/**/*.xlsx`。`Tool --AppType=ExcelExporter` 会：

- 生成 `Unity/Assets/Scripts/Codes/Model/Generate/{Client,Server,ClientServer}/Config`。
- 生成 `Config/Json/{c,s,cs}` 便于检查。
- 生成客户端 `Unity/Assets/Bundles/Config/*.bytes`。
- 生成服务端 `Config/Excel/{c,s,cs}/**/*.bytes`。

文件名 `@c/@s/@cs` 和表头端标记控制输出端；StartConfig 按子目录区分 Localhost、Release、Benchmark、RouterTest 等拓扑。修复配置问题时修改 xlsx 并重新导出，不直接修补 JSON、bytes 或 Generate 代码。

### 9.2 Proto 消息

权威输入是 `Unity/Assets/Config/Proto/*.proto`。文件名形如 `OuterMessage_C_10001.proto`：名称、端标记和起始 opcode 会驱动生成器。`Proto2CS` 会先删除三个 Message 生成目录再整体重建，因此：

- 不在 `Model/Generate/**/Message` 中保留手写代码。
- 修改协议后运行完整 Proto2CS，并检查 Client/Server/ClientServer 差异。
- 使用 `//ResponseType` 和消息接口注释约定声明 RPC/Actor 类型，参考现有 proto。

### 9.3 运行配置加载

- 客户端从 Bundles Config 加载。
- 独立服务端从 `../Config/Excel/s` 加载，StartConfig 类型再拼接 `Options.StartConfig`。
- ClientServer 模式加载相应 `cs` 数据。

修改 StartConfig 后要重新导出；仅编辑 `Config/Json` 不会自动改变实际 bytes。

## 10. 构建、运行与验证

### 10.1 环境

- Unity：`6000.0.43f1`，打开 `Unity/`。
- .NET SDK：项目 TargetFramework 为 `net7.0`。
- C#：10；多个 .NET 工程开启 warnings-as-errors。
- Windows 开发路径避免中文；旧 `Book/1.1运行指南.md` 中的 Unity/.NET 版本提示已过时，但 IDE/Unity 工程生成注意事项仍可参考。

### 10.2 常用构建

在仓库根目录：

```powershell
dotnet build ET.sln
```

也可针对服务端解决方案：

```powershell
dotnet build DotNet/DotNet.sln
```

应优先构建整个相关解决方案，因为 Model/Hotfix、Analyzer、Tool 和 App 存在生成/链接依赖。不要只看到单个项目编译成功就认为双端边界正确。

Unity 动态 DLL 模式：打开 `ET/Build Tool`，确认 CodeMode，再执行 `BuildModelAndHotfix`。`ENABLE_CODES` 模式直接由 Unity 编译，不执行该按钮。

### 10.3 运行

- All-in-One：打开 `Unity/Assets/Scenes/Init.unity` 并 Play；当前 GlobalConfig 为 ClientServer。
- 独立服务端：Unity 菜单 `ET/ServerTools` 最终从 `Bin` 执行近似命令：

```powershell
dotnet App.dll --Process=1 --StartConfig=StartConfig/Localhost --Console=1
```

- Watcher 使用 `--AppType=Watcher`，依据 Machine/Process/Scene 配置拉起进程。
- 服务端主循环不会自行退出。AI 做验证时不要无期限阻塞或遗留后台服务；只有任务确实需要运行时才启动，并负责停止。

### 10.4 生成工具

先保证 `Bin/Tool(.exe)` 已由解决方案构建，再从 Unity `ET/Build Tool` 使用 `ExcelExporter` 或 `Proto2CS`。工具的相对路径假设工作目录为 `Bin/`，随意从其他目录直接运行可能读写错误位置。

### 10.5 验证策略

仓库没有覆盖核心业务的传统单元测试项目；NPBehave 自带的 Editor Tests 属于第三方插件。按修改范围验证：

1. 纯共享/服务端 C#：至少构建相关 .NET 解决方案，确保 ET Analyzer 全部通过。
2. Model/Hotfix/View 或 asmdef：执行 Unity 编译；非 ENABLE_CODES 模式再执行 Model/Hotfix 构建。
3. 配置/协议：运行对应生成器，检查生成差异，再构建 Client、Server 或 ClientServer 相关组合。
4. 场景/资源/UI：运行 Init 场景完成目标流程，检查 Unity Console 与服务端 `Logs/`。
5. 网络/Actor/Scene 拓扑：优先使用 RobotCase 做端到端验证；性能相关使用 Benchmark 配置，不能拿一次本地运行替代正确性检查。
6. HybridCLR 打包：按 `Book/1.1运行指南.md` 的双次打包与 AOT DLL 复制流程，并以当前 HybridCLR 菜单/API 为准。

## 11. AI 执行任务时的工作流程

1. 先读本文件，再读目标目录的 Model 与 Hotfix 成对实现、相关 asmdef/csproj、入口安装点和生成源。
2. 用 `rg` 定位同类 Entity、System、Event、Handler，沿用最近的成熟模式，不凭通用 Unity/服务端经验发明第二套框架。
3. 明确改动属于 Share/Client/Server 和 Model/Hotfix/View 哪一格，再动代码。
4. 区分源文件与生成物；如果任务触及 xlsx/proto，修改源并通过工具再生。
5. 保持改动局部，不顺手升级包、重写第三方源码或格式化无关文件。
6. 修改后查看 `git diff`，特别检查 Unity `.meta`、生成器批量删除重建、DLL/bytes 与配置拓扑差异。
7. 按第 10 节进行与风险相称的构建/运行验证；如果环境缺少 Unity 或 .NET 7，明确说明未执行的验证，不把静态检查说成运行成功。
8. 若发现本文件与当前代码不一致，在任务内同步更新本文件，避免后续 AI 继承过时假设。

## 12. 高风险误区

- 不要把 `Book/` 中旧 API 或版本号直接复制到新代码。
- 不要在 Hotfix 中保存实例状态，或让 Model Entity 承担大段业务流程。
- 不要让共享逻辑引用 `UnityEngine`；Unity 表现应留在 View 层。
- 不要把 Entity 的 `Id` 与 `InstanceId` 混用，也不要在 await 后无检查地继续使用可能已 Dispose 的 Entity。
- 不要绕开 Mailbox/Actor 直接跨 Scene 并发操作 Entity。
- 不要写死 Scene 与 Process 的部署关系；使用 StartConfig 和 SceneType。
- 不要手改 Generate、bytes、动态 DLL、Unity 自动 csproj 来“修复”源头问题。
- 不要同时混用 `ENABLE_CODES` 和动态 DLL 构建流程。
- 不要假设 README 中列出的集成库已经替代 ET 自身的任务、AI 或网络抽象；先搜索当前业务用法。
- 不要提交或删除用户已有的本地改动；开始和结束都检查 `git status`。

