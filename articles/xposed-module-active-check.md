# Xposed/LSPosed 模块的「已激活」状态怎么写才不会翻车

**方法**：激活检测不要只依赖一条路径。
「模块 Hook 自己」需要三个条件同时成立，任一环断掉都会误报未激活 ——
再加两条不依赖 Hook 的兜底检测，成本几行代码。

---

## 反模式

几乎所有教程都是这么写的：

```kotlin
class MainActivity : Activity() {
    // 「这个方法会被 LSPosed 改写返回值」
    private fun isModuleEnabled(): Boolean = false

    override fun onCreate(b: Bundle?) {
        val active = isModuleEnabled()
        banner.text = if (active) "已激活" else "未激活"
    }
}
```

这段代码有**三个**独立的失效点，而且失效时的表现完全一样
（永远显示"未激活"），没有任何线索指向真正的原因。

### 失效点 ①：那句注释是错的

LSPosed **不会**自动去猜你的方法叫什么名字然后改写它。

真正的机制是**模块 Hook 自己**：模块自己的进程被注入后，
由模块的 Hook 入口把这个方法的返回值替换掉。

所以入口里必须有这段：

```kotlin
override fun handleLoadPackage(lpparam: XC_LoadPackage.LoadPackageParam) {
    if (lpparam.packageName == MODULE_PKG) {
        XposedHelpers.findAndHookMethod(
            "com.example.ui.MainActivity", lpparam.classLoader,
            "isModuleEnabled",
            XC_MethodReplacement.returnConstant(true)
        )
        return
    }
    // ... 正常的目标应用逻辑
}
```

而绝大多数模块的入口第一行是 `if (packageName != TARGET) return`，
把自己的进程直接过滤掉了 —— 这半边逻辑压根没机会执行。

### 失效点 ②：模块自己不在作用域里

```
scope.list:  com.target.app          ← 只写了目标应用
module.prop: staticScope=true        ← 还锁死了
```

LSPosed 只给**作用域内**的应用注入。模块自己不在作用域里，
自己的进程就不会被注入，①里那段 Hook 代码根本不会运行。

`staticScope=true` 更要命：它会锁死作用域列表，
用户在 LSPosed 界面里连手动补勾的入口都没有，只能重装。

正确写法是两处都要加上模块自身包名：

```
# assets/META-INF/xposed/scope.list
com.target.app
com.example.mymodule       ← 模块自己

# res/values/arrays.xml
<string-array name="xposed_scope">
    <item>com.target.app</item>
    <item>com.example.mymodule</item>
</string-array>
```

并且把 `staticScope` 设为 `false`，留一条人工补救的路。

### 失效点 ③：方法被编译器内联

```kotlin
private fun isModuleEnabled(): Boolean = false
```

private + 常量返回，Kotlin 编译器和 D8 会直接把它内联成字面量。
**调用点根本不产生方法调用**，Hook 挂上去也永远不触发。

用 `dexdump -d` 看就很清楚。错误写法编译出来是：

```
const/4 v0, 0x0        ← 直接是常量，方法可能整个消失
return v0
```

正确写法应该是：

```
access: 0x0011 (PUBLIC FINAL)              ← public，Hook 找得到
iget-boolean v0, v0, ...hookedFlag:Z       ← 读字段，编译器不敢内联
return v0
```

调用点也要确认是 `invoke-virtual` 而不是被消掉了。

---

## 正确写法

### 1. 方法本身杜绝内联

```kotlin
@Volatile
private var hookedFlag = false

// public + 读字段，两条都是防内联的关键
fun isModuleEnabled(): Boolean = hookedFlag
```

### 2. 入口处理模块自身

```kotlin
companion object {
    const val TARGET_PKG = "com.target.app"
    const val MODULE_PKG = "com.example.mymodule"
}

override fun handleLoadPackage(lpparam: XC_LoadPackage.LoadPackageParam) {
    if (lpparam.packageName == MODULE_PKG) {
        markSelfActive(lpparam)
        return
    }
    if (lpparam.packageName != TARGET_PKG) return
    // ...
}
```

### 3. 两条不依赖 Hook 的兜底

这是整篇最实用的部分。上面三个条件任何一个没配好都会误报，
而这两条检测完全不依赖 Hook 是否成功：

```kotlin
/**
 * XposedBridge 是 compileOnly 依赖，不会打进 APK。
 * 那它唯一的来源就是 LSPosed 注入时塞进来的 ——
 * 加载得到 = 这个进程确实被注入了。
 */
private fun xposedInjected(): Boolean = try {
    Class.forName("de.robv.android.xposed.XposedBridge", false, classLoader)
    true
} catch (t: Throwable) {
    false
}

/**
 * MODE_WORLD_READABLE 在原生 Android 上会抛 SecurityException，
 * 只有 LSPosed 专门为模块放开了它。
 * 不抛异常 = 模块被 LSPosed 加载了。
 * 顺带还能判断配置能不能被宿主进程读到。
 */
private var prefWritable = false
sp = try {
    val p = getSharedPreferences(PREF_NAME, Context.MODE_WORLD_READABLE)
    prefWritable = true
    p
} catch (t: Throwable) {
    prefWritable = false
    getSharedPreferences(PREF_NAME, Context.MODE_PRIVATE)
}

// 三选一命中即可
val active = isModuleEnabled() || xposedInjected() || prefWritable
```

### 4. 状态条显示命中了哪几项

出问题时能一眼看出是哪一环断的，比一句干巴巴的"未激活"有用得多：

```kotlin
text = if (active) {
    val how = buildList {
        if (isModuleEnabled()) add("Hook")
        if (xposedInjected()) add("注入")
        if (prefWritable) add("配置可写")
    }.joinToString("/")
    "模块已激活（$how）"
} else {
    "模块未激活 · 请在 LSPosed 中启用，作用域勾选目标应用和本模块，然后重启手机"
}
```

---

## 实测踩坑记录

在 CMCCSlim（中国移动精简模块）上，这三个 bug **同时存在**了三个版本。

最坑的地方在于：**拦截功能一直是正常工作的**，
LSPosed 日志里拦截记录一条不少，只有状态条一直显示"未激活"。
用户完全有理由认为模块坏了。

排查顺序建议：

| 检查 | 方法 |
|---|---|
| 功能到底有没有生效 | LSPosed 日志里搜模块 TAG，有记录就是好的 |
| 模块自己在不在作用域 | LSPosed → 模块 → 作用域，看列表里有没有模块自己 |
| 方法有没有被内联 | `dexdump -d classes.dex \| grep -A5 "name.*isModuleEnabled"` |

第三条最容易被忽略，因为源码看着完全正常。**必须看字节码。**

---

## 一句话总结

> 激活检测的三个前置条件（作用域含自己 / 入口处理自己 / 方法不被内联）
> 全是隐式的，任一失败都静默降级成"未激活"。
> **加两条不依赖 Hook 的兜底检测，比把三个条件都配对更可靠。**
