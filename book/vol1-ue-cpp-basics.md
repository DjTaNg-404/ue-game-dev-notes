# 第一卷 · UE C++ 基础

> 本笔记是一份 UE C++ 入门基础沉淀，从「引擎总览」开篇，按「工程基础 → 蓝图与 C++ → 类型与对象模型 → 常用类型 → 内存管理与持久化 → 模块体系 → C++ 语法要点 → 命名规范 → 实战」的顺序组织，覆盖：引擎概念、工程核心文件、蓝图定位与 C++ 分工、宏与反射系统、UObject 对象模型、NewObject/CDO、基类指针与多态、字符串三兄弟（FString/FName/FText）、UE 容器（TArray/TMap/TSet）、智能指针与 GC、序列化、模块拆分原则、#include 规则、运算符（双冒号 / 点 / 箭头）、类定义与 public 继承、命名规范与各类型前缀（F/U/A/S/T/I/C/E/b）。

**目录**
- 引擎总览（开篇）：引擎是什么、它和 C++ 的关系
- 一、工程基础：1.1 UE 工程核心文件 / 1.2 #include 路径规则
- 二、蓝图与 C++：2.1 蓝图定位 / 2.2 C++ 与蓝图分工 / 2.3 Lua 与 Python 的角色
- 三、类型与对象模型：3.1 宏与反射系统 / 3.2 反射深入 / 3.3 UObject 与对象模型 / 3.4 创建 UObject 实例 / 3.5 构造器要轻与 CDO / 3.6 基类指针与多态
- 四、常用类型：4.1 字符串三兄弟（FString/FName/FText）/ 4.2 UE 容器（TArray/TMap/TSet）
- 五、内存管理与持久化：5.1 指针与智能指针 / 5.2 GC 与两套指针体系 / 5.3 序列化
- 六、模块体系：6.1 Agent Harness 类比理解模块拆分
- 七、C++ 语法要点：7.1 运算符 / 7.2 类定义与 public 继承
- 八、命名规范：8.1 PascalCase / 8.2 类型前缀总览 / 8.3 前缀：人看还是编译 / 8.4 模板类 T / 8.5 接口 I / 8.6 枚举 E / 8.7 概念类 C 与布尔 b
- 九、实战：创建并使用一个结构体

---

## 引擎总览（Engine Overview）

学 UE C++ 之前先建立一个总览：我们平时说「引擎」，到底指什么。

一种常见的直觉是「底层是 C++，为了做游戏封装了很多概念，这个封装叫引擎」。这个直觉方向对，但把引擎说小了——引擎不是 C++ 外面薄薄的一层封装，而是用 C++ 造好的一整套「会真正运行起来的游戏世界 + 开发工具」。它包含三块：

- **运行时子系统（真正干活的）**：渲染器、物理、音频、垃圾回收与反射、网络、资源加载，以及那个每帧推进世界的「游戏主循环（Tick）」。这些不是概念，是在后台实打实运行的系统，负责把像素画出来、把碰撞算出来。
- **对象模型 / API（引擎给的词汇）**：UObject、AActor、UActorComponent、AGameMode、UFUNCTION / UPROPERTY 等。这是引擎留给我们的「插口」，让我们能把自己要的东西接进那套运行时里。前面几章学的宏、UObject、基类指针，都属于这一层。
- **编辑器与工具链**：Content Browser、关卡编辑器、蓝图、构建系统（UBT 而不是 CMake）、性能分析器。这一块是「引擎」和一个普通 C++ 游戏库最大的区别——它不只是个能 `#include` 的库，而是一个带完整生产环境的开发平台。

一个关键点要拎清：**C++ 不是被引擎盖住了，而是同时是「材料」和「笔」**。C++ 既是引擎自身的实现语言（引擎底层那几百万行就是 C++），也是你写扩展代码的语言。你写 `class ACat : public AActor` 时，是在 C++ 里继承引擎的 C++ 类，你的代码和引擎代码跑在同一个 C++ 世界里。所以不是「C++ 被封装成引擎、你改用引擎」，而是「C++ 之上长出了引擎，你继续用 C++ 站在引擎肩膀上写」。

类比帮你定型：纯 C++ 从零写游戏，像自己炼钢、自己铺水管电线、从地基盖一栋楼；用 UE，像搬进一栋已经装好结构、水电、电梯的大楼，你只负责按需求布置房间和家具（你的游戏逻辑），用引擎给的接口（UObject / Actor…）接上水电。你之前学的 `BeginPlay` / `Tick`，就是引擎主循环在合适时机「敲门叫你干活」的钩子——这正是「你的代码跑在引擎运行时里」的具体体现。

一句话：引擎是用 C++ 造好的一整套「会运行的游戏世界 + 开发工具」，你用 C++（或蓝图）在它提供的对象模型上写自己的游戏；你学到的那些概念，是这整套机器留给你的「插口」。

---

## 一、工程基础

### 1.1 UE 工程核心文件：.uproject / .h / .cpp / .Build.cs / .generated.h

**`.uproject`**——项目的身份证，一个 JSON 文件，告诉 UE「我是谁、用哪个引擎版本、有哪些模块、启用了哪些插件」。

- `EngineAssociation`：指定引擎版本或路径，双击 `.uproject` 时据此找到引擎启动 Editor。
- `Modules`：列出项目的 C++ 模块名，每个 `Name` 对应 `Source/` 下的同名目录。

**`.h`（头文件）**——声明，管「有什么」：类名、继承谁、成员变量、函数签名、宏。给编译器和别人看的接口。

**`.cpp`（实现文件）**——管「怎么做」：把 `.h` 里声明的函数具体实现出来，写业务逻辑。

这和竞赛里的差别在于规模：竞赛一两百行全塞一个文件没事；工程里几十个类、几百个结构体，不拆分维护不下去。竞赛里类型定义和逻辑混在一起，工程里必须拆成声明（`.h`）+ 实现（`.cpp`）。

**`.Build.cs`**——模块的编译配置，本质是一个 C# 文件，里面是一个"我要用哪些模块"的清单。填的是**模块名**，不是具体的 `.h` 文件名：

```csharp
PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine" });
```

UBT 读到后会把这个模块的**全部公开头文件**纳入可见范围。是**模块级别的依赖声明**，不是逐个 `.h` 去填——一个模块可能包含几百个 `.h`，填模块名就全拿到了。比如你要 `#include "Engine/Engine.h"`，就在 `PublicDependencyModuleNames` 里加 `"Engine"`，否则链接报错。

**`.generated.h`**——UHT 自动生成的反射样板代码。流程：UHT 扫描你的 `.h` → 找到 `UCLASS()`、`UFUNCTION()` 等宏标记 → 针对这些标记生成对应的反射代码（类型信息、序列化、网络复制等）→ 输出为 `.generated.h`。

```
你的 MyActor.h（含 UCLASS、UFUNCTION 等标记）
       ↓ UHT 扫描
UHT 自动生成 → MyActor.generated.h（反射样板代码）
       ↓ 编译
二进制
```

你在 `.h` 末尾写 `#include "XXX.generated.h"` 引入它，**永远不手动编辑**——每次编译时自动重新生成，且必须作为 `.h` 的最后一行 `#include`。

### 1.2 #include 路径规则：什么时候直接写，什么时候加前缀

**日常直觉层（Python 类比）**：同目录直接 `import my_utils`，跨目录需要路径。UE 里也一样——同模块内 `#include "MyStruct.h"` 直接写文件名，跨模块 `#include "Engine/Engine.h"` 加路径前缀。这个直觉日常写代码完全够用。

**底层真实原因**：UBT 编译前根据 `.Build.cs` 算出 include path。流程是：

```
Build.cs 里写 PublicDependencyModuleNames.AddRange(..., "Engine")
    → UBT 把 Engine/Public/、Engine/Classes/ 加入编译器的 include path
    → 你写 #include "Engine/Engine.h"，编译器在 include path 里搜到 Engine/Public/Engine.h
```

所以路径前缀不是「从项目根往下找的目录路径」，而是**模块 Public 目录的别名**。写 `"GameFramework/Actor.h"` 的意思是「在 `GameFramework` 模块的 Public 目录里找 `Actor.h`」。

**实际写法速查**：

| 头文件类型 | 写法 | 原因 |
|---|---|---|
| 同模块内的 `.h` | `#include "MyActor.h"` | 自己模块的目录天然在 include path 里 |
| 其他模块的 `.h` | `#include "Engine/Engine.h"` | 跨模块，路径从该模块 Public 目录起算 |
| C++ 标准库 | `#include <vector>` | 尖括号，系统路径，不走 UE 体系 |
| `.generated.h` | `#include "MyModule.generated.h"` | UHT 生成在 Intermediate 目录里，直接写文件名 |

**总结**：表象是「同模块 vs 跨模块」路径写法不同，真相是 `.Build.cs` 决定了哪些模块的 Public 目录被加进 include path。Python 层面是直觉入口，UE 层面是底层原理，两层都对。

---

## 二、蓝图与 C++

### 2.1 蓝图定位与 C++ 的关系

蓝图是 UE 引擎内部的**可视化脚本系统**，拖拽节点连线就能写逻辑，和 C++ 是「同一种逻辑的两种写法」。

**蓝图 ≠ 外部编排工具**：Dify/Coze 这类工具是独立的外部编排层，调用 API 去驱动外部服务。蓝图不一样——它跑在引擎**内部**，和 C++ 共享同一套 UObject 类型系统、同一个 GC、同一个游戏世界。写成 C++ 的 Actor 和写成蓝图的 Actor 在引擎眼里都是 AActor，互相能调、能传数据、能继承。

**蓝图和 C++ 双向互通**：
- C++ → 蓝图：加 `UFUNCTION(BlueprintCallable)` 让蓝图能调 C++ 函数；加 `UPROPERTY(BlueprintReadWrite)` 让蓝图能读写 C++ 变量。
- 蓝图 → C++：C++ 也能调蓝图里定义的函数（`CallFunctionByNameWithArguments`）、读蓝图变量。一个常见模式是「C++ 写基类（性能关键逻辑），蓝图写子类（策划调数值/搭流程）」。

**蓝图性能**：蓝图编译成字节码，跑在 UE 虚拟机里。比 C++ 大约慢 10 倍，但比 Lua/Python 等外部脚本快（因为是引擎原生字节码）。热点路径可以随时把蓝图节点重写成 C++ 函数。

**反射宏不止为蓝图**：`UPROPERTY()`、`UFUNCTION()` 这些宏的用途有四个方向——蓝图交互只是其一：
1. **蓝图**：`BlueprintCallable`、`BlueprintReadWrite`
2. **编辑器**：让属性在 Details 面板可编辑、可拖拽
3. **GC 追踪**：标记的指针引用，GC 才知道别回收（不加会出野指针）
4. **网络复制**：`Replicated` 说明符让属性自动同步客户端

所以即使不用蓝图，`UPROPERTY()` 也必须加——它是 GC 的「请勿回收」标记。

### 2.2 C++ 与蓝图的职责分工

「蓝图没覆盖的功能用 C++ 写」这个说法不太准——蓝图是图灵完备的，什么逻辑都能搭。用 C++ 不是因为蓝图「做不到」，而是蓝图在某些事上「做不好」：

| 场景 | 蓝图能做吗 | 为什么还是用 C++ |
|---|---|---|
| 大量循环/数学计算 | 能 | 慢约 10 倍，每帧跑会掉帧 |
| 复杂逻辑 | 能 | 节点图变意大利面，改不动 |
| 底层内存/指针操作 | 不能 | 蓝图没有指针概念 |
| 版本控制 | 灾难 | 蓝图是二进制文件，冲突了没法 diff/merge |
| 多人协作 | 难 | 同一个蓝图文件不能两人同时改 |

**正确的分工**：C++ 写地基（骨架、性能关键逻辑），蓝图做表现层（策划调参、搭流程、快速原型）。典型模式是「C++ 写基类，蓝图继承做子类」——BaseCharacter 用 C++ 写移动/血量/伤害，BP_Orc 和 BP_Elf 在蓝图里调数值、换模型、搭技能连招。

### 2.3 Lua 和 Python 在 UE 中的实际角色

- **Python**：UE 自带 Python 插件，但只用于**编辑器工具/自动化**（批量导入资源、编辑器脚本），不跑在游戏运行时里。不是用来写 Gameplay 的。
- **Lua**：UE **默认不内置** Lua 运行时。需要接入第三方插件（UnLua、SLua-Unreal），主要场景是国内手游项目的**热更新**（Lua 脚本可以不下发新包就改逻辑）。

学 UE C++ 期间，Lua/Python 可以当作不存在——它们不是 UE 标准工作流的一部分。

---

## 三、类型与对象模型

### 3.1 宏与反射系统（Macro & Reflection）

宏是 预处理指令，以 #define 开头（也有 #ifdef、#ifndef 等）。在编译之前，预处理器会先把代码中的宏做文本替换，然后再编译。

```c++
  #define PI 3.14159
  #define MAX(a, b) ((a) > (b) ? (a) : (b))

  double area = PI * r * r;           // 编译前变成: 3.14159 * r * r
  int bigger = MAX(x, y);             // 编译前变成: ((x) > (y) ? (x) : (y))
```

结构体相关的宏有：
- USTRUCT()
- GENERATED_USTRUCT_BODY()

这两个宏属于 Unreal Engine 的反射系统。UE 在 标准 C++ 之上加入了一套自己的类型信息机制。

`USTRUCT()`，U = UObject 体系的类型前缀，STRUCT = 结构体。它告诉 UE 的 Unreal Header Tool (UHT)："这个结构体需要被 UE 的反射系统管理"。

```c++
USTRUCT(BlueprintType)   // 括号里可以加"说明符"来控制行为
  struct FMyStruct
  {
      GENERATED_USTRUCT_BODY()
      // ...
  };
```

`GENERATED_USTRUCT_BODY()`，  这个宏展开后是一大堆样板代码，由 UE 自动生成，不用管里面是什么。它大概做了这些事：
  - 生成该结构体的反射类型信息（类型名、大小、对齐等）
  - 生成 StaticStruct() 函数
  - 支持序列化、网络复制、垃圾回收等 UE 特性

本质上宏里面写了很多 C++ 代码，只不过藏在了宏里。

宏其三个功能：
  1. 标记：告诉 UHT 工具“来处理这个结构体”
  2. UHT：读到标记后，自动写一大堆 C++ 底层代码到 .generated.h 文件里
  3. 展开：编译时，宏替换成这些自动生成的代码

UE 的宏 = 标记 + 自动生成 + 编译时展开。它不是普通的 `#define PI 3.14` 那么简单，而是 UE 用来扩展 C++、加入反射/序列化/蓝图交互等能力的一套代码生成机制。

### 3.2 反射深入：为什么需要"运行时照镜子"

**核心原因：竞赛程序是一次性的，UE 程序是持续运行的。**

竞赛程序编译完 → 跑一遍 → 输出结果 → 结束。运行时不需要知道「Cat 类有哪些成员变量」，反正代码是你写的，全在脑子里。类型信息在编译期存在，编译完就丢了，不影响。

UE 程序编译完 → 编辑器一直开着，游戏一直跑着。这时候必须能回答：
- 编辑器要在 Details 面板里显示这个 Actor 有哪些可调属性 → 需要知道类型信息
- 蓝图要拖出某个函数的调用节点 → 需要知道函数签名
- GC 要判断这个 UObject 还有没有人引用 → 需要知道指针关系
- 存档时要遍历所有需要保存的变量 → 需要知道哪些属性该序列化

这些事全发生在**运行时**，而标准 C++ 编译完后类型信息就没了——所以 UE 自己造了一套：**反射系统**。

**宏 + UHT + .generated.h 的协作流程：**

```
你写的 MyActor.h
  ├─ UCLASS()          ← 标记：这个类需要反射
  ├─ UPROPERTY()       ← 标记：这个属性需要反射（编辑器可见、GC 追踪、序列化）
  └─ UFUNCTION()       ← 标记：这个函数需要反射（蓝图可调）
         ↓ 编译前，UHT 扫描
UHT 读取这些宏 → 生成一份「类型身份证」→ MyActor.generated.h
  身份证里写明了：类名叫 "MyActor"、父类是 AActor、属性有 Health(int32)、函数有 TakeDamage(int32)
         ↓ 运行时
蓝图/编辑器/GC/序列化系统 → 查身份证 → 知道这个类的一切
```

**反射 ≠ 宏**。宏是「照这儿」的标记，反射是 UHT 根据标记生成的那套运行时可查询的类型信息系统。前者是因，后者是果。

### 3.3 UObject 与对象模型

之前我们讲了 `USTRUCT()` 是给结构体办的「登记证」，让 UHT 生成反射代码。UObject 是另一回事：它是 UE 自己提供的一个 **C++ 基类**，是所有「引擎托管对象」的共同祖先。

这里要把两件事分清，别混在一起：

- **宏 = 登记证**：`UCLASS()` / `USTRUCT()` 告诉 UE 「这个类/结构体需要被反射系统管理」。
- **UObject = 被托管的基类**：`UCLASS class AMyThing : public UObject` 这种写法里，你用 `UCLASS()` 登记，同时**必须继承 UObject**。继承了它，你的类才「进了引擎系统」——引擎替你管生死（垃圾回收 GC），并提供序列化、网络同步、编辑器显示等一整套服务。

而 `USTRUCT struct FFoo` 登记的是**值类型**，它**不继承 UObject**，像 `std::pair` 那样轻量、按值拷贝、没有 GC；但它一旦作为 `UPROPERTY` 嵌在某个 UObject 里，就会被一起序列化和 GC 跟踪。

### 3.4 创建 UObject 实例：NewObject 与 CreateDefaultSubobject

`NewObject<T>()` 创建的是一个**活的、被引擎接管的 UObject 实例**：引擎知道它存在、能垃圾回收它、能把它序列化存盘、能在编辑器里显示它的属性、能网络同步它。

`CreateDefaultSubobject<T>()` 只能在**构造器里**调用，创建的是「作为某个 Actor 组成部分的默认子对象」（比如一个 SceneComponent、一个碰撞体）。它被写进 Actor 的模板里、在编辑器里可见，每个该 Actor 的实例都共享这套部件。

### 3.5 构造器要轻，以及 CDO 机制

引擎启动时，会给每个 `UCLASS` 用默认构造器造一个「**类默认对象**」(CDO，Class Default Object) 作为所有运行时实例的原型。这件事带来三条铁律：

1. **必须有无参默认构造器**，否则编译不过——因为引擎要靠它先造 CDO 模板。
2. **构造器只能设默认值、挂子对象**，不能做真正的逻辑（比如读文件、调别的系统、算业务）。否则每个实例都会重跑、而且编辑器和运行实例共享同一个模板，会出乱子。
3. **真正的初始化放到 `BeginPlay()`**（Actor）或 `PostInitProperties()`（其他 UObject）。

**防坑**：不要手动 `delete` UObject，它交给 GC 管，你只负责 `NewObject<>()` 和持有 `UPROPERTY()` 引用；容器里装基类指针要用 `TArray<UObject*>` + `UPROPERTY`，别用 `std::vector`（GC 看不见，会把对象回收掉还留野指针）。

### 3.6 基类指针与多态：先定义「Object」不指定具体类型

UObject 是大家共同继承的基类，Actor、Character 等都从它派生。所以我们可以**先声明一个基类指针，运行时再赋具体派生实例**，需要具体能力时再向下转——这正是 C++ 多态在 UE 里被用到极致的地方：

```cpp
UPROPERTY()
UObject* SomeRef;                 // 最通用的基类指针，可指向任意 UObject 派生实例
SomeRef = NewObject<UMyDataAsset>();

AActor* HitActor = ...;           // 射线打中的东西，类型未知
ACharacter* Char = Cast<ACharacter>(HitActor);   // 安全向下转型
if (Char)
{
    Char->GetCharacterMovement()->...   // 转成功才调用 Character 专属接口
}
```

`Cast<T>()` 是 UE 的「安全向下转型」，转失败返回 `nullptr`，所以一定要判空。

一个小提醒：UObject 体系里的对象几乎都在堆上、由 GC 管，你声明的是**指针**（`UObject*`）不是栈上的值——它是指针语义，不是 C++ 里 `int x` 那种值类型。游戏里最常用、最通用的基类指针其实是 `AActor*` / `UActorComponent*`，`UObject*` 太泛，一般只在「要引用任意资源或任意对象」时才用。

---

## 四、常用类型

### 4.1 字符串三兄弟：FString / FName / FText

三者的选型核心看**用途**：

- **FString**：正常的、可变的、常用的字符串类型。日志、拼接、文件路径、临时消息都用它。相当于 UE 里的 `std::string`。
- **FName**：用作标识符，大量重复使用、需要频繁比较的场景（资源名、属性名、tag）。底层是哈希索引，O(1) 比较。不要求内容"固定不动"——也可以动态创建 `FName(FString::Printf(TEXT("Enemy_%d"), id))`，关键在**需要不需要频繁比较**和**能不能接受它永远占着注册表**。
- **FText**：输出给玩家的文本，方便未来换各种翻译版本。UI、对话、任务描述等玩家肉眼能看到的都用它。

**互转**：`Name.ToString()` / `FName(*Str)` / `Text.ToString()` / `FText::FromString(Str)`。

**关键注意**：UE 字符串底层用 `TCHAR` 不是 `char`，`TEXT()` 宏做跨平台适配，日志输出时 `*Str` 取出 `const TCHAR*`。

**代码样例**：

```cpp
// ========== FString：可变字符串，最常用 ==========
FString Name = TEXT("Player_001");                     // 创建：TEXT() 包裹
FString Msg = FString::Printf(TEXT("HP: %d"), 100);    // 格式化
FString Full = Name + TEXT(" -> ") + Msg;              // 拼接
int32 Len = Name.Len();                                // 长度
bool bHit = Msg.Contains(TEXT("HP"));                  // 是否包含子串
UE_LOG(LogTemp, Warning, TEXT("Log: %s"), *Full);      // 日志输出：* 解引用

// ========== FName：哈希标识符，O(1) 比较 ==========
FName Tag1 = TEXT("Interactable");                     // 创建：同样是 TEXT() 包裹
FName Tag2 = TEXT("Interactable");                     // 相同字符串只存一份
bool bSame = (Tag1 == Tag2);                           // true，比较的是索引（整数）
FName DynTag = FName(FString::Printf(TEXT("Enemy_%d"), 5)); // 也可以动态创建

// ========== FText：玩家可见，可翻译 ==========
FText Welcome = NSLOCTEXT("MyGame", "Welcome", "欢迎！");     // 可本地化
FText Score = FText::Format(
    NSLOCTEXT("MyGame", "Score", "分数: {0}"),
    FText::AsNumber(999)
);

// ========== 互转 ==========
FString FromFName = Tag1.ToString();                   // FName → FString
FName FromFString = FName(*Name);                      // FString → FName（*取出 TCHAR*）
FText FromStr = FText::FromString(Name);               // FString → FText
FString FromFText = Welcome.ToString();                 // FText → FString（返回当前语言翻译）
```

**核心规律**：所有字符串字面量都用 `TEXT("...")` 包裹，因为 UE 底层是 `TCHAR`（不是 `char`），TEXT() 在不同平台自动转成正确的字符编码。

### 4.2 UE 容器：TArray / TMap / TSet

**为什么不直接用 STL？** 关键原因是 **GC 可见性**：`std::vector<UObject*>` 里的指针 GC 看不见，对象会被当成垃圾回收，留下野指针。必须用 UE 容器 + `UPROPERTY()` 标记才能让 GC 正确追踪。此外 UE 容器自带序列化、网络复制、蓝图互操作。

**与 STL 对应关系**：

| STL | UE | 说明 |
|---|---|---|
| `std::vector` | `TArray` | 动态数组 |
| `std::unordered_map` | `TMap` | 哈希表，**无序**（不是红黑树） |
| `std::unordered_set` | `TSet` | 哈希集合，只有键没有值 |

**命名差异速查**：

| 操作 | STL | UE |
|---|---|---|
| 元素个数 | `.size()` | `.Num()` |
| 尾部添加 | `.push_back()` | `.Add()` |
| 清空 | `.clear()` | `.Empty()` |
| 预分配 | `.reserve()` | `.Reserve()` |
| 检查包含 | 无内置 | `.Contains()` |

**TArray 常用**：
```cpp
TArray<int32> Scores;
Scores.Add(100);                // 尾部添加
Scores.Insert(150, 1);          // 索引 1 插入
Scores.Remove(150);             // 按值删
Scores.RemoveAt(0);             // 按索引删
Scores.Pop();                   // 删尾部并返回
int32 Idx = Scores.Find(200);   // 返回索引，找不到返回 INDEX_NONE(-1)
Scores.Sort();                  // 升序排序
Scores.Reserve(100);            // 预分配容量

for (int32 Score : Scores) { }              // 范围 for
for (const FString& Name : Names) { }       // 引用遍历避免拷贝
```

**TMap 常用**：
```cpp
TMap<FString, int32> HealthMap;
HealthMap.Add(TEXT("Orc"), 100);
HealthMap.Emplace(TEXT("Dwarf"), 120);      // 原地构造
HealthMap[TEXT("Orc")] = 150;               // 覆盖/新增
int32* Ptr = HealthMap.Find(TEXT("Orc"));   // 返回指针，查不到 nullptr
int32 Safe = HealthMap.FindRef(TEXT("X"));  // 查不到返回默认值
HealthMap.Remove(TEXT("Orc"));

// ⚠️ TMap[key] 下标访问找不到会断言崩溃！不确定存在时先 Find() / Contains()
for (auto& [Key, Value] : HealthMap) { }    // C++17 结构化绑定遍历
```

**TSet 常用**：只关心"某个东西在不在"，如 tag 标签：
```cpp
TSet<FName> Tags;
Tags.Add(TEXT("Interactable"));
Tags.Add(TEXT("Interactable"));             // 重复自动忽略
bool bHas = Tags.Contains(TEXT("Breakable"));
Tags.Remove(TEXT("Breakable"));
```

**TSet vs TMap**：结构和使用方式几乎一致，唯一区别是 TMap 多存一个"值"（键→值的映射），TSet 只有键。关心"键能取回什么"用 TMap，只关心"存不存在"用 TSet。两者都是哈希实现，无序。

**GC 相关（最重要）**：
```cpp
// ❌ GC 看不见，UObject 可能已被回收
std::vector<AActor*> Enemies;

// ✅ GC 看得见，安全
UPROPERTY()
TArray<AActor*> Enemies;
```
任何存 UObject 指针的容器必须用 UE 容器 + `UPROPERTY()`。纯值类型（int32/float/FString/FVector）不需要。

**边遍历边删除**：直接 `RemoveAt` 会让数组移位导致跳元素或越界，用迭代器：
```cpp
for (auto It = Scores.CreateIterator(); It; ++It)
{
    if (*It == 0) { It.RemoveCurrent(); }   // 安全删除当前元素并继续
}
```

---

## 五、内存管理与持久化

### 5.1 指针与智能指针

#### 指针基础（为什么需要指针）

- 指针变量存的是**某个变量的地址**（`int32* Ptr = &HP;`，`*Ptr` 解引用取值/改值）。
- **工程上用指针的原因**：不搬数据，只传地址。反复传值会导致一堆内存空间重复存同样的值，且拷贝/释放浪费时间。传地址则直接对原位置修改，省内存、省时间。竞赛对内存/时间要求不高所以无所谓，工程上这是核心需求。
- **手动指针的噩梦**：new 了必须记得 delete。忘了 → 内存泄漏（空间一直占着）；删早了 → 悬空指针；删两次 → 崩溃。中途提前 return 也容易漏。

**指针的两层结构**（容易混淆的点）：
- 指针变量本身 = 存地址的小格子（8 字节）
- 它指向的内存空间 = new 出来的那块大空间
- 两者寿命不同：指针变量出作用域就销毁，但**指向的空间不会自动销毁**（除非 delete）

#### 智能指针：自动 delete 的指针

- 智能指针是**一个类**，封装了普通指针。核心机制：**对象离开作用域时自动调用析构函数，析构函数里执行 delete**。
- 作用域 = 一对花括号 `{}` 围起来的区域。变量出花括号就销毁。函数体本身就是一个作用域。
- 出作用域后：
  - 普通指针：指针变量销毁，但指向的内存泄漏（箭头没了，靶子还立着）
  - 智能指针：指针变量销毁，**顺带把指向的内存也释放**（箭头没了，靶子也收了）
- 最大的价值：无论从哪个路径离开作用域（正常返回 / 中途 return / 抛异常），销毁一定会发生，释放一定会执行，不需要手动管理。
- 注意：**"指针指向别的空间"不会触发释放**。触发释放的是智能指针对象本身的生命周期结束（出作用域）。

#### 三种智能指针（UE 版）

| UE 智能指针 | 对应 std | 语义 | 适用 |
|---|---|---|---|
| `TSharedPtr` | `std::shared_ptr` | 共享所有权，引用计数 | 多个系统共享的资源 |
| `TUniquePtr` | `std::unique_ptr` | 独占所有权，可转移不可复制 | 单一主人拥有的对象 |
| `TWeakPtr` | `std::weak_ptr` | 弱引用，不计数，能检测存活 | 持有"可能已死"的引用 |

**TSharedPtr（共享所有权）**：多个指针指向同一个对象，谁都不能独占。引用计数：多一个指向 +1，一个指向销毁 -1，计数归 0 时最后一个销毁的负责 delete。适用于多个系统共享的资源（一份配置、一个管理器）。

```cpp
TSharedPtr<FMyData> A = MakeShared<FMyData>();   // 计数=1
TSharedPtr<FMyData> B = A;                        // 计数=2
A.Reset(); B.Reset();                             // 归 0 → 自动 delete
```

**TUniquePtr（独占所有权）**：只有一个指针拥有对象。可转移不可复制（`MoveTemp` 转移所有权，类似 `std::move`）。适用于对象只属于一个主人，主人销毁它就销毁。

**TWeakPtr（弱引用）**：不参与引用计数、不阻止对象销毁，只是"看着"。核心用途：安全检测对象是否还活着，防止悬空指针崩溃。
- 平时：只是看，不计数、不阻止销毁
- 用时：`Pin()` 尝试锁住——对象还活着 → 返回临时 TSharedPtr 可以用；对象已死 → 返回空，安全跳过
- 类比："凭记忆找门的人"——门在就进去做事，门没了就走开不撞墙。绝不"扶着门不让拆"（那是 TSharedPtr 做的事）
- 适用：持有"可能已经死了"的引用（如 UI 持有场景里的 Actor，场景切换后 Actor 没了，用 TWeakPtr 不会崩）

```cpp
TWeakPtr<FMyData> Weak = Shared;                  // 只是看着，计数不变
if (auto Locked = Weak.Pin())                     // 活着 → 返回共享指针
{
    Locked->DoSomething();
}
else { /* 已经死了，安全跳过 */ }
```

### 5.2 GC 与两套指针体系

#### 和 UObject / GC 的关系（UE 指针体系两大阵营）

```
继承 UObject 的类（UObject/AActor 等）  → UObject* + UPROPERTY() → GC 管，不能用智能指针包
普通 C++ 类/结构体（F 开头，纯数据）    → TSharedPtr/TUniquePtr/TWeakPtr → 智能指针管
```

**为什么 UObject 不能用智能指针包**：UObject 已由 GC 管理生命周期，智能指针的引用计数和 GC 是两套系统，会互相打架（引用计数说没人用但 GC 还活着 / 反过来提前销毁）。**一个对象只归一套系统管。**

```cpp
// ✅ UObject：UPROPERTY() + TObjectPtr/原始指针，GC 管
UPROPERTY()
TObjectPtr<AActor> TargetActor;

// ❌ UObject 不能用智能指针包（两套系统打架）
TSharedPtr<AActor> BadActor;
```

**核心口诀**：UObject 体系（U/A 开头）→ `UPROPERTY()` + GC；非 UObject（F 开头）→ 智能指针。

#### GC 到底是什么

**GC = Garbage Collection（垃圾回收）**：一种**自动管理内存**的机制。

- 普通 C++：`new` 必须手动 `delete`，忘了就泄漏。
- UObject 世界：`NewObject<>()` 创建后什么都不用写，**GC 定期自动扫描回收**。

**GC 怎么决定回收谁**：引擎定期扫描"这个对象还有没有被引用"。被 `UPROPERTY()` 标记的指针引用着 → 活着保留；没有任何引用 → 是垃圾，回收释放。`MyData = nullptr;` 不再引用 → 下次 GC 扫描发现没人要就回收。

**类比**：GC 像扫地机器人，定期检查房间：有人坐着的椅子不动，没人的椅子收走。

#### 为什么分成两套系统（核心）

| | UObject（GC 管） | 非 UObject（智能指针管） |
|---|---|---|
| 创建 | `NewObject<>()` | `MakeShared<>()` / `MakeUnique<>()` |
| 销毁 | GC 扫描，没人引用就回收 | 引用计数归 0 / 独占者销毁 |
| 谁决定死活 | 引擎的 GC 后台机制 | 代码里的作用域和引用关系 |
| 管理方式 | `UPROPERTY()` 标记引用 | 智能指针本身 |

**根本区别：生命周期由谁说了算。**
- UObject：生命周期归**引擎**管（GC）。引擎需要它做序列化、蓝图、网络复制等全套服务，所以引擎必须掌控它。
- 非 UObject（纯 C++ 数据）：生命周期归**你的代码**管。引擎不关心它，用 C++ 自己的智能指针机制。

**为什么不能混用**：给 UObject 套智能指针 = 两套系统同时决定一个对象的死活（GC 说回收，智能指针说还活着 / 反过来）。**一个对象的命运只能由一套系统决定。**

**GC 是 UE 在 C++ 基础上自己加的机制，标准 C++ 没有**。C++ 程序员的"自动管理"方案本来就是智能指针（`std::shared_ptr`/`std::unique_ptr`）。所以：
- 非 UObject → 智能指针 = 沿袭 C++ 标准做法
- UObject → GC = UE 额外加的机制

逻辑链：**UE 有 GC（C++ 没有）→ 所以 UObject 特殊 → 所以分两套管理**。

### 5.3 序列化（Serialization）

**定义**：把内存中"活的、连着的"对象（结构体、指针、嵌套关系），变成一条**连续的字节流**存到磁盘；反序列化是反向——从字节流**重建**出原对象。

**核心点**：
- 不是"有序"这个动作本身，而是**存的内容必须包含重建所需的全部细节**，能 100% 还原对象。
- 顺序本身就是重建需要的"细节"之一，所以要按固定规则存。

**四个应用场景**：
| 场景 | 说明 |
|---|---|
| 存档/读档 | 玩家进度存盘，下次打开还原 |
| 网络传输 | 对象变字节流发到服务器/其他客户端，对方反序列化还原（游戏同步就是靠这个） |
| 编辑器保存 | 蓝图、关卡等资产本质是序列化后的数据 |
| 配置/数据传递 | 跨系统统一成字节流 |

**和 UE 的关系**：`UPROPERTY()` 就是序列化的一部分——标记过的属性，引擎知道"存盘时要存这个变量"，存哪些、怎么存由引擎代劳。所以 UE 容器"自带序列化"是指 `UPROPERTY()` + UE 容器组合后能被引擎正确保存/网络复制。

**序列化的两种实现机制**：

1. **基类虚函数方案**：在基类加一对虚函数（`Serialize`/`Deserialize`），每个派生类自己实现，自己说明如何序列化。灵活但每类都要写，代码重复。
2. **反射系统方案**：写一个**通用**序列化器，靠反射信息自动遍历"这个类有哪些属性"并自动存取。UE 用的就是这种——`UPROPERTY()` 就是反射标记，告诉引擎存哪些字段。

**关于序列化格式**：
- XML/JSON/二进制 是**序列化的输出格式**（存成什么样子）。
- 反射系统决定"存哪些字段"，格式决定"存成什么形态"，两者是**正交维度**，可自由组合。

**UE 的格式选择**（不是"从 XML 换到 JSON"）：
| 格式 | 用在哪 |
|---|---|
| 二进制（数据流/Archive） | 游戏资产（.uasset/.umap）、存档、网络同步——性能优先，**主要格式** |
| JSON | 配置、数据表导入导出、编辑器工具——人可读、易调试 |

XML 冗长 + 解析慢，不适合性能敏感场景（如加载关卡），所以游戏引擎用自定义二进制或 JSON。

**修正一个容易错位的图景**：源码不参与序列化这条线。正确理解：
```
[编译期] 源码.cpp ──编译──▶ 机器码/引擎字节码     ← 翻译，不是序列化
[运行期] 内存中的游戏对象 ──序列化──▶ 磁盘文件     ← 序列化发生在这里
```
编译 = 把源码翻译成机器码（一次性的翻译）；序列化 = 运行期内存对象 ↔ 磁盘文件（可反复来回）。

**Encoder/Decoder 类比**：序列化 ≈ Encoder（对象 → 格式的数据流），反序列化 ≈ Decoder（数据流 → 对象）。逻辑完全一致：A 格式 → 转换 → B 格式 → 转回 → A 格式。

**关键差别：序列化必须"保真"（无损）**。Encoder/Decoder 可以有损（压缩、降采样），但序列化严格要求 100% 还原对象——"存的内容必须包含重建所需的全部细节"。所以准确说法是：
> 序列化 = **无损编码**（对象→格式），反序列化 = **无损解码**（格式→对象）。

---

## 六、模块体系

### 6.1 模块拆分的工程直觉：用 Agent Harness 类比

我们用 AI Agent 应用的架构来类比 UE 模块体系，可以很直观地理解「为什么要拆模块」。

**Agent Harness（智能体运行时基础设施层）**里，Agent Runtime（核心推理循环）只管「理解任务 → 选工具 → 执行」，而以下东西不该塞进 Runtime：
- Token 限制额度管理
- Prompt 安全审查
- 输出内容审核
- 日志、监控、限流

这些功能塞进 Runtime 也能跑，但代码会变成一坨——因为它们和核心推理循环是**不同职责**。正确的做法是：Runtime 只干核心推理，审查/限流/监控各自写在 Runtime 外面，共同组成 Harness。

**UE 游戏完全一样**。一个游戏项目不是「一个大的 Runtime 模块」，而是由多个模块组成的「运行时基础设施层」：

| Agent 世界 | UE 游戏世界 |
|---|---|
| Agent Runtime（核心推理循环） | 主 Game Module（核心玩法逻辑） |
| Token 限制额度 | UMG/Slate UI 模块 |
| Prompt 安审 | 输入系统 / Online Subsystem 模块 |
| 输出内容审查 | 网络同步 / 数据验证模块 |
| 日志、监控、限流 | 分析、存档、调试工具模块 |

这些确实都不该塞进 Gameplay 模块里——**塞进去能跑，但代码框架会很杂很丑**。

**一个精确修正**：Harness 不仅仅是「你的那些模块」，还包括 UE 引擎自带的 `Core`、`Engine`、`Renderer`、`Slate` 等底层模块。它们提供了游戏 Runtime 运行所需的基础设施（就像 Agent Harness 提供了 token 管理、安审等基础设施）。所以准确说：**引擎底层 + 你的多个模块 = Harness**，你的 Gameplay 代码跑在里面。

**工程原则**：不是功能上做不到放一起，而是**可维护性上不应该**。一个模块 = 一个编译边界 = 一个职责边界。这个认知比「模块怎么写」更重要。

---

## 七、C++ 语法要点（UE 语境）

### 7.1 运算符：双冒号、点、箭头

#### `::`（双冒号）= 作用域解析

`::` 叫**作用域解析运算符**，意思是「去左边这个作用域里，找右边这个名字」，和你用过的文件路径是同一个直觉——`std::vector` 就约等于「`std` 目录下的 `vector`」，而 `::` 就是那个 `/`。

三种常见用法：

1. **命名空间**：`std::cout`、`std::vector` —— `std` 是把标准库名字装进去的「名字盒子」。
2. **类/结构体里的嵌套类型或静态成员**：`ConstructorHelpers::FObjectFinder` —— `ConstructorHelpers` 是 UE 提供的一个结构体，`FObjectFinder` 是它里面定义的嵌套模板类型。所以那行 `static ConstructorHelpers::FObjectFinder<UStaticMesh> CubeVisualAsset(TEXT("..."));` 就是「在 `ConstructorHelpers` 里找到 `FObjectFinder`，用 `UStaticMesh` 当模板参数实例化」，本质是一个按路径查找静态网格资源的加载器。
3. **全局作用域**：单独的 `::Func()` 表示去最外层全局找这个名字。

`using namespace std;` 只是「默认也去 `std` 里翻」的省略写法，跟 `::` 本身无关；UE 工程几乎不写 `using namespace`，全写显式 `XXX::YYY`。

#### `.` 和 `::` 和 `->` 的区别

- **`.`（点）**：左边是**具体对象实例**，访问「这个对象的成员」。运行时操作，每个对象各有一份。
- **`::`（双冒号）**：左边是**类名/结构体名/命名空间**，访问「属于这个类型/作用域本身的名字」（静态成员、嵌套类型、枚举）。是编译期概念，不依赖任何实例。
- **`->`（箭头）**：左边是**指针**，等价于 `(*ptr).成员`。因为 UE 的 UObject 全是指针，你在工程里反而最常用 `->`。

一个容易绕进去的误区要拎出来：**运算符选哪个，跟「打比赛还是做工程」没关系，只看左边是实例还是类型。** 工程里照样用 `->` 操作具体对象（`Player->Health -= 10;`），纯算法里照样用 `::`（`std::sort(...)`）。

而且 `::` 访问的（静态成员/类型）恰恰是不随某个实例变、所有实例共享的；**会跟着这个 Player 一起变的那些实例成员（Health、Stamina…），正是用 `->` 访问的**——所以「封装让成员作为整体一起变」这件事，在语法上体现为「你操作的是 `Player` 这个实例（`Player->Health`），而不是类（`APlayerCharacter::`）」。

### 7.2 类定义里的冒号与 public 继承

定义类时，`class AMyActor : public AActor { ... };` 这一行里：
- `class AMyActor` → 定义一个叫 `AMyActor` 的类；
- `:` → **冒号是「继承说明符」**，引出基类列表，意思是「这个类的基类是……」（冒号 ≈ 继承自）；
- `public` → 继承的**访问方式**（继承方式），表示公开继承：`AActor` 的 public/protected 成员在 `AMyActor` 里还是 public/protected，保留「is-a」关系；
- `AActor` → 基类；
- `{ ... }` → 类体，放成员（变量、函数）；
- `;` → 类声明结尾。

一个容易混的点：`class X : public Base` 里的 `public` 和类体里的 `public:` 是两回事：

```c++
class AMyActor : public AActor   // ← 这里的 public 是「继承方式」
{
public:                         // ← 这里的 public: 是「成员可见性」
    int Health;
};
```

为什么 UE 一定要写 `public`：`class` 默认的继承方式是 **private**（不写 public 就变成私有继承，基类的对外接口在子类里不再对外），这对 UE 这种「到处用基类指针、靠反射」的框架是灾难。所以 UE 永远写 `public` 继承：

```c++
class AMyActor : AActor { ... };        // ❌ 默认 private 继承，不是你想要的
class AMyActor : public AActor { ... }; // ✅ 正确
```

冒号后还能跟多个基类（多重继承 `class X : public A, public B`），但 UE 的 UObject 体系不鼓励对 UObject 类用多重继承，一般只单继承一条链（UObject→AActor→ACharacter）。上面那行的 `UCLASS()` 是反射标记，跟冒号语法无关。

---

## 八、命名规范（Naming Convention）

### 8.1 PascalCase 与基本约定

UE 有一套强制命名约定：所有代码和注释用美式英语；命名（类型名、变量名）用 PascalCase——每个单词首字母大写、单词间无下划线，例如 `Health`、`UPrimitiveComponent`。`lastMouseCoordinates` 或 `delta_coordinates` 这种带下划线或驼峰的写法都是错的。

### 8.2 类型前缀总览

每个类型名带一个大写字母前缀，用来区分它和变量，也用来标记它的类别。前缀是**类型本名的一部分**（不是调用时现加的）：

| 前缀 | 含义 | 例子 |
|---|---|---|
| F | 普通类型 / 值类型（struct） | FVector、FSkin |
| U | 继承自 UObject | UActorComponent |
| A | 继承自 AActor | AActor、ACat |
| S | 继承自 SWidget | SCompoundWidget |
| T | 模板类（泛型） | TArray、TAttribute |
| I | 抽象界面类（接口 interface） | IAnalyticsProvider |
| C | 概念类（concept class，模板元编程约束） | CStaticClassProvider |
| E | 枚举类型（enum） | EColorBits、EDirection |
| b | 布尔变量（bool） | bPendingDestruction、bHasFadedIn |

一个具体的例子：如果你有一个概念叫「猫」——
- 它是关卡里能放进去的实体（从 `AActor` 派生）→ 命名为 `ACat`；
- 它只是一堆数据（年龄、品种），是个普通结构体 → 命名为 `FCat`。

前缀选哪个，取决于「它是什么」，不是你喜不喜欢。前缀还顺带避免了和基类重名（你不会把自己的类命名为 `Actor` 去撞 `AActor`）。

### 8.3 前缀：为人看还是为编译？

主要是**给人看**：一眼知道类别、避免重名。但对**带反射标记的类**（`UCLASS()` / `USTRUCT()`），UHT（Unreal Header Tool）会在编译期**强制**检查前缀是否和基类匹配，不匹配直接报错、编译不过。比如：

```cpp
UCLASS()
class MyActor : public AActor { ... };   // ❌ UHT 报错：继承 AActor 必须叫 AMyActor
```

反过来，不加 `UCLASS` 的纯 C++ 辅助类，前缀纯是约定，编译器不在乎。所以：**前缀本质是给人设计的约定；UE 通过 UHT 把这条约定对「反射类」升级成了编译红线。**

### 8.4 模板类前缀 T：带「类型插槽」的类

`T` 前缀标记的是模板类。模板类就是你打比赛天天用的东西——`std::vector`、`std::pair`、`std::map` 全都是模板类。它自己不指定装什么类型，而是留一个占位符（UE 里常叫 `ObjectType`，STL 里叫 `T`），等你用时再把具体类型填进去：

```cpp
template <typename ObjectType>   // ObjectType 是占位符，不是具体类型
class TAttribute { /* ... */ };

TAttribute<float>   Health;      // 填 float   → 一个装 float 的属性
TAttribute<FString> Name;        // 填 FString → 一个装字符串的属性
```

填进去的过程叫**模板实例化**，发生在编译期——编译器替你基于 `TArray<float>` 生成一份真正的「装 float 的动态数组」代码。你比赛里写的 `std::vector<int>`，就是给 `std::vector` 这个模板类填了 `int`。

**关键**：T 前缀和 U/A/S 不是一回事。U/A/S 看「继承谁」，T 看「怎么定义的」（是不是泛型）。`TArray`、`TMap`、`TSharedPtr` 这些不是 UObject 的子类，只是泛型工具类，所以打 T。实践里 UObject 类几乎不可能是模板，所以你不会看到 `TUObject`。

### 8.5 抽象界面类 / 接口前缀 I

`I` 前缀标记的是**抽象界面类**，在 UE 文档里中文常叫「抽象界面类」，本质就是**接口（interface）**。它解决的是这样一个问题：想给一批**互不相干的类**都加上同一种「能力」，但又不想让它们都去继承同一个基类（C++ 只能单继承，基类链早就定好了）。

接口就是一份「**能力契约**」：它只**声明**「能做什么」（一组函数），**不实现**（或只给默认空实现）。谁想要这个能力，谁就去**实现**它。比如 `IAnalyticsProvider` 声明了「能记录事件」（`RecordEvent()`），然后八竿子打不着的 `AGoogleAnalytics` 和 `AUnityAnalytics` 两个类都去实现它——它们从此都能被当作「分析提供者」来用，但彼此之间没有任何继承关系：

```cpp
class IAnalyticsProvider        // I 前缀 = 接口类
{
public:
    virtual void RecordEvent(const FString& Name) = 0;  // 只声明「能做什么」，不实现
};

class AGoogleAnalytics : public AActor, public IAnalyticsProvider
{
    virtual void RecordEvent(const FString& Name) override { /* 谷歌的具体实现 */ }
};

class AUnityAnalytics : public AActor, public IAnalyticsProvider
{
    virtual void RecordEvent(const FString& Name) override { /* Unity 的具体实现 */ }
};
```

接口本身**不能实例化**（它是抽象的，只是一份约定）。它的好处是：写代码时可以「面向接口编程」，不关心具体是哪个类。比如任何实现了 `IDamageable` 的 Actor，你都能调它的 `TakeDamage()`，管它是玩家还是箱子。UE 里接口有两种：纯 C++ 接口（`I` 前缀 + 虚函数，不进反射）和 `UINTERFACE()`（让接口也能被蓝图/反射识别）。前期写 Gameplay 先知道 `I` = 接口即可。

### 8.6 枚举前缀 E

`E` 前缀标记的是**枚举类型**。枚举可以理解成一组**带名字的取值**：变量同一时刻只能取其中一个。它是你打比赛时 `const int UP=0, DOWN=1, ...` 那套写法的**类型安全升级版**——编译器知道这个变量只能取那几个值，不会让你不小心赋个 `5` 进去，调试时直接显示 `Up` 而不是数字 `0`。

最经典的例子就是角色移动的方向：上、下、左、右四个方向，正适合用枚举来表示：

```cpp
enum class EDirection   // E 前缀；enum class = 强类型作用域枚举（UE 用的形式）
{
    Up,
    Down,
    Left,
    Right
};

EDirection Current = EDirection::Up;   // 变量某刻只取其一（这里取 Up）
```

`enum class`（C++11 起的「枚举类」）叫「作用域枚举」——名字必须带作用域写（`EDirection::Up`），不会污染全局命名空间。你贴的 `EColorBits` 里 `ECB_Red` 就是这种写法：枚举类型带 `E` 前缀，枚举值（成员）也带自己的前缀（`ECB_`，取自类型名）。老式 `enum`（不带 class）名字会裸奔到全局、容易撞名，UE 不推荐。

所以这个直觉「四个方向用枚举」完全正确。以后写角色朝向、状态机（Idle/Run/Jump）、道具类型，全都是枚举的用武之地。

### 8.7 概念类前缀 C 与布尔前缀 b

- **`C` 前缀 = 概念类（concept class）**：你贴的 `CStaticClassProvider` 属于 C++20 `concept` / SFINAE 那套玩法，用来在**编译期**约束「某个类型必须满足什么条件」。

  ```cpp
  struct CStaticClassProvider
  {
      template <typename T>
      auto Requires(UClass*& ClassRef) -> decltype(
          ClassRef = T::StaticClass()   // 约束：T 必须有 StaticClass()
      );
  };
  ```

  这是**模板元编程**层面的高级东西，写普通 Gameplay 逻辑基本碰不到，先知道有这回事、看到 `C` 前缀知道是「编译期概念约束」即可。

- **`b` 前缀 = 布尔变量**：UE 规定 `bool` 变量加 `b` 前缀，一眼就知道它是个真/假开关，例如 `bPendingDestruction`（等待销毁？）、`bHasFadedIn`（是否已淡入？）。

---

## 九、实战：创建并使用一个结构体

最简单的结构体，只需要在 C++ 结构体定义的基础上添加一些宏即可。

创建一个 C++ 类，类型选无，修改内容。

这里举例创建一个 MYStruct 的结构体。
文件名可以随意起，文件中也可以包含多个结构体。

**结构体必须以 F 打头，必须引入 文件名.generated.h**

```C++
#pragma once //防止头文件被重复 include，C++ 写法
#include "CoreMinimal.h"// UE 引擎的核心头文件，写 UE 代码基本都要引入
#include "MYStruct.generated.h"

USTRUCT()
struct FMYStruct
{
    GENERATED_USTRUCT_BODY()
    
public: 

};
```

文件名为 MYStruct.cpp。未来想要使用这个结构体，就在文件里写`#include "MYStruct.h"`就能直接用：
```c++
FMYStruct myData;
```

这样的结构体基本只能在C++中使用，如果想为蓝图提供接口，那么就需要为宏添加一些修饰符。

蓝图（Blueprint）是 UE 的可视化脚本系统。不用写 C++ 代码，通过拖拽节点、连线的方式就能写游戏逻辑。

