# Android WebView 封装：六个配置项决定体验上限

> **一句话方法**：WebView 封装不是「加载个 URL 就完事」，真正决定用户体验的是六个配置项——JS/DOM 存储、返回键后退、下拉刷新、UA 标识、视口适配、离线缓存——漏一个就是「能用但难用」。

**标签**：`Android` `WebView` `封装` `用户体验` `Gradle`
**来源**：WorkBuddy Android WebView 封装项目（[WorkBuddy](https://github.com/Horizen5/WorkBuddy)）

---

## 为什么 WebView 封装容易做烂

WebView 封装看起来是 Android 里最简单的活——`WebView.loadUrl()` 一行就跑起来了。但用户拿到手的第一感觉往往是：

> 「这跟在浏览器里打开有什么区别？甚至更卡。」

区别就在那六个配置项上。原生浏览器帮你做的事，WebView 默认全关着：

| 配置项 | 默认值 | 用户感知 |
|--------|--------|----------|
| JavaScript | 关 | 页面白屏 / 交互失效 |
| DOM 存储 | 关 | 登录态丢失 / 页面报错 |
| 返回键 | 退出 App | 误触退出，重新加载 |
| 下拉刷新 | 无 | 只能靠页面自己的刷新按钮 |
| UA 标识 | 默认 WebView UA | 服务端不识别，返回桌面版页面 |
| 视口适配 | 未设 | 手机上显示桌面版缩略图 |

六个全漏 = 用户还不如直接用浏览器打开。

---

## 正确配置（Java + XML）

### 1. WebSettings：六个必开项

```java
WebSettings settings = webView.getSettings();

// ① JavaScript —— 不开等于白封装
settings.setJavaScriptEnabled(true);

// ② DOM 存储 —— localStorage / sessionStorage / IndexedDB
//    不开的话，登录态、缓存全丢，SPA 页面直接报错
settings.setDomStorageEnabled(true);

// ③ 数据库存储 —— 旧版 WebSQL 兼容（API 19 以下）
settings.setDatabaseEnabled(true);

// ④ 视口适配 —— 不开就是桌面版缩略图
settings.setUseWideViewPort(true);
settings.setLoadWithOverviewMode(true);

// ⑤ 缩放控制 —— 封装场景通常关掉，避免双指缩放破坏布局
settings.setBuiltInZoomControls(false);
settings.setDisplayZoomControls(false);

// ⑥ 混合内容 —— 允许 HTTPS 页面加载 HTTP 资源
//    Android 5.0+ 默认禁止，很多老站点图片会裂
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    settings.setMixedContentMode(WebSettings.MIXED_CONTENT_ALWAYS_ALLOW);
}
```

### 2. 返回键后退而非退出

这是体验差距最大的一项。默认按返回键直接 `finish()` 退出 App，用户每次误触都要等页面重新加载。

```java
@Override
public void onBackPressed() {
    if (webView.canGoBack()) {
        webView.goBack();   // 先退网页历史
    } else {
        super.onBackPressed();  // 没有历史了才退出
    }
}
```

**一个细节**：`canGoBack()` 会在页面有重定向时产生「假后退」——你以为退了一层，实际还在同一个页面。如果目标站点有 302 重定向，用户按返回键会卡在重定向循环里。解法是维护一个手动历史栈，在 `shouldOverrideUrlLoading` 里记录真实跳转路径。

### 3. 下拉刷新：SwipeRefreshLayout + WebView

WebView 自己没有下拉刷新，用 `SwipeRefreshLayout` 包一层：

```xml
<!-- activity_main.xml -->
<androidx.swiperefreshlayout.widget.SwipeRefreshLayout
    android:id="@+id/swipeRefresh"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <WebView
        android:id="@+id/webView"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</androidx.swiperefreshlayout.widget.SwipeRefreshLayout>
```

```java
SwipeRefreshLayout swipeRefresh = findViewById(R.id.swipeRefresh);

swipeRefresh.setOnRefreshListener(() -> {
    webView.reload();
});

// 页面加载完成后自动收起刷新动画
webView.setWebViewClient(new WebViewClient() {
    @Override
    public void onPageFinished(WebView view, String url) {
        swipeRefresh.setRefreshing(false);
    }
});
```

**踩坑**：`SwipeRefreshLayout` 会拦截竖直方向的触摸事件。如果网页内有竖向滚动的列表（大多数页面都有），下滑到顶部时再继续拉才能触发刷新——这是正确行为。但如果你发现网页完全无法滚动，检查 `SwipeRefreshLayout` 是否包了多余的 `NestedScrollView`。

### 4. User-Agent 追加标识

服务端经常靠 UA 判断客户端类型。默认 WebView 的 UA 是 `...; wv`，很多网站不认这个后缀，会返回桌面版页面。

```java
String ua = settings.getUserAgentString();
// 追加自定义标识，不覆盖原始 UA
settings.setUserAgentString(ua + " WorkBuddyApp/1.0");
```

**为什么不直接覆盖**：原始 UA 里有 Android 版本、设备型号、WebView 内核版本等信息，服务端可能依赖这些做兼容。只追加不覆盖，服务端既能识别「这是 WorkBuddy App」，又能拿到设备信息做适配。

### 5. 外部链接在浏览器打开

封装场景下，用户点页面里的外部链接（比如「联系我们」「隐私政策」）时，通常希望在系统浏览器打开，而不是在 WebView 里跳走。

```java
webView.setWebViewClient(new WebViewClient() {
    @Override
    public boolean shouldOverrideUrlLoading(WebView view, WebResourceRequest request) {
        String url = request.getUrl().toString();
        // 同域链接在 WebView 内打开，跨域跳系统浏览器
        if (url.startsWith("https://www.workbuddy.cn")) {
            return false;  // 不拦截，WebView 正常加载
        }
        Intent intent = new Intent(Intent.ACTION_VIEW, Uri.parse(url));
        startActivity(intent);
        return true;  // 拦截，交给系统浏览器
    }
});
```

---

## 移动端视口问题：封不住的坑

WebView 封装能解决客户端体验，但解决不了服务端的布局问题。在 WorkBuddy 项目中，手机端（视口 320-412px）发现两类典型问题：

### 问题一：固定宽度容器超出视口

```css
/* 服务端的 CSS */
.cloud-welcome__header {
    width: 540px;  /* 固定 540px，390px 视口直接溢出 */
}
```

登录按钮右边缘到达 x=524px，超出 390px 视口约 134px，按钮被裁切。这不是 WebView 的问题，是服务端 CSS 没做响应式。

**WebView 侧的临时解法**（治标不治本）：

```java
// 注入 CSS 强制适配视口
String css = "document.head.insertAdjacentHTML('beforeend', " +
    "'<style>.cloud-welcome__header{width:100%!important;max-width:100vw!important;}</style>');";
webView.evaluateJavascript(css, null);
```

**正确解法**：服务端加媒体查询 + 弹性布局：

```css
@media (max-width: 600px) {
    .cloud-welcome__header {
        width: 100%;
        max-width: 100vw;
        box-sizing: border-box;
    }
}
```

### 问题二：标题未居中

```css
/* 服务端用 text-align: center，但父容器宽度未约束 */
.cloud-welcome__title {
    text-align: center;
    /* 父容器 540px，视口 390px，整体偏移导致视觉上不居中 */
}
```

根因和问题一相同：容器宽度超出视口。修了容器宽度，标题自然居中。

> **教训**：WebView 封装前，先在手机浏览器里测一遍目标网站。如果浏览器里就有布局问题，WebView 里一样会有——这不是封装能修的。

---

## 构建与签名

### Gradle 配置要点

```gradle
android {
    compileSdk 35
    defaultConfig {
        applicationId "com.workbuddy.app"
        minSdk 21              // Android 5.0，覆盖 99%+ 设备
        targetSdk 35           // 最新 API level
    }
    // Release 签名配置
    signingConfigs {
        release {
            storeFile file("workbuddy.keystore")
            storePassword "workbuddy"
            keyAlias "workbuddy"
            keyPassword "workbuddy"
        }
    }
    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled false  // WebView 应用不建议混淆
        }
    }
}
```

**为什么不混淆**：WebView 应用几乎不包含业务逻辑（就一个 `MainActivity`），混淆收益极低，反而可能导致 `shouldOverrideUrlLoading` 等回调被混淆后失效。如果确实要混淆，保留 WebView 相关回调：

```proguard
-keepclassmembers class * extends android.webkit.WebViewClient {
    public *;
}
-keepclassmembers class * extends android.webkit.WebChromeClient {
    public *;
}
```

### 签名密钥管理

```bash
# 生成密钥
keytool -genkey -v -keystore workbuddy.keystore \
    -alias workbuddy -keyalg RSA -keysize 2048 -validity 10000

# 查看签名信息
keytool -list -v -keystore workbuddy.keystore
```

> **坑**：`.gitignore` 里一定要排除 `*.keystore`。密钥泄露 = 别人可以用你的包名发 APK 覆盖你的应用。WorkBuddy 仓库里的 keystore 是临时演示用的，正式发布必须替换。

---

## 图标：一套源图生成全分辨率

Android 需要 5 套分辨率的启动图标，手切太累：

| 目录 | 密度 | 尺寸 |
|------|------|------|
| `mipmap-mdpi` | 160dpi | 48x48 |
| `mipmap-hdpi` | 240dpi | 72x72 |
| `mipmap-xhdpi` | 320dpi | 96x96 |
| `mipmap-xxhdpi` | 480dpi | 144x144 |
| `mipmap-xxxhdpi` | 640dpi | 192x192 |

准备一张 512x512 的源图（`ic_launcher_512.png`），用 Android Studio 的 Image Asset Studio 自动生成各分辨率。如果用命令行，ImageMagick 一行搞定：

```bash
for size in 48 72 96 144 192; do
    dir="mipmap-$([ $size -eq 48 ] && echo 'mdpi' || [ $size -eq 72 ] && echo 'hdpi' || [ $size -eq 96 ] && echo 'xhdpi' || [ $size -eq 144 ] && echo 'xxhdpi' || echo 'xxxhdpi')"
    mkdir -p "app/src/main/res/$dir"
    convert ic_launcher_512.png -resize ${size}x${size} "app/src/main/res/$dir/ic_launcher.png"
done
```

---

## 一句话总结

> WebView 封装的价值不在「封装」本身，而在于把浏览器的体验差距补上——
> **六个配置项（JS / DOM 存储 / 返回键 / 下拉刷新 / UA / 视口）全到位，用户才感觉这是个 App 而不是浏览器书签。**
> 但 WebView 修不了服务端的布局问题——封装前先在手机浏览器里测一遍目标网站。
