## XBinder Unity 数据绑定管理插件 (源于 Unity 6000.0.38f1 开发)

[![license](https://img.shields.io/badge/license-AGPLv3.0-white.svg)](https://github.com/SevenStrike/XBinder/blob/main/LICENSE)&nbsp;&nbsp;
[![supported](https://img.shields.io/badge/Supported-Unity-success.svg)](https://unity.com/)

> 一次写入，多处响应：数据进总线，界面只订阅。
> 单向只读、双向回写、多路批量三种适配器，12 种类型通道，断线自动重连。

XBinder 是一套面向 Unity 的轻量级数据绑定框架。它把分散在各个脚本与组件里的运行时数据收拢到一条**数据总线**上：由数据源（`XBinder_Source`）统一创建容器、拉取并解析数据并写入，由适配器（`XBinder_Adapter_*`）统一订阅，UI 控件从此不必关心数据从哪里来、多久刷新一次，只声明“我要哪个 key”。数据变化时自动广播，写入相同值时直接短路，不产生多余的 UI 刷新；整条链路事件驱动，没有逐帧轮询。当数据源被销毁、或脚本重编译（Domain Reload）导致引用失效时，适配器通过总线事件自动重新绑定并回显当前值，不会留下断链。借助这套结构，本地 Json、HTTP 接口、数据库轮询都可以用同一套写法接进界面，更换数据来源无需改动 UI 层。

<br>

| 开源不易，您的支持是持续更新的动力，<br>这个小工具倾注了我无数个深夜的调试与优化，它永远免费，但绝非无成本，如果您觉得这个工具<br>能为您节省时间、解决问题，甚至带来一丝愉悦，请考虑赞助一杯咖啡，让我知道：有人在乎这份付出，而这将成为我熬夜修复Bug、<br>添加新功能的最大动力。开源不是用爱发电，您的认可会让它走得更远|![](Docs/donate.jpg) |
|:-|-:|
| **欢迎加入技术研讨群，在这里可以和我以及大家一起探讨插件的优化以及相关的技术实现思路，同时在做项目时遇到的众多问题以及瓶颈<br>阻碍都可以互相探讨学习**|![](Docs/qqgroups.jpg) |

- **命名空间**：`SevenStrikeModules.XBinder`
- **程序集**：`XBinder.Runtime`（运行时，引用 `XGUI.Runtime`）、`XBinder.Editor`（仅 Editor）
- **前置依赖**：Inspector 面板与工具窗口由 XGUI 绘制，必须先安装 `com.sevenstrike.xgui`，否则会因缺少该命名空间而编译失败

<br>

### 📦 它解决什么问题
---

| 没有绑定层的常见写法 | 用 XBinder 之后 |
| :--- | :--- |
| 每个 UI 脚本各自持有一份数据，改字段要同步改多处 | 数据只存在 `Category` 里一份，所有 UI 订阅同一个 key |
| 在 `Update` 里每帧刷新文本、滑块、颜色 | 值变化才回调；值不变则整条链路无开销 |
| 数据源销毁或脚本重编译后引用变空，UI 卡住 | 总线广播创建 / 销毁事件，适配器自动重绑并回显 |
| 换数据来源（本地 Json → HTTP）要连 UI 代码一起改 | 只改 `XBinder_Source` 子类的 `request()`，UI 层不动 |

<br>

### 📦 安装
---

⚠️ **必须先装 XGUI**：XBinder 的 Inspector 与工具窗口完全由 XGUI 绘制，`XBinder.Runtime` 的程序集引用也指向 `XGUI.Runtime`。顺序反过来会先报缺命名空间的编译错误。

### ▶️ 前置：安装 XGUI

**Git 地址**

```text
https://github.com/SevenStrike/XGUI.git?path=com.sevenstrike.xgui
```

在 `Window > Package Manager` 中点左上角 `+` → `Add package from git URL...` 填入即可；也可以把这行写进 `Packages/manifest.json` 的 `dependencies`。

### ▶️ 方式 A：从本地磁盘添加

1. 打开 `Window > Package Manager`
2. 点击左上角 `+`，选择 `Add package from disk...`
3. 选择 `com.sevenstrike.xbinder/package.json`

### ▶️ 方式 B：写入 `manifest.json`

```json
{
  "dependencies": {
    "com.sevenstrike.xbinder": "file:../com.sevenstrike.xbinder"
  }
}
```

### ▶️ 方式 C：通过 Git 安装（推荐）

仓库地址：<https://github.com/SevenStrike/XBinder>。在 `Add package from git URL...` 中填入：

```text
https://github.com/SevenStrike/XBinder.git?path=com.sevenstrike.xbinder
```

需要锁版本时在末尾追加 `#tag`，例如 `#v1.0.0`。

### ▶️ 方式 D：网络受限 / 无 Git 环境

在浏览器打开仓库页面，`Code` → `Download ZIP`，解压后用 `Add package from disk...` 选择解压目录中的 `com.sevenstrike.xbinder/package.json`。这条路径完全不经过 Unity 内的 git，代价是不会随仓库自动升级。

<br>

### 📦 数据流
---

```
XBinder_Source（数据源 / MonoBehaviour）
      │  创建容器、拉取并解析数据、写入 key
      ▼
XBinder_Bus（全局数据总线）
      │  按 name 注册 / 注销容器，广播 created / destroyed
      ▼
XBinder_Category（数据容器）
      │  12 条类型通道：key-value 存储 + 订阅者列表，值变才回调
      ▼
XBinder_Adapter_*（适配器 / MonoBehaviour）
      │  单向只读 / 双向回写 / 多路批量
      ▼
UI 控件（Text、Image、Slider、Toggle …）
```

| 角色 | 类型 | 职责 |
| :--- | :--- | :--- |
| 数据总线 | `XBinder_Bus`（静态类） | 登记全部容器；`on_category_created` / `on_category_destroyed` 广播，是重连机制的根 |
| 数据容器 | `XBinder_Category` | 12 条类型通道，`key → value` 存储 + 每 key 一份订阅者列表 |
| 数据源 | `XBinder_Source`（MonoBehaviour） | `Awake` 建容器；按手动按键或定时间隔调 `request()`；解析后 `bind()` 写入 |
| 适配器 | `XBinder_Adapter_Base<T>` / `_Sync<T>` / `_Multi` | 订阅、回写、多键批量；负责初始值回显与断线重连 |

<br>

### 📦 特性一览
---

✅ **数据总线与容器**

- 全局静态总线 `XBinder_Bus.categories`，容器以 `xbinder_` 前缀登记，查询时自动补前缀，脚本里只写裸名字
- **12 种类型通道**：`Float`、`Int`、`Bool`、`String`、`Color`、`Vector2`、`Vector3`、`Vector4`、`Quaternion`、`Rect`、`RectOffset`、`Object`，同名字段分通道存储、互不干扰
- 写入即比较：`float` 带 `0.0001` 容差，其余按精确相等判断，值未变化直接返回，不产生任何回调
- 订阅者按 key 分类型维护，遍历时逆序索引，允许在回调里安全退订
- Editor 只读视图：`get_all_keys` / `get_key_type` / `get_subscriber_count` / `get_value_as_object`，全部包在 `#if UNITY_EDITOR` 内，不进构建

✅ **三种适配器**

- `XBinder_Adapter_Base<T>` **单向只读**：绑定 → 订阅 → 初始值回显 → 断线重绑，四个抽象方法由子类实现
- `XBinder_Adapter_Sync<T>` **双向回写**：`push(T)` 写回容器，`_syncing` 闸门拦截自身回环，杜绝“UI 改 → 数据变 → UI 再改”
- `XBinder_Adapter_Multi` **多路批量**：一个组件订阅多个 key，`entries` 配置 + 按类型事件（`on_float` … `on_rectoffset`，共 12 个）分发；构建期自动校验空键、空容器与重复订阅并跳过
- 12 个叶子类型实现 `XBinder_Adapter_Sync_Float` / `_Int` / `_Bool` / `_String` / `_Color` / `_Vector2` / `_Vector3` / `_Vector4` / `_Quaternion` / `_Rect` / `_RectOffset` / `_Object`，挂到 Slider、Toggle、InputField 上即可用
- 初始值推送：绑定成功立即回显；若绑定时数据源还没建好，`Update` 会兜底补推一次

✅ **生命周期与重连**

- `Awake` 建容器、`OnDestroy` 摘除容器，容器不随 GameObject 销毁而残留
- Domain Reload 后容器引用失效会自动重建；编辑器侧提供 `Editor_Recomplier_Rebuild` 手动重连
- 总线事件驱动重连：数据源重生 → 适配器重新绑定 → 立刻推送当前值，中途无空白帧
- 重连与断线都有事件出口（`act_handle_category_created` / `act_handle_category_destroyed`），可在 UI 上做变灰、断线提示

✅ **数据源**

- 两种刷新节奏：手动按键触发（默认 `Space`）或按 `request_interval` 定时刷新（默认 1 秒）
- `resolve<T>(json)` 内置 `JsonUtility` 解析，空串、格式错误、结果为空都会打日志并返回 `default`
- `request()` / `bind(json)` 均为虚方法，本地文件、HTTP GET/POST、数据库轮询都走同一接口
- 示例已覆盖：`TextAsset` 本地 Json、绝对路径读盘、HTTP POST、以及把“字符串化 data 字段”解包成真 Json 的中间层

✅ **编辑器工具**

- 定制 Inspector（XGUI 绘制）：`快捷操作`（加载 / 卸载数据源、手动请求、重建）、`参数`、`数据源概览`、`选项`、`扩展面板`、`源脚本`
- `数据源概览` 同时显示“绑定的数据项”与“实际数据量”，两值不等即说明有同名键被写入了不同类型，订阅方会读不到更新
- 绑定总线查看器：左栏数据源列表（搜索、仅看有订阅者的键），右栏虚拟滚动键表（键名 / 类型 / 当前值 / 订阅数）
- Json 转 C# 实体生成器：选 `TextAsset`、设根类名与命名空间，一键生成可直接被 `resolve<T>()` 使用并带 `[Serializable]` 的类
- Inspector 支持多选批量操作（`[CanEditMultipleObjects]`），一套参数可同时写入多个数据源

✅ **工程化**

- 资源路径通过 `PackageInfo.FindForAssembly` 自动解析，UPM 包与旧式 `Assets/` 目录两种安装形态都能正确定位 GUI 资源
- 程序集拆分 `XBinder.Runtime` / `XBinder.Editor`，运行时部分不引用 `UnityEditor`，可直接进构建
- 数据通道用 `Dictionary` + 订阅者 `List`，热路径无反射、无 LINQ、无装箱；Editor 侧的装箱读值只在面板里发生

✅ **环境与兼容**

|🌱 项目|🌱 说明|
|:---:|:---:|
|**Unity 版本**|`6000.0.38f1` 及以上（`package.json` 已声明 `6000.0` / `38f1`）|
|**包管理器**|UPM，包名 `com.sevenstrike.xbinder`|
|**依赖**|XGUI（`com.sevenstrike.xgui`），需先安装|
|**程序集**|`XBinder.Runtime`（引用 `XGUI.Runtime`）、`XBinder.Editor`|
|**运行平台**|全部；编辑器面板与工具窗口仅 Editor|
|**授权**|GNU AGPL-3.0|

<br>

### 📦 快速开始
---

### 🧊 第一步：写一个数据源

继承 `XBinder_Source`，给它一个容器名，在 `bind()` 里把数据写进通道即可。`request()` 决定“什么时候去拿数据”，本地造数据、读文件、发请求都放在这里。

```csharp
using SevenStrikeModules.XBinder;
using UnityEngine;

public class player_source : XBinder_Source
{
    public override void Awake()
    {
        identifier = "player";      // 运行时登记为 xbinder_player
        base.Awake();
    }

    public override void request()
    {
        bind("");                   // 真实项目里换成读取 Json / HTTP 返回
    }

    public override void bind(string json = "")
    {
        // resolve<T>(json) 可把 Json 直接转成实体，这里用固定值演示
        category.set_string("name", "Engine-A");
        category.set_float("speed", 3200f);
        category.set_color("tint", new Color(0.23f, 1f, 0.61f));
    }
}
```

- ✅ 键名是 UI 侧订阅的唯一标识，`set_*` 与订阅方的 `target_key` 必须逐字一致（区分大小写）。

### 🧊 第二步：挂一个双向适配器，接上 Slider

在需要绑定的物体上添加 `XBinder_Adapter_Sync_Float`，填 `target_category` 与 `target_key`（**不带 `xbinder_` 前缀**），然后在脚本里接控件：

```csharp
using SevenStrikeModules.XBinder;
using UnityEngine;
using UnityEngine.UI;

public class speed_slider : MonoBehaviour
{
    public XBinder_Adapter_Sync_Float adapter;
    public Slider slider;

    private void OnEnable()
    {
        adapter.act_on_value_changed += on_data_changed;   // 数据 → UI
        slider.onValueChanged.AddListener(on_slider_changed); // UI → 数据
    }

    private void OnDisable()
    {
        adapter.act_on_value_changed -= on_data_changed;
        slider.onValueChanged.RemoveListener(on_slider_changed);
    }

    private void on_data_changed(float value) => slider.SetValueWithoutNotify(value);

    private void on_slider_changed(float value) => adapter.push(value);
}
```

- ✅ 回写一律走 `push()`：它内部会开 `_syncing` 闸门，直接调 `set_value` 会绕开闸门并触发自身回环。

### 🧊 第三步（可选）：单向只读控件

只显示、不修改的控件（文本、图片、进度条）继承 `XBinder_Adapter_Base<T>`，四个抽象方法实现完即可，重连与初始值回显由基类负责：

```csharp
using SevenStrikeModules.XBinder;
using UnityEngine;
using UnityEngine.UI;

public class name_text : XBinder_Adapter_Base<string>
{
    public Text label;

    protected override void subscribe() => category.subscribe_string(target_key, on_value_changed);
    protected override void unsubscribe() => category.unsubscribe_string(target_key, on_value_changed);
    public override string get_value() => category.get_string(target_key);
    protected override void on_value_changed(string value) => label.text = value;
}
```

<br>

### 📦 API 速记
---

**总线 `XBinder_Bus`（静态）**

|API|用途|
|:---|:---|
|📌 `create_category(name)` / `destroy_category(category)`|创建 / 销毁容器，并广播对应事件|
|📌 `register(category)` / `unregister(category)`|直接登记 / 注销已有容器|
|📌 `get_category_by_name(name)`|按裸名字取容器（内部补 `xbinder_` 前缀）|
|📌 `get_category_index(name)` / `categories`|取序号 / 直接遍历全部容器|
|📌 `get_category_names()`|把当前所有容器名打印到 XGUI 控制台|
|📌 `on_category_created` / `on_category_destroyed`|`Action<string>`，参数为带前缀的完整名|

**数据源 `XBinder_Source`**

|成员|类型|说明|
|:---|:---|:---|
|`identifier`|`string`|容器名（裸名），留空会创建空名容器|
|`category`|`XBinder_Category`|当前容器，`Awake` 中创建|
|`manual_request` / `request_key`|`bool` / `KeyCode`|手动模式与触发键（默认 `Space`）|
|`request_interval`|`float`|自动模式的刷新间隔（默认 1 秒）|
|`request()`|`virtual`|数据获取入口，子类重写|
|`bind(json)`|`virtual`|把数据写入通道，子类重写|
|`resolve<T>(json)`|`T`|`JsonUtility` 反序列化 + 错误日志|
|`category_create()` / `category_remove()`|—|手动装载 / 卸载容器（Inspector 快捷操作调用）|
|`Editor_Recomplier_Rebuild()`|—|仅运行时可用，重编译后重连容器并补一次请求|

**容器 `XBinder_Category`：12 组同构方法**

每种类型都提供四个方法，命名后缀相同：`set_` / `get_` / `subscribe_` / `unsubscribe_`。

|后缀|类型|`get_` 默认值|
|:---|:---|:---|
|`float`|`float`|`0f`|
|`int`|`int`|`0`|
|`bool`|`bool`|`false`|
|`string`|`string`|`null`|
|`color`|`Color`|`default`|
|`vector2` / `vector3` / `vector4`|`Vector2` / `Vector3` / `Vector4`|`default`|
|`quaternion`|`Quaternion`|`default`|
|`rect`|`Rect`|`default`|
|`rectoffset`|`RectOffset`|`default`（`null`）|
|`object`|`object`|`null`|

**三种适配器对比**

|类型|方向|典型用途|关键 API|
|:---|:---:|:---|:---|
|`XBinder_Adapter_Base<T>`|数据 → UI|文本、图片、进度条等只读展示|`get_value()` / `on_value_changed(T)` / `subscribe()` / `unsubscribe()`|
|`XBinder_Adapter_Sync<T>`|双向|Slider、Toggle、InputField|`push(T)` / `set_value(T)` / `act_on_value_changed`|
|`XBinder_Adapter_Multi`|数据 → 业务层|一个组件吃多个 key、按类型分发|`entries` / `global_category` / `on_float` … `on_rectoffset`|

**多路适配器的绑定项 `XBinder_Adapter_BindEntry`**

|字段|说明|
|:---|:---|
|`target_key`|订阅的键名|
|`target_category`|容器名，留空则使用 `global_category`|
|`value_type`|`XBinder_ValueType` 枚举，决定走哪条通道与哪个事件|

<br>

### 📦 编辑器工具入口
---

| 入口 | 位置 | 说明 |
| :--- | :--- | :--- |
| 数据源 Inspector | 选中挂有 `XBinder_Source` 子类组件的物体 | 快捷操作、参数、容器概览与多选批量编辑 |
| 绑定总线查看器 | `Tools > XBinder > 绑定总线查看器` | 左右双栏、搜索、仅看有订阅者的键、虚拟滚动键表 |
| Json 转 C# 实体 | `Tools > XBinder > Json 解析到 C# 实体类工具` | 选 `TextAsset` 生成 `[Serializable]` 实体类 |
| 网络数据源模拟服务 | `Tools > XBinder > 网络数据源模拟服务` | 随 Web 示例提供，一键拉起本地 Node 服务 |

- ✅ 总线查看器会把选中的容器记在 `SessionState`，脚本重编译后自动恢复；左右栏宽度写入 `EditorPrefs`。

<br>

### 📦 示例
---

示例以 UPM Sample 形式随包分发，首次使用在 Package Manager 的 XBinder **Samples** 区域点 **Import**，导入后位于 `Assets/Samples/XBinder/1.0.0/Demo/`。

| 示例 | 场景文件 | 演示内容 |
| :--- | :--- | :--- |
| **Local** | `Samples~/Local/Scene/Demo_Local.unity` | 从 `TextAsset` 或绝对路径读 Json 写入容器；滑块双向控制风扇转速；覆盖全部 12 种通道的写入与回显 |
| **Web** | `Samples~/Web/Scene/Demo_WebRequestData.unity` | Node.js + Express + MySQL 服务端（默认 `http://localhost:9090/data`）；POST 拉取、`data` 字段字符串化解包、多路批量绑定、按状态切换序列帧 |

**运行 Web 示例**

1. 进入 `Samples~/Web/Servers/NodeJS/Requester`，执行 `npm install` 补齐依赖
2. 按你的数据库配置改连接串，启动服务端（或在编辑器里用 `Tools > XBinder > 网络数据源模拟服务`）
3. 打开 `Demo_WebRequestData` 场景进入 Play 模式，数据源按 `request_interval` 定时拉取

- ✅ 示例中的 `request()` 只做“发请求”，解析与写通道交给 `bind()`，两者职责分离是这个框架的推荐写法。

<br>

### 📦 目录结构
---

```
com.sevenstrike.xbinder/
├─ GUI/                     XGUI 绘制用的面板图标（binder / adapter / bus_viewer / json_to_entity）
├─ Scripts/
│  ├─ Runtime/              XBinder_Runtime 程序集
│  │  ├─ Core/              XBinder_Bus、XBinder_Category、XBinder_Source
│  │  ├─ Adapter/           Adapter_Base / Adapter_Sync / Adapter_Multi + 12 个 Sync 叶子类型
│  │  └─ Utilitys/          XBinder_Dashboard（路径与主题色）、XBinder_Enums（通道枚举）
│  └─ Editor/               XBinder_Editor 程序集
│     ├─ Core/              XBinder_Source 的定制 Inspector
│     ├─ Adapter/           Adapter_Multi 与 12 个 Sync 类型的定制 Inspector
│     └─ Utilitys/          总线查看器、Json 转实体生成器
├─ Samples~/
│  ├─ Local/                本地数据源示例（场景、脚本、Json、精灵）
│  └─ Web/                  网络数据源示例（场景、脚本、Node 服务端）
├─ package.json             UPM 描述
├─ CHANGELOG.md
└─ LICENSE.md               AGPL-3.0 全文
```

<br>

### 📦 注意事项
---

1. ⚡ **先装 XGUI**：`XBinder.Runtime` 的程序集引用指向 `XGUI.Runtime`，缺少时整个包无法编译。
2. ⚡ **容器在 `Awake` 创建**：未进入 Play 模式时，Inspector 的“数据源概览”只会显示提示文字，这是预期行为。
3. ⚡ **同名键不要跨类型写入**：同一个 key 同时写进 `float` 和 `string` 通道后，订阅 `float` 的一方永远读不到更新。Inspector 里“绑定的数据项”与“实际数据量”两个数字不相等，就是这种情况的信号。
4. ⚡ **`target_category` 不带前缀**：填的是裸名字（如 `player`），总线上登记的才是 `xbinder_player`。
5. ⚡ **回写必须走 `push()`**：`Sync` 适配器直接调 `set_value` 会绕开 `_syncing` 闸门，造成 UI 与数据互相触发。
6. ⚡ **手动模式依赖旧版输入**：`request_key` 判断走 `Input.GetKeyDown`，项目若只启用新 Input System，需要在 Player Settings 里保留旧版输入或自行改写。
7. ⚡ **数据源销毁后 UI 会短暂留白**：此时适配器处于解绑等待状态，重生后自动回填；不要在 `act_handle_category_destroyed` 里清空业务数据。
8. ⚡ **版本信息同步维护**：`package.json` 的 `version` 与 `CHANGELOG.md` 保持一致，Package Manager 展示的是 `package.json` 的 `description`。

<br>

### 📦 授权
---

本项目遵循 **GNU Affero General Public License v3.0（AGPL-3.0）**。

- 可自由使用、修改、分发源代码，但必须保留文件头部的版权注释
- 基于本项目的衍生作品需同样以 AGPL-3.0 发布
- 若以网络服务形式提供（如云端编辑器、在线数据工具），需公开修改后的完整源代码

如需闭源商用或定制授权，请先与作者联系确认授权方式。完整协议文本：<https://www.gnu.org/licenses/agpl-3.0.html>

<br>

### 📦 关于
---

|||
|:---:|:---:|
|作者|徐寅智（Isaac.Xu）|
|团队|南京塞维斯传媒有限公司（SevenStrikeMedia）|
|官网|<http://sevenstrike.com/>|
|仓库|<https://github.com/SevenStrike/XBinder>|
|相关插件|XTween、XHUD、XGUI、XBinder、XGraph|
|版权|Copyright (C) 2025-Present Nanjing SevenStrike Media Co., Ltd.|
