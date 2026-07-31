# 让 Android 应用列表秒开：Room 缓存 + 异步图标 + 懒分析

> **一句话方法**：列表即缓存，深度即懒加载。快列表只取「看得见」的字段，图标只解「看得见的项」，重活（组件扫描、行为分析）只在用户点击时才做。

**标签**：`Android` `Jetpack Compose` `Room` `性能` `列表优化`
**来源**：AppSlim Analyzer v0.6.2 应用列表模块重构（[Appslim](https://github.com/Horizen5/Appslim)）

---

## 为什么这件事最该做、却最常被做错

应用列表页（已安装 App、文件管理器、消息会话列表……）是用户**每次进入都要等**的页面，理应最快。
但它恰恰是最常见的性能黑洞，因为很多人把它写成：

> 进页面 → 遍历所有项 → 顺手把每个项的「重数据」也一起算出来 → 一起塞进列表。

具体到 Android 已安装应用列表，最常见的错误写法：

```kotlin
// ❌ 慢：列表阶段就把所有重活干完了
pm.getInstalledPackages(0)
    .mapNotNull { pi ->
        val ai = pi.applicationInfo ?: return@mapNotNull null
        val icon = ai.loadIcon(pm).toBitmap()   // 每个应用同步解码 Bitmap
        AppItem(pkg, label, version, icon, isSystem)
    }
    .sortedBy { it.label.lowercase() }
```

三个叠加的雷：

1. `getInstalledPackages(0)` 比 `getInstalledApplications` 重——它把每个包的 `PackageInfo` 全拉出来了，列表页根本用不到；
2. 在列表循环里**同步 `loadIcon` + `toBitmap`**，200 个应用逐个解码 Bitmap，且这些 Bitmap 全塞进 `StateFlow`，列表数据一变就跟着重组；
3. **每次启动都重来一遍**，没有缓存。装得多的机器上，列表出来前能明显感到白屏/卡顿。

---

## 正确架构（MT Manager 式）

核心思想：**列表只负责「展示 + 跳转」，重分析推到点击时**。

```
AppRepository
  ├─ fastList()     getInstalledApplications(0)        → 100~300ms 出列表，不碰组件、不解码图标
  ├─ cachedAll()    Room.app_cache 全表                → 命中即免扫描，直接显计数/版本/时间
  └─ enrich()       仅「缓存缺失 / APK 变化」的应用      → 一次轻量 PackageInfo 扫描，写回 Room
                     └─ 后台分批（每 25 个）刷新列表，避免 200 次重组
```

四条规则：

| 规则 | 做法 |
|------|------|
| **快列表零额外 flag** | 用 `getInstalledApplications(0)`，只取包名 / 名 / 源路径 / 系统标志 |
| **缓存兜底** | Room 存「列表展示 + 智能排序」所需的最小字段；APK 没变就直接命中 |
| **图标异步、只解可见项** | 图标不进列表数据流，卡片进入组合时才解码，命中内存 LRU 后滚回来复用 |
| **深度分析懒加载** | 组件扫描、行为分类只在用户点击某一项时才跑 |

---

## 关键代码骨架（Kotlin + Compose）

**数据层**：快列表即时返回，重活推到后台、批量回写。

```kotlin
class AppRepository(private val app: Context) {
    private val dao get() = AppDatabase.get(app).appCacheDao()

    // 快列表：零额外 flag，不碰组件、不解码图标
    suspend fun fastList(): List<FastApp> = withContext(Dispatchers.IO) {
        pm.getInstalledApplications(0)
            .filter { it.packageName != app.packageName }
            .map { ai ->
                FastApp(ai.packageName,
                        pm.getApplicationLabel(ai).toString(),
                        ai.sourceDir ?: "",
                        (ai.flags and ApplicationInfo.FLAG_SYSTEM) != 0)
            }
    }

    // 仅富化缓存缺失 / APK 变化的项；APK 没变（apkPath 一致）直接复用
    suspend fun enrich(fast: FastApp): Meta = withContext(Dispatchers.IO) {
        val cached = dao.get(fast.pkg)
        if (cached != null && cached.apkPath == fast.apkPath) return@withContext cached.toMeta()
        val info = pm.getPackageInfo(fast.pkg, COMPONENT_FLAGS)   // 轻量一次，拿计数即可
        val meta = info.toMeta()
        dao.put(meta.toEntity())   // 写回 Room
        meta
    }
}
```

**图标缓存**：内存 LRU，只解可见项。

```kotlin
object IconCache {
    private val mem = LruCache<String, Bitmap>(400)
    suspend fun load(pm: PackageManager, pkg: String, apkPath: String): Bitmap? {
        mem.get(pkg)?.let { return it }
        return withContext(Dispatchers.IO) {
            val bmp = runCatching {
                pm.getApplicationInfo(pkg, 0).loadIcon(pm).toBitmap()
            }.getOrNull()
            bmp?.also { mem.put(pkg, it) }
        }
    }
}
```

**卡片**：图标异步加载，不拖累列表数据流。

```kotlin
@Composable
private fun AppItemCard(vm: MainViewModel, app: AppItem, onClick: () -> Unit) {
    var icon by remember(app.pkg) { mutableStateOf(vm.iconFor(app.pkg)) } // 先查内存命中
    LaunchedEffect(app.pkg) { if (icon == null) icon = vm.loadIcon(app.pkg, app.apkPath) }
    // ...icon 为 null 时用占位图，解码完自动刷新这一张卡
}
```

**排序**：放在组合层（`combine`），切排序只重排、不重扫数据。

```kotlin
val filteredApps = combine(apps, appFilter, search, appSort) { list, filter, q, sort ->
    list.filter { /* 过滤 + 搜索 */ }.let { filtered ->
        when (sort) {
            AppSort.NAME   -> filtered.sortedBy { it.label.lowercase() }
            AppSort.SLIM   -> filtered.sortedByDescending { it.backgroundCount * 1000 + it.compCount }
            AppSort.UPDATED-> filtered.sortedByDescending { it.updateTime }
            AppSort.RECENT -> filtered.sortedByDescending { launchMap[it.pkg] ?: 0L }
        }
    }
}.stateIn(...)
```

---

## 两个容易踩的坑（比原方案更优的两处偏差）

1. **用 `getInstalledApplications(0)`，不要 `GET_META_DATA`。**
   `GET_META_DATA` 会去读 APK 里的 `<meta-data>`，反而更慢；列表页用不到 meta-data，零 flag 更快一档。

2. **图标只进内存 LRU，不要落磁盘。**
   图标随 APK 版本变、又只占一屏，内存 LRU 已覆盖「可见 + 滚回看」；磁盘缓存收益低，还要处理版本失效，是负优化。

---

## 收益 & 适用场景

- **首屏**：快列表即时出现（文字 + 占位图标），计数/版本随后台富化陆续补齐；
- **二次进入 / 杀进程重开**：Room 命中，几乎瞬间出带计数的列表，富化只在「新装 / 更新」时发生；
- **滚动**：只解可见卡片图标，内存平稳，不再有「一进列表就解码 200 个 Bitmap」的峰值。

**适用任何「列表 + 详情」结构**：文件管理器、会话列表、已装应用、音乐/视频库……一句话——
**列表层只做轻量展示与跳转，把重的 IO 和分析推到点击时，并用缓存避免重复劳动。**

> 注：真实耗时需在真机/模拟器用 systrace 确认；以上为架构层面的确定性改进，非基准测试数字。
