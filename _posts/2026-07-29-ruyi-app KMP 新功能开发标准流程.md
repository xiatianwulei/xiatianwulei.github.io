---
layout:     post
title:      ruyi-app KMP 新功能开发标准流程（转）
subtitle:   
date:       2026-07-29
author:     夏天无泪
catalog: true
tags:
    - 学习记录
---

---
name: ruyi-kmp-feature-dev
description: 在 ruyi-app KMP 项目现有架构下从零开发一个功能的标准流程指南。当用户提到「开发新功能」「新增页面」「新建模块」「添加 KMP 功能」或在 ruyi-app KMP 项目中从零实现一个业务需求时触发。覆盖 domain、data、platform、presentation 四层，以及 Koin DI 注册、Android 入口和编译验证。
---

# ruyi-app KMP 新功能开发标准流程

## 1. 开发前必须确认的五件事

在创建任何文件之前，必须按顺序确认以下五项：

### 1.0 项目目录路径

在编写代码之前，**必须先与用户确认项目的完整目录路径**：

1. 根据当前工作区推断或用户提供的信息，输出计划使用的项目目录路径。
2. 向用户提供明确的 **"正确"** 和 **"错误"** 两个选项，提示用户验证。
3. 若用户确认 **"正确"**，继续执行后续步骤。
4. 若用户选择 **"错误"**，暂停所有操作，要求用户手动输入或指定正确的项目目录路径，并在获取并确认新目录后，方可开始编写代码。

### 1.1 功能名称

**功能名称是什么？**（例如：`systemNotification`、`profile`、`wallet`）

- 用于命名目录、类、接口、UseCase、ViewModel、Screen 等。

### 1.2 表现层模块

**表现层代码放在哪个 app-main 模块？**（例如：`app-main/mine`、`app-main/home`、`app-main/main`）

- 决定 `ViewModel`、`Screen`、`Activity` 以及 `Koin.kt` 中的注册位置。

### 1.3 列表页交互（仅限列表页场景）

当检测到用户需要实现的是**列表页**（即页面主体为一个可滚动的数据列表）时，必须主动询问用户：

**"该列表页是否需要集成下拉刷新和上拉加载更多功能？"**

向用户提供 **"需要"** 和 **"不需要"** 两个选项：

- 若用户选择 **"需要"**：在代码实现中完整包含下拉刷新、上拉加载更多的交互逻辑，以及相关的加载状态提示（如：加载中、加载完毕、无更多数据）。
- 若用户选择 **"不需要"**：仅实现基础的列表数据渲染，不添加任何下拉或上拉触发的额外事件处理逻辑。

> 对于非列表页（如表单页、详情页等），无需询问此项。

### 1.4 缺省视图

必须主动询问用户：

**"该页面是否需要显示缺省视图（空数据视图、加载失败视图等状态页面）？"**

向用户提供 **"需要"** 和 **"不需要"** 两个选项：

- 若用户选择 **"需要"**：缺省视图的 UI 设计与状态处理逻辑必须参考 `MessageListScreen.kt` 和 `MessageListViewModel.kt` 的实现方式：
  - **Screen 层**：使用 `DefaultScreenUI` 包裹整个页面，传递 `errors` 和 `progressBarState`；在内容为空且 loading 已结束时，通过 `PageStateContainer` 展示空数据 / 加载失败等状态。
  - **ViewModel 层**：在数据为空时设置 `error = UIComponent.Empty()`；在加载失败且无已有数据时通过 `errorMapper.map(error)` 生成全屏错误组件存入 state。

- 若用户选择 **"不需要"**：仅实现基础的数据渲染，不包含空数据、加载失败等缺省状态视图。

> 不要在未确认以上五项信息前直接生成代码。

## 2. 架构分层与依赖方向

项目采用 Clean Architecture + MVI，依赖方向严格单向：

```
app-main/<feature>  →  app-common/domain
app-common/ui       →  app-common/domain
app-common/data     →  app-common/domain
app-common/data     →  kmp-common/common / kmp-core
```

| 模块 | 职责 | 典型内容 |
|------|------|----------|
| `kmp-core/core` | 基础设施 / MVI 基类 | `BaseViewModel`、`BaseUseCase`、`Result`、`Response`、`UIComponent` 等 |
| `kmp-core/base` | 平台上下文 | `PlatformContext`、`getPlatformContext()` |
| `kmp-common/common` | 平台能力（expect/actual） | 通知状态、拨号、图片选择、跳转链接等 |
| `app-common/domain` | 领域层 | 领域模型、Repository 接口、UseCase |
| `app-common/data` | 数据层 | API 接口/实现、DTO、Repository 实现 |
| `app-common/ui` | 应用级通用 UI 层 | 跨业务复用的原子组件：按钮、分割线、卡片、列表 Item、弹窗等 |
| `app-common/datasource` | 旧网络/数据源层 | 现有 Service、Response DTO（新功能尽量走 data/domain） |
| `app-common/interactor` | 旧业务用例层 | 现有 Interactor（新功能尽量走 domain UseCase） |
| `app-main/<feature>` | 表现层 | Compose Screen、ViewModel、Android Activity，仅负责页面组装与业务逻辑 |
| `app-main/main` | DI 汇总 | `appModule`、`dataModule` 注册入口 |

## 3. 标准开发顺序

按照以下顺序实现，避免循环依赖和返工：

1. **领域层（domain）**：定义数据模型、Repository 接口、UseCase。
2. **数据层（data）**：定义 API、DTO、Repository 实现。
3. **平台能力层（kmp-common/common）**：如需访问系统能力，定义 `expect/actual object`。
4. **通用 UI 层（app-common/ui）**：若列表项、按钮、卡片等在多个业务页面复用，优先下沉为 `app-common/ui` 的原子组件。
5. **表现层（app-main/<feature>）**：编写 ViewModel、Screen、Android Activity，只做页面组装与业务逻辑。
6. **入口与 DI**：注册 Activity、Koin 依赖、模块跳转方法。
7. **编译验证**：先跑 `compileKotlinMetadata`，再跑 `compileDebugKotlinAndroid`。

## 4. 各层文件与命名约定

以功能名 `systemNotification` 在 `app-main/mine` 模块为例：

### 4.1 domain 层

目录：`app-common/domain/src/commonMain/kotlin/com/app/common/domain/<feature>/`

| 文件 | 作用 |
|------|------|
| `<Feature>State.kt` | 领域模型（纯数据类），`@Stable`。 |
| `repository/<Feature>Repository.kt` | 仓库接口，返回 `com.kmp.core.Result<T>`。 |
| `usecase/Get<Feature>UseCase.kt` | 获取数据 UseCase，继承 `BaseUseCase<Params, ResultType>`。 |
| `usecase/Save<Feature>UseCase.kt` | 写数据 UseCase。 |
| `usecase/<Feature>Param.kt` | UseCase 参数数据类。 |

### 4.2 data 层

目录：`app-common/data/src/commonMain/kotlin/com/app/common/data/<feature>/`

| 文件 | 作用 |
|------|------|
| `api/<Feature>Api.kt` | 接口 + 内部实现 `<Feature>ApiImpl`。 |
| `dto/<Feature>Dto.kt` | 请求/响应 DTO，`@Serializable`，含 `toDomain()` 扩展。 |
| `repository/<Feature>RepositoryImpl.kt` | 仓库接口实现，使用 `safeApiCall` 与 `converterResult`。写操作需检查业务码 `bizCode == it.businessSuccess`。 |

### 4.3 platform 能力层（按需）

目录：`kmp-common/common/src/<sourceSet>/kotlin/com/kmp/common/<feature>/`

- 优先使用 `expect/actual object`。
- Android 实现需要 `Context` 时，通过 `com.kmp.core.base.getPlatformContext()` 获取。
- `expect/actual object` 由编译器自动解析，**不需要在 Koin 中注册**。

### 4.4 表现层

目录：`app-main/<module>/src/commonMain/kotlin/com/app/main/<module>/.../<feature>/`

| 文件 | 作用 |
|------|------|
| `viewmodel/<Feature>ViewModel.kt` | MVI 核心：定义 `State`、`Event`、`Action`，继承 `BaseViewModel`。 |
| `<Feature>Screen.kt` | Compose 页面，注入 ViewModel，收集状态、派发事件。 |
| `androidMain/.../<feature>/<Feature>Activity.kt` | Android 入口 Activity。 |

**表现层职责边界**：业务页面（`<Feature>Screen.kt`）只负责页面组装与业务逻辑，不要直接实现可复用的底层 UI 组件。按钮、分割线、卡片、列表项等应下沉到 `app-common/ui` 层，沉淀为高内聚、可预览、可全局复用的 UI 原子。

**组件库优先原则**：开发表现层时，必须先到组件库 `kmp-ui/ui/src/commonMain/kotlin/com/kmp/ui/components/` 中查找是否已有现成组件。若已有，直接复用；若没有，再自行实现。若同一组件在多个业务模块中使用，必须将该组件下沉到组件库中统一维护。

> 组件库现有组件速览：`Banner`（轮播）、`Buttons`（圆形/图标/加载/默认按钮）、`CommonDialog` / `MultiButtonDialog` / `BottomSheet`（弹窗）、`Spacer_*dp`（间距）、`Tabs`（Tab 切换）、`PullRefreshLayout` / `LoadMoreView`（下拉刷新/加载更多）、`showToast`（轻提示）、`MultiSwitch`（开关）、`MultiplatformWebView`（跨平台 WebView）。

**UI 组件约定**：所有可点击的列表项（无论静态还是动态）统一使用 `com.app.common.ui.components.surface.Item`（默认高度 50dp，提供统一点击 ripple 效果），避免直接在 `Row`/`Column` 上使用 `clickable`。若传入自定义 `modifier`，需显式保留 `.height(50.dp)`，因为默认高度会被覆盖。详细示例见 `references/development_guide.md` 第 4.6 节。

#### 列表页交互组件

当列表页需要下拉刷新和上拉加载更多时，使用以下两个组件：

**下拉刷新 — `PullRefreshLayout`**

来自 `com.kmp.ui.components.refresh.PullRefreshLayout`：

```kotlin
PullRefreshLayout(
    modifier = Modifier.fillMaxSize(),
    viewModel = pullRefreshVM,
    pullRefreshEnable = pullRefreshEnabled,  // 控制下拉刷新是否启用
    onRefresh = { viewModel.onTriggerEvent(<Feature>Event.Refresh) }
) {
    LazyColumn(
        modifier = Modifier.fillMaxSize(),
        state = listState
    ) {
        // 列表内容
    }
}
```

- `pullRefreshVM` 通过 `rememberPullRefreshViewModel()` 创建。
- `pullRefreshEnable` 为 `true` 时启用下拉刷新，默认通常为 `true`。
- `onRefresh` 回调中触发 ViewModel 的刷新事件。

**上拉加载更多 — `LoadMoreView`**

来自 `com.kmp.ui.components.refresh.LoadMoreView`，置于 `LazyColumn` 的 `item` 中，通常放在列表末尾：

```kotlin
LazyColumn {
    items(list) { ... }
    item {
        LoadMoreView(
            modifier = Modifier.fillMaxWidth().padding(top = 16.dp).wrapContentHeight(),
            loadMoreState = loadMoreState,    // LoadMoreState 枚举：IDLE / LOADING / NO_MORE / ERROR
            onRetry = { viewModel.onTriggerEvent(<Feature>Event.LoadMore) },
            onLoadMore = { viewModel.onTriggerEvent(<Feature>Event.LoadMore) }
        )
    }
}
```

- `loadMoreState` 为 `LoadMoreState` 枚举值，由 ViewModel 的 State 驱动：
  - `IDLE` — 空闲，不显示任何提示。
  - `LOADING` — 加载中，显示 loading 动画。
  - `NO_MORE` — 无更多数据，显示"没有更多了"。
  - `ERROR` — 加载失败，显示错误提示 + 重试按钮。
- `onRetry` 在加载失败时用户点击重试触发。
- `onLoadMore` 在组件曝光时自动触发（通过 `LaunchedEffect` 监听可见性）。

### 4.5 MVI 约定

- `State`：`@Immutable` 数据类，实现 `ViewState`。
- `Event`：`@Immutable` 密封类，实现 `ViewEvent`。
- `Action`：`@Immutable` 密封类，实现 `ViewSingleAction`。
- `ViewModel`：继承 `BaseViewModel<Event, State, Action>`，在 `onTriggerEvent` 中分发事件。
- **ViewModel 注册必须使用 `factory`**（在 `Koin.kt` 中），禁止使用 `single`，以保证每次进入页面获得全新的 ViewModel 实例。
- **ViewModel 获取必须使用 `koinViewModel()`**（来自 `org.koin.compose.viewmodel.koinViewModel`），禁止手动 `get()` 或 `by viewModel()`，确保与 Compose 生命周期绑定。
- 状态更新统一用 `update { copy(...) }`，lambda 接收者就是当前 State，不要写 `update { current -> copy(...) }`。
- 调用 UseCase 时优先使用 `BaseViewModel.executeUseCase(...)` 辅助方法，统一处理加载、成功/失败回调与错误提示。

### 4.6 跨平台 Screen 约定

表现层的 Compose Screen **不要为 iOS 单独编写 `<Feature>IOS` 组件**。统一在 `<Feature>Screen` 中提供 `showTopBar: Boolean = true` 参数，由调用方决定是否需要显示顶部导航栏：

- Android 独立 `Activity` 中通常使用默认值 `showTopBar = true`，并通过 `onBack` 处理返回。
- iOS 通过 `ComposeUIViewController` 包装时传入 `showTopBar = false`，由 iOS 原生导航栏控制返回。

```kotlin
@Composable
fun SystemMessageScreen(
    showTopBar: Boolean = true,
    onBack: () -> Unit = {},
    // ... 其他参数
) {
    // ...
    DefaultScreenUI(
        // ...
        topBar = {
            if (showTopBar) {
                TopAppBar(
                    title = { Text("系统消息") },
                    navigationIcon = {
                        IconButton(onClick = onBack) {
                            // 返回图标
                        }
                    }
                )
            }
        }
    ) { /* ... */ }
}
```

## 5. Koin DI 注册规则（严格）

**在 `app-common/data/src/commonMain/kotlin/com/app/common/data/DataModule.kt` 中：**

- Repository 实现用 `single` 注册。
- API 实现用 `single` 注册。
- UseCase 用 `single` 注册。

```kotlin
single<NotificationSettingRepository> { NotificationSettingRepositoryImpl(get()) }
single<NotificationSettingApi> { NotificationSettingApiImpl(get(), get()) }
single { GetNotificationSettingUseCase(get()) }
single { SaveNotificationSettingUseCase(get()) }
```

**在 `app-main/main/src/commonMain/kotlin/com/app/main/di/Koin.kt` 中：**

- ViewModel 用 `factory` 注册。

```kotlin
factory { SystemNotificationViewModel(get(), get(), get()) }
```

> 若使用 `expect/actual object`，无需注册；若使用 `interface + Provider` 模式，则需注册接口与实现。

## 6. Android 入口与跳转

1. 创建 `app-main/<module>/src/androidMain/kotlin/com/app/main/<module>/.../<feature>/<Feature>Activity.kt`。
2. 在对应模块的 `AndroidManifest.xml` 中注册 Activity。
3. 在对应模块入口文件（如 `MineModule.kt`）中添加 `open<Feature>(context: Context)` 跳转方法。

## 7. 最小开发清单

开发完成后逐项检查：

- [ ] `app-common/domain/<feature>/` 已创建领域模型、Repository 接口、UseCase。
- [ ] `app-common/data/<feature>/` 已创建 API、DTO、Repository 实现。
- [ ] 如需平台能力，`kmp-common/common/src/<sourceSet>/.../<feature>/` 已创建 `expect/actual`。
- [ ] `app-main/<module>/.../<feature>/viewmodel/` 已创建 ViewModel。
- [ ] `app-main/<module>/.../<feature>/` 已创建 Screen；页面仅做组装，底层可复用 UI 已下沉到 `app-common/ui`。
- [ ] `app-main/<module>/src/androidMain/.../<feature>/` 已创建 Activity。
- [ ] `AndroidManifest.xml` 已注册 Activity。
- [ ] 对应模块入口文件已添加跳转方法。
- [ ] `DataModule.kt` 已按 `single` 注册 Repository / API / UseCase。
- [ ] `Koin.kt` 已按 `factory` 注册 ViewModel。
- [ ] 已执行编译验证：`compileKotlinMetadata` → `compileDebugKotlinAndroid`。

## 8. 常用编译命令

```bash
# 1. 验证 commonMain 元数据编译
./gradlew :app-common:data:compileKotlinMetadata \
          :app-common:domain:compileKotlinMetadata \
          :app-main:<module>:compileKotlinMetadata \
          :app-main:main:compileKotlinMetadata \
          :kmp-common:common:compileKotlinMetadata --no-daemon -q

# 2. 验证 Android 平台编译
./gradlew :app-main:<module>:compileDebugKotlinAndroid \
          :kmp-common:common:compileDebugKotlinAndroid --no-daemon -q
```

## 9. 参考示例

完整参考实现：

- 领域层：`app-common/domain/src/commonMain/kotlin/com/app/common/domain/notification/`
- 数据层：`app-common/data/src/commonMain/kotlin/com/app/common/data/notification/`
- 平台能力：`kmp-common/common/src/.../kotlin/com/kmp/common/notification/`
- 表现层：`app-main/mine/src/commonMain/kotlin/com/app/main/mine/setting/systemNotification/`
- Android 入口：`app-main/mine/src/androidMain/kotlin/com/app/main/mine/setting/systemNotification/SystemNotificationActivity.kt`
- DI 注册：`app-common/data/DataModule.kt`、`app-main/main/src/commonMain/kotlin/com/app/main/di/Koin.kt`

详细代码示例与约定，请阅读 `references/development_guide.md`。

