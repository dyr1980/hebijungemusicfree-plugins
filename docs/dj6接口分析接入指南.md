# dj6.cn（皇族DJ学院）接口全面分析与安卓接入指南

> 目标站点：`https://www.dj6.cn/mlist-2-1.html`
> 适用项目：AjunDJ（Android / Jetpack Compose + Media3 + 多音乐源）
> 分析日期：2026-08-11
> 结论先行：**该站是织梦(DedeCMS) 服务端渲染站点，没有 JSON/REST API，所有数据通过 HTML 页面提供，安卓端必须用 HTML 解析（Jsoup）获取。** 搜索已关闭、无每首歌封面、无结构化时长/大小/歌手字段、音质仅约 32kbps 且无高品分轨——这些是接入前必须知道的限制。

---

## 一、站点架构总览

| 维度 | 结论 |
|------|------|
| 建站程序 | 织梦 DedeCMS（HTML 服务端渲染） |
| 数据接口 | **无** JSON API，全部为 `.html` 页面 |
| 接入方式 | 抓取 HTML + Jsoup 解析（或正则） |
| 移动端 | 有独立移动站 `/m/*.php`，但 `m/list.php` 只返回菜单不返回歌曲，**音乐列表以桌面版 `mlist-*.html` 为准** |
| 音频托管 | `mp3.dj6.cn`（与 `www.dj6.cn` 同主体，支持 Range 断点续传） |
| 视频托管 | **腾讯视频**（仅存 vid，iframe 内嵌） |

---

## 二、URL 路由规则（接入核心）

| 功能 | URL 模板 | 说明 |
|------|----------|------|
| 音乐列表 | `mlist-{tid}-{page}.html` | 分类列表+分页，核心数据源 |
| 歌曲详情 | `play-{id}.html`（桌面） / `m/view.php?aid={id}`（移动） | 播放页，含音频/视频 |
| 视频列表 | `videolist-{tid}-{page}.html` | 音乐视频分类列表 |
| 视频详情 | `video-{id}.html` | 含腾讯视频 iframe |
| 图片列表 | `piclist-{tid}-{page}.html` | 图库 |
| 资讯列表 | `newslist-{tid}-{page}.html` | 文章 |
| 课程列表 | `kclist-{tid}.html` | 培训课程 |
| 音频播放器 | `/player/m.php?m=media/YYMMDD/文件名.mp3&t=歌名` | iframe 包装，内部 jPlayer 加载直链 |
| **音频直链** | `https://mp3.dj6.cn/media/YYMMDD/YYMMDDHHMMSSsss.mp3` | **安卓播放/下载直接用这个** |
| 视频播放 | `https://v.qq.com/iframe/player.html?vid=XXXX` | 腾讯视频，需 WebView |

> 音频路径规律：`media/` 后第一段 `260807` = 上传日期 2026-08-07；文件名 `2608071714071536` = `YYMMDDHHMMSSsss`（精确到毫秒）。更新时间可从该日期推断。

---

## 三、各功能模块接口详解

### 1. 分类（Categories）

**音乐主分类（4 个，无子分类，音乐列表页只有这 4 个 Tab）：**

| tid | 名称 | 列表首页 |
|-----|------|----------|
| 2 | DJ舞曲 | `/mlist-2-1.html` |
| 3 | 酒吧舞曲 | `/mlist-3-1.html` |
| 4 | 中文舞曲 | `/mlist-4-1.html` |
| 5 | 舞曲串烧 | `/mlist-5-1.html` |

**视频子分类（5 个，对应 `videolist-{tid}-1.html`）：**

| tid | 名称 |
|-----|------|
| 24 | DJ视频 |
| 25 | DJ舞曲视频 |
| 26 | DJ大赛视频 |
| 27 | DJ活动视频 |
| 28 | DJ学员视频 |

> 其他板块（图库 piclist、资讯 newslist、课程 kclist）非音乐核心，按需接入。

### 2. 列表 / 分页

- 分页示例：`mlist-2-1.html` → `mlist-2-460.html`（**仅 DJ舞曲就有 460 页**，约 120 首/页，总量约 5.5 万首）。
- 列表页结构极简：**只有歌名 + 链接 + new/re 角标**，无封面/时长/大小。
- 解析选择器：`div.ny_main_song > ul > li > a[href^="/play-"]`，取 `href` 里的 id 与 `title` 文本。

### 3. 歌曲详情页（play 页面）元素解析映射

| 字段 | 是否在 HTML | 解析来源 / 方式 | 示例 |
|------|------------|----------------|------|
| 歌曲名 / 标题名 | ✅ | `<h1>` 文本 | `国语ProgHouse音乐情歌现场专属DJ舞曲` |
| 歌手 | ❌ | **无独立字段**，歌手信息混在标题文本里（如"国语""中英文"），需自行 NLP 拆分，准确度低 | — |
| 播放音频 URL | ✅ | `<iframe src="/player/m.php?m=media/...mp3&t=...">` → 拼为 `https://mp3.dj6.cn/{m参数}` | `https://mp3.dj6.cn/media/260807/2608071714071536.mp3` |
| 下载链接 | ✅ | `<a class="musicdown" title="media/...mp3">`，JS 拼为 `https://mp3.dj6.cn/{title}` | 同上直链 |
| 封面图 | ❌ | **无每首歌封面**；全站默认图标 `https://www.dj6.cn/images/000.png`。建议用默认图或按标题首字生成占位图 | — |
| 文件大小 | ❌ | 不在 HTML；用 `HEAD https://mp3.dj6.cn/...mp3` 读 `Content-Length`（实测 747746 B ≈ 730 KB） | — |
| 歌曲时长 | ❌ | 不在 HTML；用 ExoPlayer/MediaMetadata 读取，或 ffprobe（实测 **186.8 秒 ≈ 3:07**） | — |
| 更新时间 | ❌ | 不在 HTML；从媒体路径日期 `260807`→2026-08-07，或 `Last-Modified` 响应头推断 | — |
| 高品链接 | ❌ | **不存在**；`_320`/`_hq`/`_vip` 后缀全部 404；仅单一 mp3（**实测码率 ~32 kbps**） | — |
| 相关视频 | ✅ | `a[href^="/video-"]` | `/video-102671.html` |
| 排行榜 | ✅ | 右侧栏 `.ny_teach_r_rank > ul > li`（编号 1-9） | — |

### 4. 搜索（⚠️ 重点限制）

- `https://www.dj6.cn/plus/search.php?keyword=xxx`：**GET 与 POST 均返回 301 重定向到首页**（162 字节），即**搜索功能已在服务端关闭**。
- `tags.php?/关键词.html`：返回"DedeCMS提示信息"（标签搜索不可用）。
- `/m/search.php`：同样 301。
- **结论：网站没有任何可用的搜索接口。**
- **安卓端替代方案**：
  1. 正式方案：按分类抓取列表页缓存到本地数据库（Room），在本地做歌名关键词 `LIKE` 过滤（适合"在我的曲库里搜"）。
  2. 全站搜：调用第三方搜索引擎 `site:dj6.cn 关键词`（不稳定、易被判爬、不满足实时性）。
  3. 不建议依赖服务端搜索。

### 5. 榜单 / 排行榜

- **无独立排行榜 URL**。播放页右侧栏 "DJ舞曲排行榜" 是编号 1-9 的 Top 列表，"更多>>" 指向 `mlist-2-1.html`。
- 实质榜单 = 分类最新列表 `mlist-{tid}-1.html`（按上传时间倒序）。
- 安卓实现：榜单可直接复用"分类最新列表"接口；若需固定 Top N，解析播放页 `.ny_teach_r_rank` 侧栏。

### 6. 音乐视频

- 列表：`videolist-{tid}-{page}.html`，条目含**封面图** `https://www.dj6.cn/uploads/...jpg` + 标题 + `video-{id}` 链接。
- 详情：`video-{id}.html`，`<h1>` 标题 + 封面 + `<iframe src="https://v.qq.com/iframe/player.html?vid=XXXX">`。
- **播放**：视频托管在腾讯视频，dj6.cn 只有 `vid`。安卓端用 `WebView` 加载该 iframe，或提取 `vid` 后用腾讯视频 SDK/播放器。dj6.cn 本域无 mp4 直链。

---

## 四、安卓接入技术方案

### 4.1 依赖
```toml
// build.gradle.kts
implementation("org.jsoup:jsoup:1.17.2")          // HTML 解析
implementation("com.squareup.okhttp3:okhttp:4.12.0") // 网络 + UA 伪装
// Media3/ExoPlayer 已在 AjunDJ 技术栈中
```

### 4.2 数据模型（Kotlin）
```kotlin
data class Dj6Song(
    val id: String,            // play 页 id，如 "134746"
    val title: String,         // 歌名（含风格前缀）
    val audioUrl: String,      // https://mp3.dj6.cn/media/...mp3
    val downloadUrl: String,   // = audioUrl
    val coverUrl: String? = null, // 该站无封面，可置默认图
    val durationSec: Long? = null,    // 由 ExoPlayer 元数据获取
    val fileSize: Long? = null,       // 由 HEAD Content-Length 获取
    val updatedAt: String? = null,    // 由 media 路径日期推断
    val artist: String? = null        // 需从 title 拆分，准确度低
)

data class Dj6Category(
    val tid: Int,
    val name: String,
    val type: CategoryType     // MUSIC / VIDEO
)

enum class CategoryType { MUSIC, VIDEO }
```

### 4.3 列表解析（Jsoup 示例）
```kotlin
fun parseSongList(html: String): List<Dj6Song> {
    val doc = Jsoup.parse(html)
    return doc.select("div.ny_main_song ul li a[href^=/play-]").map { a ->
        val href = a.attr("href")            // /play-134746.html
        val id = href.substringAfter("play-").substringBefore(".html")
        Dj6Song(
            id = id,
            title = a.attr("title").ifEmpty { a.text() },
            audioUrl = "",   // 列表页无音频，需进 play 页或按规则推断
            downloadUrl = ""
        )
    }
}
```

### 4.4 播放页解析 + 音频直链构造
```kotlin
fun parsePlayPage(html: String): Dj6Song? {
    val doc = Jsoup.parse(html)
    val title = doc.selectFirst("h1")?.text() ?: return null
    val iframe = doc.selectFirst("iframe[src~=player/m\\.php]") ?: return null
    val media = Regex("m=([^&]+)").find(iframe.attr("src"))?.groupValues?.get(1)
        ?: return null
    val audioUrl = "https://mp3.dj6.cn/$media"
    val id = Regex("play-(\\d+)").find(doc.baseUri())?.groupValues?.get(1) ?: ""
    return Dj6Song(id = id, title = title, audioUrl = audioUrl, downloadUrl = audioUrl)
}
```

### 4.5 播放（Media3 / ExoPlayer）
```kotlin
// 直链可直接喂给 ExoPlayer，支持 Range 断点续传
val player = ExoPlayer.Builder(ctx).build()
player.setMediaItem(MediaItem.fromUri("https://mp3.dj6.cn/media/260807/2608071714071536.mp3"))
player.prepare()
// 时长/码率从 player.getDuration() / MediaMetadata 读取
```

### 4.6 视频播放（WebView 加载腾讯 iframe）
```kotlin
val vid = Regex("vid=([^\"&]+)").find(videoPageHtml)?.groupValues?.get(1)
webView.loadUrl("https://v.qq.com/iframe/player.html?vid=$vid&auto=0")
```

### 4.7 请求注意事项
- **UA 伪装**：列表/播放页用桌面 UA 抓取最稳（`Mozilla/5.0 ... Chrome/120`）；移动 UA 会触发跳转 `/m/view.php`。
- **请求频率**：无官方 API，建议加延时 + 本地缓存，避免被封 IP。
- **HTTPS**：`mp3.dj6.cn` 与 `www.dj6.cn` 均需 HTTPS。

---

## 五、限制与风险清单（接入前必读）

| 限制 | 影响 | 应对 |
|------|------|------|
| 搜索已关闭 | 无法实现服务端搜索 | 本地曲库 LIKE 过滤 |
| 无每首歌封面 | 列表/播放页无图 | 默认占位图或标题首字生成 |
| 无结构化时长/大小/歌手 | 需客户端补算 | HEAD 取大小、ExoPlayer 取时长、歌手靠 NLP 拆分（不准） |
| **音质仅 ~32 kbps** | DJ 播放器音质体验差 | 如实告知用户；该站无高品源 |
| 无高品链接 | `_320`/`_hq` 全 404 | 放弃高品，仅 32kbps 可用 |
| HTML 结构无版本控制 | 改版即解析失效 | 解析层做容错 + 监控 |
| 数据量巨大 | 分类2 约 5.5 万首 | 分页懒加载 + 本地索引 |
| 版权 | 需确认授权 | 上线前核实 |

---

## 六、验证记录（已实测）

- ✅ 解析脚本验证：播放页正确提取 标题 / 媒体路径 / 下载直链 / 16 个相关视频 / Top3 排行；列表页正确提取 120 首（id+标题）+ 4 个分类 Tab。
- ✅ 音频直链 `https://mp3.dj6.cn/media/260807/2608071714071536.mp3` → HTTP 200，`Content-Type: audio/mpeg`，支持 `Accept-Ranges: bytes`。
- ✅ 音频实测参数：**时长 186.8s，大小 747746B，码率 32020 bps（≈32 kbps）**。
- ✅ 高品变体探测：`_320` / `_hq` / `_vip` / `-320` 后缀全部 404。
- ✅ 搜索探测：`search.php`（GET/POST）、`tags.php`、`/m/search.php` 均不可用。
- ✅ 视频确认：腾讯视频 iframe 托管，vid 形式。

---

## 七、推荐接入优先级

1. **分类列表** `mlist-{tid}-{page}.html` → 歌曲 id+标题（数据源）
2. **播放页** `play-{id}.html` → 音频直链 + 视频 + 排行榜（详情源）
3. **视频** `videolist` + `video-{id}` → 腾讯 vid（音乐视频源）
4. **搜索** → 本地缓存 + LIKE（绕过已关闭的服务端搜索）
5. **补充字段** → HEAD 取大小、ExoPlayer 取时长、路径日期取更新时间
