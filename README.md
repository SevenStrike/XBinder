# XBinder

**Unity 数据绑定管理插件 —— 数据与界面解耦，一次写入，多处响应**

XBinder 是一套面向 Unity 的轻量级数据绑定框架。它把分散在各个脚本与组件里的运行时数据收拢到一条**数据总线**上：由数据源（`XBinder_Source`）统一创建容器、拉取并解析数据并写入，由适配器（`XBinder_Adapter_*`）统一订阅，UI 控件从此不必关心数据从哪里来、多久刷新一次，只声明 "我要哪个 key"。数据变化时自动广播，写入相同值时直接短路，不产生多余的 UI 刷新；整条链路事件驱动，没有逐帧轮询。当数据源被销毁、或脚本重编译（Domain Reload）导致引用失效时，适配器通过总线事件自动重新绑定并回显当前值，不会留下断链。借助这套结构，本地 Json、HTTP 接口、数据库轮询都可以用同一套写法接进界面，更换数据来源无需改动 UI 层。

## 数据流



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

## 核心特性



* **12 种类型通道**：float、int、bool、string、Color、Vector2/3/4、Quaternion、Rect、RectOffset、Object，同名字段按类型分通道存储，互不干扰。

* **只在变化时广播**：写入相同值直接返回（float 带 0.0001 容差），避免无意义的重复刷新。

* **三种适配器**：`Adapter_Base<T>` 单向只读；`Adapter_Sync<T>` 双向回写，`_syncing` 闸门拦截自身回环，杜绝 "UI 改 → 数据变 → UI 再改" 的死循环；`Adapter_Multi` 用一个组件批量订阅多个 key，按类型事件分发。

* **断线自动重连**：数据源销毁即从总线摘除，重生时适配器自动重新绑定并推送初始值，热重载后无需手动重新挂钩。

* **请求节奏可控**：数据源支持手动按键触发或按间隔自动刷新，`request()` / `bind()` 交由子类实现，本地 Json 与 HTTP GET/POST 走同一接口。

* **配套编辑器工具**：定制 Inspector（折叠分组、容器键预览、多选批量操作）、总线监视窗口（搜索数据源、只读查看键值与订阅者数量）、Json 转 C# 实体生成器。

## 环境与示例



* Unity 6000.0.38f1（Unity 6），命名空间 `SevenStrikeModules.XBinder`，编辑器界面绘制依赖 XGUI。

* 内置两套示例：`Samples/Local`（本地轮询驱动，滑块调速控制风扇转速）、`Samples/Web`（网络驱动，Node.js + Express + MySQL 服务端，GET/POST 拉取数据并双向同步）。

* 授权协议：AGPL-3.0。