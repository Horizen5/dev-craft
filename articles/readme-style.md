# 怎么写一个让人想点 Star 的 GitHub README

> **一句话方法**：README 不是说明书，是门面。头部三件套（一句话简介 + 下载链接 + shields.io 徽章）决定用户停留还是划走；剩下的结构化信息（表格、目录、更新日志）决定他 Star 还是关掉。

**标签**：`GitHub` `README` `shields.io` `文档` `项目门面`
**来源**：WorkBuddy / CMCCSlim / dev-craft 三个仓库的 README 实践（[WorkBuddy](https://github.com/Horizen5/WorkBuddy) · [CMCCSlim](https://github.com/Horizen5/CMCCSlim)）

---

## 为什么大部分 README 是失败的

打开 GitHub Trending，90% 的仓库 README 长这样：

```markdown
# my-project

一个 xxx 项目。

## 安装

npm install xxx
```

问题不在内容少，在于**没有视觉焦点**。用户扫一眼 README 只花 3 秒，这 3 秒里他要看清楚三件事：

1. **这是什么** —— 一句话
2. **能装吗** —— 下载链接
3. **靠谱吗** —— 版本号、下载量、维护状态

三件全无 = 划走。

---

## 头部三件套：3 秒抓住注意力

### 1. 一句话简介

不是「一个基于 xxx 的 xxx 框架」，而是**说清楚解决什么问题**：

```markdown
# WorkBuddy

一个基于 WebView 的 WorkBuddy 移动端封装应用。打开即加载，支持 JavaScript、下拉刷新、返回键后退。
```

```markdown
# AppSlim 应用瘦身

一个通用的 Android 应用瘦身工具。识别 App 里塞的第三方 SDK 垃圾组件，用系统级组件冻结把它们关掉。
```

关键：**用一句完整的话，主语 + 动词 + 宾语**，不要用名词堆叠。

### 2. 下载链接 —— 给一个「点一下就能装」的入口

```markdown
### 下载

**[WorkBuddy v1.0.0](https://github.com/Horizen5/WorkBuddy/releases/latest)** · Release APK · Android 5.0+
```

```markdown
### 下载

**[AppSlim v2.0.0](https://github.com/Horizen5/CMCCSlim/releases/latest)** · 1.06 MB · 需要 Root
```

要素：**链接指向 `/releases/latest`**（永远跳最新版），后面跟上文件大小和前置条件。

### 3. shields.io 徽章 —— 视觉锚点

这是整篇文章的核心。shields.io 提供动态徽章，自动从 GitHub API 拉取数据：

```markdown
[![Release](https://img.shields.io/github/v/release/Horizen5/WorkBuddy?label=%E6%9C%80%E6%96%B0%E7%89%88%E6%9C%AC)](https://github.com/Horizen5/WorkBuddy/releases/latest)
[![Downloads](https://img.shields.io/github/downloads/Horizen5/WorkBuddy/total?label=%E4%B8%8B%E8%BD%BD%E9%87%8F)](https://github.com/Horizen5/WorkBuddy/releases)
[![Platform](https://img.shields.io/badge/platform-Android-3DDC84?logo=android&logoColor=white)](https://github.com/Horizen5/WorkBuddy/releases)
[![minSdk](https://img.shields.io/badge/minSdk-21-EF6C00?logo=android&logoColor=white)](https://github.com/Horizen5/WorkBuddy)
```

渲染效果：

- **最新版本** —— `github/v/release` 自动拉取最新 release tag
- **下载量** —— `github/downloads/.../total` 自动统计 release 附件总下载次数
- **平台** —— 静态徽章，用 Android 品牌色 `#3DDC84` + logo
- **minSdk** —— 静态徽章，用橙色 `#EF6C00` 区分

---

## shields.io 徽章速查

### GitHub 动态徽章（自动更新）

| 徽章 | URL 模板 | 效果 |
|------|----------|------|
| 最新版本 | `https://img.shields.io/github/v/release/{owner}/{repo}` | v1.0.0 |
| 下载总量 | `https://img.shields.io/github/downloads/{owner}/{repo}/total` | 1.2k |
| Star 数 | `https://img.shields.io/github/stars/{owner}/{repo}` | 42 |
| Fork 数 | `https://img.shields.io/github/forks/{owner}/{repo}` | 8 |
| Issues | `https://img.shields.io/github/issues/{owner}/{repo}` | 3 |
| 最后提交 | `https://img.shields.io/github/last-commit/{owner}/{repo}` | 2026-07 |
| 许可证 | `https://img.shields.io/github/license/{owner}/{repo}` | MIT |
| 语言数 | `https://img.shields.io/github/languages/count/{owner}/{repo}` | 4 |
| 代码大小 | `https://img.shields.io/github/languages/code-size/{owner}/{repo}` | 120 KB |

### 静态徽章（自定义文本）

```
https://img.shields.io/badge/{标签}-{内容}-{颜色}
```

| 示例 | URL |
|------|-----|
| `platform-Android-3DDC84` | 绿色 Android 标识 |
| `minSdk-21-EF6C00` | 橙色 SDK 版本 |
| `build-passing-brightgreen` | 绿色构建状态 |
| `coverage-85%25-green` | 测试覆盖率 |

### 参数技巧

| 参数 | 作用 | 示例 |
|------|------|------|
| `?label=自定义标签` | 替换左侧文字 | `?label=最新版本` |
| `?logo=android` | 左侧加图标 | 官方 [Simple Icons](https://simpleicons.org) 名 |
| `?logoColor=white` | 图标颜色 | white / black / hex |
| `?style=for-the-badge` | 大号样式 | 适合顶部横排 |
| `?style=flat-square` | 方角样式 | 适合表格内 |

### 中文 label 的 URL 编码

shields.io 的 label 支持中文，但需要 URL 编码：

| 中文 | 编码后 |
|------|--------|
| 最新版本 | `%E6%9C%80%E6%96%B0%E7%89%88%E6%9C%AC` |
| 下载量 | `%E4%B8%8B%E8%BD%BD%E9%87%8F` |
| 平台 | `%E5%B9%B3%E5%8F%B0` |

> 偷懒方法：用 `?label=` 传英文（`Latest` / `Downloads`），省去编码麻烦，也更适合国际化。

---

## 推荐的 README 结构

从 WorkBuddy 和 CMCCSlim 提炼出的模板：

```markdown
# {项目名}

{一句话简介：主语 + 动词 + 宾语，说清楚解决什么问题}

### 下载

**[版本号](releases/latest 链接)** · 文件大小 · 前置条件

[![Release](版本徽章)](releases/latest 链接)
[![Downloads](下载量徽章)](releases 链接)
[![Platform](平台徽章)](仓库链接)
[![License](许可证徽章)](LICENSE 链接)

---

## 一、功能
{用列表列出核心功能，每条一句话}

## 二、技术参数 / 架构
{用表格展示：包名、minSdk、技术栈等}

## 三、目录结构
{用代码块展示目录树}

## 四、构建 / 安装
{用代码块展示命令}

## 五、使用
{步骤列表，配截图更佳}

## 六、更新日志
### v1.0.0
- 功能点列表

## 七、免责 / 许可
{简短声明}
```

### 关键原则

| 原则 | 说明 |
|------|------|
| **头部不超 10 行** | 简介 + 下载 + 徽章，超过 10 行用户还没看到重点就划走了 |
| **徽章不超过 5 个** | 太多反而没有视觉焦点，选最重要的（版本、下载量、平台、许可证） |
| **表格代替段落** | 技术参数、配置项用表格，别用长段落 |
| `---` 分隔章节 | 水平线让长文档有呼吸感 |
| **更新日志倒序** | 最新版本在最上面 |

---

## 实战对比

### Before（典型烂 README）

```markdown
# my-app

一个 app。

## install

download the apk
```

### After（套用三件套）

```markdown
# MyApp

一个 Android WebView 封装应用，支持 JavaScript、下拉刷新、返回键后退。

### 下载

**[MyApp v1.0.0](https://github.com/user/repo/releases/latest)** · 4.5 MB · Android 5.0+

[![Release](https://img.shields.io/github/v/release/user/repo?label=Latest)](https://github.com/user/repo/releases/latest)
[![Downloads](https://img.shields.io/github/downloads/user/repo/total?label=Downloads)](https://github.com/user/repo/releases)
[![Platform](https://img.shields.io/badge/platform-Android-3DDC84?logo=android&logoColor=white)](https://github.com/user/repo)

---

## 功能
- WebView 封装，支持 JavaScript 和 DOM 存储
- 下拉刷新、返回键后退
- 全分辨率启动图标

## 技术参数

| 项目 | 值 |
|------|-----|
| minSdk | 21 |
| targetSdk | 35 |
| 包名 | `com.myapp.app` |
```

同一个项目，前者让人划走，后者让人想点进去看。

---

## 一句话总结

> README 的头部三件套——**一句话简介 + 下载链接 + shields.io 徽章**——决定用户停留还是划走。
> 版本号和下载量徽章不是装饰，是让用户在 3 秒内判断「这项目靠谱吗」的视觉锚点。
> **先写好头部，再填结构化内容。**
