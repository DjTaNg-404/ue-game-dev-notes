# UE4 C++ 基础

> 本笔记是一份 UE C++ 入门基础沉淀，从「引擎总览」开篇，再按「类型与对象模型 → C++ 语法要点 → 命名规范 → 实战」的顺序组织，覆盖：引擎概念、宏与反射系统、UObject 对象模型、NewObject/CDO、基类指针与多态、运算符（双冒号 / 点 / 箭头）、类定义与 public 继承、命名规范与各类型前缀（F/U/A/S/T/I/C/E/b）。

**目录**
- 引擎总览（开篇）：引擎是什么、它和 C++ 的关系
- 一、类型与对象模型：1.1 宏与反射系统 / 1.2 UObject 与对象模型 / 1.3 创建 UObject 实例 / 1.4 构造器要轻与 CDO / 1.5 基类指针与多态
- 二、C++ 语法要点：2.1 运算符 / 2.2 类定义与 public 继承
- 三、命名规范：3.1 PascalCase / 3.2 类型前缀总览 / 3.3 前缀：人看还是编译 / 3.4 模板类 T / 3.5 接口 I / 3.6 枚举 E / 3.7 概念类 C 与布尔 b
- 四、实战：创建并使用一个结构体

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

## 一、类型与对象模型

### 1.1 宏与反射系统（Macro & Reflection）

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

### 1.2 UObject 与对象模型

之前我们讲了 `USTRUCT()` 是给结构体办的「登记证」，让 UHT 生成反射代码。UObject 是另一回事：它是 UE 自己提供的一个 **C++ 基类**，是所有「引擎托管对象」的共同祖先。

这里要把两件事分清，别混在一起：

- **宏 = 登记证**：`UCLASS()` / `USTRUCT()` 告诉 UE 「这个类/结构体需要被反射系统管理」。
- **UObject = 被托管的基类**：`UCLASS class AMyThing : public UObject` 这种写法里，你用 `UCLASS()` 登记，同时**必须继承 UObject**。继承了它，你的类才「进了引擎系统」——引擎替你管生死（垃圾回收 GC），并提供序列化、网络同步、编辑器显示等一整套服务。

而 `USTRUCT struct FFoo` 登记的是**值类型**，它**不继承 UObject**，像 `std::pair` 那样轻量、按值拷贝、没有 GC；但它一旦作为 `UPROPERTY` 嵌在某个 UObject 里，就会被一起序列化和 GC 跟踪。

### 1.3 创建 UObject 实例：NewObject 与 CreateDefaultSubobject

`NewObject<T>()` 创建的是一个**活的、被引擎接管的 UObject 实例**：引擎知道它存在、能垃圾回收它、能把它序列化存盘、能在编辑器里显示它的属性、能网络同步它。

`CreateDefaultSubobject<T>()` 只能在**构造器里**调用，创建的是「作为某个 Actor 组成部分的默认子对象」（比如一个 SceneComponent、一个碰撞体）。它被写进 Actor 的模板里、在编辑器里可见，每个该 Actor 的实例都共享这套部件。

### 1.4 构造器要轻，以及 CDO 机制

引擎启动时，会给每个 `UCLASS` 用默认构造器造一个「**类默认对象**」(CDO，Class Default Object) 作为所有运行时实例的原型。这件事带来三条铁律：

1. **必须有无参默认构造器**，否则编译不过——因为引擎要靠它先造 CDO 模板。
2. **构造器只能设默认值、挂子对象**，不能做真正的逻辑（比如读文件、调别的系统、算业务）。否则每个实例都会重跑、而且编辑器和运行实例共享同一个模板，会出乱子。
3. **真正的初始化放到 `BeginPlay()`**（Actor）或 `PostInitProperties()`（其他 UObject）。

**防坑**：不要手动 `delete` UObject，它交给 GC 管，你只负责 `NewObject<>()` 和持有 `UPROPERTY()` 引用；容器里装基类指针要用 `TArray<UObject*>` + `UPROPERTY`，别用 `std::vector`（GC 看不见，会把对象回收掉还留野指针）。

### 1.5 基类指针与多态：先定义「Object」不指定具体类型

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

## 二、C++ 语法要点（UE 语境）

### 2.1 运算符：双冒号、点、箭头

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

### 2.2 类定义里的冒号与 public 继承

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

## 三、命名规范（Naming Convention）

### 3.1 PascalCase 与基本约定

UE 有一套强制命名约定：所有代码和注释用美式英语；命名（类型名、变量名）用 PascalCase——每个单词首字母大写、单词间无下划线，例如 `Health`、`UPrimitiveComponent`。`lastMouseCoordinates` 或 `delta_coordinates` 这种带下划线或驼峰的写法都是错的。

### 3.2 类型前缀总览

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

### 3.3 前缀：为人看还是为编译？

主要是**给人看**：一眼知道类别、避免重名。但对**带反射标记的类**（`UCLASS()` / `USTRUCT()`），UHT（Unreal Header Tool）会在编译期**强制**检查前缀是否和基类匹配，不匹配直接报错、编译不过。比如：

```cpp
UCLASS()
class MyActor : public AActor { ... };   // ❌ UHT 报错：继承 AActor 必须叫 AMyActor
```

反过来，不加 `UCLASS` 的纯 C++ 辅助类，前缀纯是约定，编译器不在乎。所以：**前缀本质是给人设计的约定；UE 通过 UHT 把这条约定对「反射类」升级成了编译红线。**

### 3.4 模板类前缀 T：带「类型插槽」的类

`T` 前缀标记的是模板类。模板类就是你打比赛天天用的东西——`std::vector`、`std::pair`、`std::map` 全都是模板类。它自己不指定装什么类型，而是留一个占位符（UE 里常叫 `ObjectType`，STL 里叫 `T`），等你用时再把具体类型填进去：

```cpp
template <typename ObjectType>   // ObjectType 是占位符，不是具体类型
class TAttribute { /* ... */ };

TAttribute<float>   Health;      // 填 float   → 一个装 float 的属性
TAttribute<FString> Name;        // 填 FString → 一个装字符串的属性
```

填进去的过程叫**模板实例化**，发生在编译期——编译器替你基于 `TArray<float>` 生成一份真正的「装 float 的动态数组」代码。你比赛里写的 `std::vector<int>`，就是给 `std::vector` 这个模板类填了 `int`。

**关键**：T 前缀和 U/A/S 不是一回事。U/A/S 看「继承谁」，T 看「怎么定义的」（是不是泛型）。`TArray`、`TMap`、`TSharedPtr` 这些不是 UObject 的子类，只是泛型工具类，所以打 T。实践里 UObject 类几乎不可能是模板，所以你不会看到 `TUObject`。

### 3.5 抽象界面类 / 接口前缀 I

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

### 3.6 枚举前缀 E

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

### 3.7 概念类前缀 C 与布尔前缀 b

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

## 四、实战：创建并使用一个结构体

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
