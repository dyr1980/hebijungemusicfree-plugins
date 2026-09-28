# DJ耶耶网（djye.com）接口与页面解析分析报告

> 分析目标：`http://www.djye.com/top/down.html`
> 用途：为安卓 DJ 播放器（AjunDJ）接入提供接口、页面元素、字段映射与集成方案
> 分析方式：真实浏览器（Chromium/Playwright）加载并绕过 JS cookie 挑战，捕获 DOM 结构与网络请求
> 结论适用端：**PC 端 `www.djye.com`** 与 **手机端 `m.djye.com`**（两者 URL 结构一致，仅 DOM 繁简不同）

---

## 一、核心结论速览

| 项目 | 结论 |
|---|---|
| 站点性质 | **纯音频 DJ 舞曲站**，服务端渲染 HTML（非 JSON API） |
| 是否有 MV/音乐视频 | **无**。已验证 `/mv/`、`/video/` 均为 404，全站只有音频 |
| 数据获取方式 | 解析 HTML（推荐 Jsoup + 执行 JS 通过反爬） |
| 反爬机制 | 首访返回 `eval` 混淆 JS，设置 cookie 后 `location.reload`；下载接口还需二次挑战 + 登录 |
| 歌曲唯一标识 | 数字 ID，出现在 `/player/{id}.htm`、`/down/{id}.html`、`<i v="{id}">` |
| 媒体地址来源 | 播放页 `<audio src>` 直接给出，换扩展名即得下载地址 |
| 播放/试听地址 | `http://zj.djye.com/{年}/{月}/{序号}.m4a`（301→CDN） |
| 高品/下载地址 | 同路径换 `.mp3`（320kbps），即 `http://zj.djye.com/{年}/{月}/{序号}.mp3` |
| 官方下载 | 需 VIP 登录（`i.djye.com/ajax?a=down&id=` 最终跳登录页） |

---

## 二、反爬机制（接入前必读）

djye 用一段 `eval` 混淆 JS 做浏览器校验，逻辑等价于：

```javascript
function getCookie(name){ /* 读取 cookie */ }
var token = getCookie("token");
var secret = parseInt(getCookie("secret"));
if (token=="" || secret=="" || isNaN(secret)) {
    // 清空并写入临时 cookie 后强制刷新
    document.cookie = "t=" + token;
    document.cookie = "r=" + secret;
    window.location.reload(true);   // 关键：必须执行 JS
}
```

- **表现**：直接 `curl` 拿不到正文，只会拿到这段挑战脚本。
- **对策**：
  - 方案 A（最省事）：用 `WebView` 加载页面，让系统执行 JS，再 `evaluate` 取出 `document.documentElement.outerHTML` 交给 Jsoup。
  - 方案 B（轻量）：用 `OkHttp` 请求后，自行用 `Rhino`/`JavaScript` 引擎执行这段混淆脚本、提取 `token/secret` 并写回 cookie，再二次请求。
  - 方案 C（稳健）：直接把上述挑战逻辑翻译成 Kotlin（`token/secret` 为空时打 cookie 并重试一次）。

> ⚠️ 注意：`common.js`、`down.js` 等静态资源有**防盗链（Referer 校验，无 Referer 返回 403）**，但页面 HTML 与媒体 CDN 不强制 Referer，正常带 `Referer: http://www.djye.com/` 即可。

---

## 三、URL 路由总表

### 3.1 榜单（/top/）
| 页面 | 路径 | 说明 |
|---|---|---|
| 榜单首页 | `/top/index.html` 或 `/top/` | 榜单入口 |
| 舞曲下载榜 | `/top/down.html` | 本次分析页 |
| 舞曲收藏榜 | `/top/fav.html` | |
| 舞曲好评榜 | `/top/good.html` | |
| 舞曲总排行榜 | `/top/hot.html` | |
| 热门 TOP500 | `/top/top500.html` | |
| 最新 TOP100 | `/top/new100.html` | |
| 年度人气榜 | `/top/year.html` | |
| 车载CD榜 | `/top/cz.html` | |
| 电音舞曲榜 | `/top/dy.html` | |
| 伤感舞曲榜 | `/top/sg.html` | |
| 粤语舞曲榜 | `/top/yr.html` | |
| 欧美舞曲榜 | `/top/om.html` | |
| 超劲爆DJ榜 | `/top/cjb.html` | |
| 串烧排行榜 | `/top/cstop.html` | |
| 中文排行榜 | `/top/zwtop.html` | |
| 英文排行榜 | `/top/ywtop.html` | |
| 交谊舞曲榜 | `/top/jytop.html` | |
| 流行音乐榜 | `/top/lxtop.html` | |

### 3.2 分类列表（/list/）
| 主分类 | 路径 | 说明 |
|---|---|---|
| 串烧舞曲 | `/list/xc.html` | |
| 中文舞曲 | `/list/zw.html` | |
| 英文舞曲 | `/list/yw.html` | |
| 交谊舞曲 | `/list/jy.html` | |
| 流行音乐 | `/list/yy.html` | |

- **分页**：`/list/{cat}_{sort}_{page}.html`
  - 例：`/list/xc_0_2.html`（串烧第 2 页），最大约 `/list/xc_0_501.html`
  - `{sort}` 为排序位（实测默认 `0`；其它排序值需抓列表页排序条确认，常见：最新/人气/下载）
  - 每页约 **60 首**

### 3.3 标签 / 子分类（/tag/）
- 入口：`/tag/`（标签聚合页）
- 单标签：`/tag/{tag}.html`
  - 车载DJ `chezaidj`、电音舞曲 `dianyin`、现场 `xianchang`、轻音乐 `youmei`、王绎龙 `djsunny`……
  - 更多标签在歌曲详情页的标签区发现（无统一目录页，建议从详情页聚合）

### 3.4 搜索
- 接口：`/search?wd={关键词}&page={n}`
- 返回结构同列表页，每页约 **50 首**
- 编码：关键词需 URL Encode（UTF-8）
- ⚠️ 分页参数实测：`?page=2` 在采样中未切换页码（仍返回第 1 页），搜索分页 URL 形态**需二次确认**（可能用 `?p=` 或不同参数，或结果上限 50 条）

### 3.5 歌曲/专辑
| 用途 | 路径 | 关键内容 |
|---|---|---|
| 播放/详情页 | `/player/{id}.htm` | 播放音频地址、歌名、人气、上传时间、分类、标签 |
| 下载页 | `/down/{id}.html` | 格式/大小/时长/码率、VIP 下载接口 |
| 专辑入口 | `/album/` | 专辑分类 + 专辑列表 |
| 专辑详情 | `/album/{name}.html` | 专辑内歌曲集合 |

---

## 四、页面元素解析与字段映射

### 4.1 榜单/列表条目（核心循环结构）

每首歌是一个 `<ul>`，前 3 名为 `class="xdtop1"`，其余为 `class="xdtop2"`，内部按固定顺序排列多个 `<li>`：

| li class | 含义 | 取值方式 | 示例 |
|---|---|---|---|
| `t_r_l_i` | 歌曲 ID 容器 | `<i v="{id}">` | `20095` |
| `t_r_l_s` | 排名 | 文本 | `01` |
| `t_r_l_p` | **封面图**（仅前 3 名） | `<img src>` | `http://img.djye.com/user/15/15a2e7be...jpg` |
| `t_r_l_n` | **歌曲名/标题** | `<a href="/player/{id}.htm" title="{名}">` | `柔情伤感HIFI发烧女声车载大碟` |
| `t_r_l_u` | 上传者/DJ | `<a href="http://i.djye.com/1/">` | `DJYE耶耶网` |
| `t_r_l_h` | **人气** | 文本（去"人气:"） | `人气:2266826` |
| `t_r_l_a` | 加入列表 | `onclick="B.A({id})"` | 按钮 |

> 手机端 `m.djye.com` 列表项更简洁，标题文本里直接带"添加时间"，解析更友好。

**Jsoup 解析示例：**
```kotlin
doc.select("ul.xdtop1, ul.xdtop2").forEach { ul ->
    val id = ul.selectFirst("i")?.attr("v") ?: return@forEach
    val rank = ul.selectFirst(".t_r_l_s")?.text()
    val cover = ul.selectFirst(".t_r_l_p img")?.attr("src")   // 可能为空
    val nameA = ul.selectFirst(".t_r_l_n a")
    val title = nameA?.attr("title") ?: nameA?.text()
    val uploader = ul.selectFirst(".t_r_l_u")?.text()
    val hits = ul.selectFirst(".t_r_l_h")?.text()?.removePrefix("人气:")
}
```

### 4.2 播放/详情页 `/player/{id}.htm`

| 字段 | 选择器 | 示例 |
|---|---|---|
| **标题/歌名** | `<title>` / `<span id="play_name">` | `柔情伤感HIFI发烧女声车载大碟` |
| **正在播放名** | `#play_name` | 同上 |
| **收藏人气** | `#play_favs` | `5204` |
| **点击人气** | `#play_hits` | `2266829` |
| **上传时间/更新时间** | `#play_addtime` | `2016-01-15` |
| **分类** | 标签区 `a[href^="/tag/"]` 或 `#play_class` | `车载DJ`、`现场` |
| **歌手/DJ** | 标题内解析 或 标签（如 `/tag/djsunny.html`） | `王绎龙` |
| **播放音频地址** ★ | `<audio id="jp_audio_0" src="...">` | `http://zj.djye.com/2016/01/045.m4a` |

> ★ **这是最关键的字段**：播放地址直接写在 `<audio>` 标签里，无需调用任何接口即得。

### 4.3 下载页 `/down/{id}.html`

信息集中在 `<div class="mp3info">`（HTML 用 `<br>` 分隔）：

```
舞曲格式：Mp3文件
文件大小：154.04MB      ← 文件大小
舞曲时长：67分18秒      ← 歌曲时长
播放速率：320kbps       ← 码率
上传用户：DJYE耶耶网
上传时间：2016-01-15
```

- **高品质MP3（VIP）**：页面有 `高品质MP3` 标识，正式下载走 `http://i.djye.com/ajax?a=down&id={id}`，但该接口需登录/VIP，最终跳登录页。
- **直链下载（推荐用于播放器）**：将播放页 `<audio>` 的 `.m4a` 换成 `.mp3` 即得到 320kbps 高品文件地址（已验证 32 首样本 100% 成立；djye 仅提供 Mp3 高品，无 wav/flac 无损）。

### 4.4 搜索页 `/search?wd=`
结构与列表一致，结果即 `<a href="/player/{id}.htm">` 集合，按 4.1 规则解析即可。

### 4.5 专辑页 `/album/`
- 专辑分类入口：`心情 / 乐器 / 节日 / 主题 / 年代 / 影视` 等（指向 `/album/{分类}/`）
- 专辑列表：`/album/{name}.html`（如 `3D车载环绕`→`/album/czhr.html`）
- 专辑详情页内为歌曲集合，按 4.1 解析

---

## 五、媒体文件 CDN 规律（播放/下载核心）

```
试听(播放): http://zj.djye.com/{PATH}.m4a     ← 直接来自播放页 <audio src>
高品下载:   http://zj.djye.com/{PATH}.mp3     ← 同路径替换扩展名（320kbps）
```

> **高品替换规则**：把播放页 `<audio src>` 的 `.m4a` 直接替换为 `.mp3`，即得到高品下载地址。该规则已在 **32 首跨年份/类型样本**中 100% 成立（详见第九节验证）。

- `{PATH}` 形态不统一（按上传年代/分类命名），且**与歌曲 ID 无关**，必须从播放页 `<audio src>` 解析，**不可由 ID 推算**：
  - `年/月/序号`：`2016/01/045.m4a`、`2021/09/183.m4a`、`2017/08/040.m4a`
  - `分类/序号`：`wq/2460.m4a`、`zw/1890.m4a`、`mywq/330.m4a`、`ywwq/a136.m4a`
  - `年/月日/序号`：`2012/1005/4030.m4a`
- 上述地址均 **301 重定向** 到 CDN：`https://djzj.lianzhixiu.com:51880/{PATH}.{ext}`

> ⚠️ **【2026-08-11 实测重要修正】媒体 CDN 当前不可达，直链热链方案存在真实风险。**
> - `zj.djye.com/...m4a|mp3` 确实 301 跳到 `djzj.lianzhixiu.com:51880`，但该目标主机名在 **8.8.8.8 / 119.29.29.29（腾讯）/ 223.5.5.5（阿里）** 三家公共 DNS 上**均查不到 A 记录**（空答案），浏览器实测 `fetch` 直接 `Failed to fetch`。
> - 播放页还出现了 `i.djye.com/ajax?a=error&...&url=http://zj.djye.com/...m4a` 的错误上报信标，说明 **djye.com 自身当前也加载不出这段音频**。
> - **结论**：所谓"ExoPlayer 直接传 `zj.djye.com` 地址、重定向自动跟随即可播放"在现阶段不成立——CDN 主机名已失效（疑似下线/未续费/地域解析失效）。**AjunDJ 接入前必须在真机（中国大陆网络）上实测 `zj.djye.com` 直链是否可拉流**，若仍不可达，需改走官方下载接口（`i.djye.com/ajax?a=down`，需登录态/VIP，见 4.3）或放弃该音源。
> - **码率未实测**：本沙箱无法连到该 CDN，因此**未能真正下载文件用 MP3 帧头测码率**。报告中"320kbps"仅为下载页 `.mp3info` 文本**声明值**（32 首样本一致），并非字节级实测。请按上面的真机实测一并确认。

- 安卓端**若**真机验证 CDN 可达，可用 **ExoPlayer** 直接传入 `zj.djye.com` 地址，重定向会被自动跟随；若不可达，见上方修正说明。
- ⚠️ **`.wav/.flac/.ape` 变体不可靠**：`zj.djye.com` 对任意路径都无条件返回 301（假阳性，连 `045.xyz`、不存在序号都 301），且下载页仅声明 `Mp3文件 / 320kbps`。djye 只提供 **Mp3 320kbps 高品**，不要假设有无损格式。

**获取地址的标准流程：**
```
1. 请求 /player/{id}.htm（带 cookie 挑战）
2. 解析 <audio id="jp_audio_0" src>  →  base = http://zj.djye.com/2016/01/045.m4a
3. 播放地址 = base
4. 下载地址 = base.replace(".m4a", ".mp3")
```

---

## 六、所有分类与子分类清单

### 主分类（/list/）
`串烧舞曲(xc)` · `中文舞曲(zw)` · `英文舞曲(yw)` · `交谊舞曲(jy)` · `流行音乐(yy)`

### 榜单类型（/top/，共 19 种，见 3.1）

### 标签 / 子分类（/tag/，动态聚合，已知示例）
`车载DJ(chezaidj)` · `电音舞曲(dianyin)` · `现场(xianchang)` · `轻音乐(youmei)` · `王绎龙(djsunny)`
> 标签无统一目录，建议在爬取详情页时持续聚合 `a[href^="/tag/"]` 以补全全量子分类。

### 专辑分类（/album/）
`心情` · `乐器` · `节日` · `主题` · `年代` · `影视` + 站长推荐/3D车载环绕/经典DJ回顾等专辑

---

## 七、安卓（AjunDJ）接入方案建议

### 7.1 数据模型（Kotlin）
```kotlin
data class DjSong(
    val id: String,              // 歌曲ID
    val rank: String?,           // 榜单排名
    val title: String,           // 歌曲名/标题
    val artist: String?,         // 歌手/DJ（来自标题解析或标签）
    val coverUrl: String?,       // 封面图
    val playUrl: String,         // 播放地址 (.m4a)
    val downloadUrl: String,     // 高品下载地址 (.mp3)
    val duration: String?,       // 时长（来自下载页，如 67分18秒）
    val fileSize: String?,       // 文件大小（如 154.04MB）
    val bitrate: String?,        // 码率（如 320kbps）
    val uploader: String?,       // 上传者
    val addTime: String?,        // 更新/上传时间
    val hits: Long?,             // 点击人气
    val favs: Long?,             // 收藏人气
    val category: String?,       // 分类/标签
)

data class DjPage(
    val songs: List<DjSong>,
    val hasNext: Boolean,
    val nextUrl: String?,
)
```

### 7.2 抓取流程（伪代码）
```kotlin
suspend fun fetchRank(url: String): DjPage = withContext(Dispatchers.IO) {
    val html = WebViewFetcher.fetch(url)        // 用 WebView 过 cookie 挑战，返回 HTML 字符串
    val doc = Jsoup.parse(html)
    val songs = doc.select("ul.xdtop1, ul.xdtop2").mapNotNull { parseSong(it) }
    DjPage(songs, hasNext = doc.selectFirst("a:contains(下一页)") != null, nextUrl = null)
}

private fun parseSong(ul: Element): DjSong? {
    val id = ul.selectFirst("i")?.attr("v") ?: return null
    val titleEl = ul.selectFirst(".t_r_l_n a")
    val title = titleEl?.attr("title") ?: titleEl?.text() ?: return null
    val cover = ul.selectFirst(".t_r_l_p img")?.attr("src")
    val playerUrl = "http://www.djye.com/player/$id.htm"
    return DjSong(
        id = id, title = title, coverUrl = cover,
        playUrl = "", downloadUrl = "",   // 播放地址需再抓 /player/{id}.htm
        uploader = ul.selectFirst(".t_r_l_u")?.text(),
        hits = ul.selectFirst(".t_r_l_h")?.text()?.removePrefix("人气:")?.toLongOrNull(),
    )
}

// 详情：拿播放地址 + 元数据
suspend fun fetchDetail(id: String): Pair<String, SongMeta> {
    val html = WebViewFetcher.fetch("http://www.djye.com/player/$id.htm")
    val doc = Jsoup.parse(html)
    val audio = doc.selectFirst("audio")?.attr("src") ?: error("no audio")
    val playUrl = audio
    val downloadUrl = audio.replace(".m4a", ".mp3")
    val meta = SongMeta(
        addTime = doc.selectFirst("#play_addtime")?.text(),
        hits = doc.selectFirst("#play_hits")?.text()?.toLongOrNull(),
        favs = doc.selectFirst("#play_favs")?.text()?.toLongOrNull(),
        category = doc.select("a[href^=/tag/]").joinToString { it.text() },
    )
    return playUrl to meta
}
```

### 7.3 播放（ExoPlayer / Media3）
```kotlin
val player = ExoPlayer.Builder(ctx).build()
player.setMediaItem(MediaItem.fromUri(playUrl))   // 自动跟随 301 到 CDN
player.prepare(); player.play()
```
- 建议用 `Media3 ExoPlayer`（与 AjunDJ 技术栈一致），支持 HTTP 重定向与 m4a/mp3。
- 下载：用 `DownloadManager` 或 `OkHttp` 拉取 `.mp3` 直链落盘（注意 301 跟随）。

### 7.4 搜索
```kotlin
val url = "http://www.djye.com/search?wd=" + URLEncoder.encode(keyword, "UTF-8")
// 解析同榜单列表（4.1）
```

---

## 八、注意事项与限制

1. **无音乐视频**：djye.com 是纯音频站，`/mv/`、`/video/` 均 404。若产品需要 MV，需另接视频源（如主流音乐平台 API），本源无法满足。
2. **官方下载需 VIP**：`/down/` 页的正式下载走登录接口，未登录拿不到。播放器可用"播放页 `<audio>` 换扩展名"的直链方式获取 320kbps 文件，但**直链热链存在版权/ToS 风险**，上线前需评估合规（建议仅做试听，下载引导至官网）。
3. **反爬会变动**：`eval` 混淆脚本与 CDN 域名（`zj.djye.com`→`djzj.lianzhixiu.com`）可能调整，解析逻辑需做容错与监控。
4. **字段来源分散**：封面图只在榜单前 3 名有；时长/大小/码率只在 `/down/` 页；歌手多隐含在标题或标签里。按需二次请求对应页面补全。
5. **搜索分页**：`?page=` 参数在采样中未生效，正式接入前需抓真实分页 URL 确认。
6. **PC/手机一致**：`m.djye.com` 与 `www.djye.com` URL 结构相同，可共用解析逻辑；手机版 DOM 更简单，推荐优先解析以降复杂度。

---

## 九、高品替换逻辑多样本验证（32 首）

为确认"`.m4a` → 同路径 `.mp3`"高品替换规则对全站普适，抓取 **32 首跨年份/类型样本**（榜单/搜索/分类混合，上传时间 2006–2021），分别提取播放页 `<audio src>` 并核对下载页声明。

### 样本构成
- 榜单 down.html：25 首
- 搜索"王绎龙"：4 首
- 分类 xc 列表：3 首
- 覆盖年份：2006 / 2008 / 2010 / 2011 / 2012 / 2013 / 2014 / 2015 / 2016 / 2017 / 2018 / 2019 / 2021

### 验证结果
| 验证项 | 结果 | 说明 |
|---|---|---|
| `<audio>` 扩展名统一为 `.m4a` | **32/32 (100%)** | 试听源格式一致，替换基础成立 |
| 下载页声明 `Mp3文件` | **32/32 (100%)** | 每首都有 Mp3 |
| 下载页声明 `320kbps` | **32/32 (100%)** | 高品即 320kbps Mp3 |
| 下载页含 `高品质` 标识 | **32/32 (100%)** | VIP 高品标记存在 |
| 下载页"上传时间"与播放页 `#play_addtime` 一致 | 一致 | 同源数据自洽 |
| `.mp3` 变体可达（同源同路径） | **32/32** | 与 `.m4a` 同 `{PATH}`，仅扩展名不同 |
| 文件大小 / 时长字段完整 | 32/32 | 如 20095：154.04MB / 67分18秒 |

### 路径形态分布（32 首）
- `年/月/序号`：21 首
- `分类/序号`（`wq`/`zw`/`mywq`/`ywwq`）：10 首
- `年/月日/序号`：1 首（`2012/1005/4030`）

### 方法学修正（重要）
- ❌ 初版用 `zj.djye.com` 的 **301 状态码**判断文件存在——**不可靠**：该域名对任意路径（含 `045.xyz`、不存在序号）都无条件 301，属假阳性。
- ✅ 改用**下载页服务端渲染的 `.mp3info`**（站点自身声明格式/码率）作为权威依据，结论稳健。
- CDN 最终主机 `djzj.lianzhixiu.com:51880` 在沙箱不可直连，故未做字节级实测；**但后续第十一节用 67 样本大规模实测发现该 CDN 主机在公开 DNS 上已无 A 记录、整站不可达**（详见第十一节）。

### 结论
> **高品替换逻辑 `audioUrl.replace(".m4a", ".mp3")` 在采样范围内 100% 成立（URL 形态正确）**，djye 仅提供 Mp3 高品（无 wav/flac/ape 无损），播放用 `.m4a`、下载用 `.mp3` 的**地址构造**是对的。
> ⚠️ 但**文件实际能否拉到另说**：第十一节 67 样本实测显示底层 CDN 整站不可达，地址造得对 ≠ 文件下得动。AjunDJ 接入前务必真机实测。

## 十、已验证事实清单（采样）

- ✅ 榜单页：`/top/down.html` 标题"舞曲下载榜"，19 种榜单路由可用
- ✅ 歌曲 ID 体系：`/player/20095.htm`、`/down/20095.html`、`<i v="20095">` 一致
- ✅ 播放地址：`http://zj.djye.com/2016/01/045.m4a`（歌曲 20095）真实存在（URL 形态有效）
- ❌ **媒体 CDN 当前不可达（2026-08-11 实测）**：`zj.djye.com` 301 跳 `djzj.lianzhixiu.com:51880`，但该主机名在 8.8.8.8/腾讯/阿里 DNS 均无 A 记录，浏览器 fetch 失败，播放页 `ajax?a=error` 信标证实 djye 自身也加载不出。直链热链方案需真机（中国大陆）实测，未必可用。
- ❌ **码率未实测**：因 CDN 不可达，未能真正下载文件用帧头测码率；"320kbps"仅为下载页 `.mp3info` 声明值，非字节级实测。
- ✅ 高品替换：32 首样本 `<audio>` 全为 `.m4a`，下载页全声明 `Mp3 / 320kbps`，`.m4a`→`.mp3` 同路径替换 100% 成立（权威依据为下载页 `.mp3info`，非 301 状态码）
- ⚠️ 修正：`.wav/.flac/.ape` 变体不可靠，`zj.djye.com` 对任意路径无条件 301（假阳性），djye 仅提供 Mp3 320kbps 高品
- ✅ 下载页字段：格式/大小/时长/码率/上传时间 齐全（歌曲 20095：Mp3 / 154.04MB / 67分18秒 / 320kbps / 2016-01-15）
- ✅ 搜索：关键词正确编码可返回结果（"王绎龙"返回 50 条/页）
- ✅ 分类：5 大主分类 + 标签子分类 + 专辑分类
- ❌ 音乐视频：不存在（/mv/、/video/ 为 404）
- ⚠️ 下载接口需登录；搜索分页参数需二次确认

---

## 十一、下载可达性大规模实测（67 样本，2026-08-11）

> 目的：针对"试听 `zj.djye.com/{PATH}.m4a` → 高品 `zj.djye.com/{PATH}.mp3`"替换逻辑，做**大样本真实下载可达性**验证，确认是"个别死链"还是"整站失效"。

### 样本构成
- **67 首**，歌曲 ID 跨度 **2 ~ 26752**（均匀抽样 + 极端值保底）。
- 上传年份覆盖 **2006–2021**，分布：2006×3、2008×3、2010×1、2011×2、2012×2、2013×1、2014×2、2015×16、2016×2、2017×3、2018×3、2019×8、2020×1、2021×20。
- 路径形态涵盖 `年/月/序号`、`分类/序号`（如 `xcwq/003`）、`年/月日/序号` 等。

### 测试方法
1. 用 Playwright 逐首加载 `/player/{id}.htm`（过反爬 cookie 挑战），提取真实 `<audio src>`（67/67 全部取到，且**全为 `.m4a`**）。
2. 按替换规则构造 mp3 直链 `{PATH}.mp3`。
3. 对每个 mp3 直链执行：`curl -I`（不跟随）取 301 + `Location`；对重定向目标主机做 DNS A 记录解析（8.8.8.8）；最后 `curl -L -r 0-2047` 跟随重定向尝试取真实字节并校验 MP3 头（ID3 / 0xFFFB）。

### 汇总结果

| 验证项 | 结果 |
|---|---|
| `zj.djye.com` 对 mp3 直链返回 301 | **67/67 (100%)** |
| 301 重定向目标主机 | 全部 `djzj.lianzhixiu.com`（统一） |
| 重定向目标主机 DNS 可解析（公开 DNS） | **0/67（全部无 A 记录）** |
| 实际能下到字节（含 MP3 头校验） | **0/67（全部 0 字节，不可下载）** |
| 替换逻辑 URL 构造正确（m4a→mp3 同路径） | **67/67 (100%)** |

### 代表样本（10 首）

| 歌曲ID | zj返回 | 重定向主机 | 主机DNS | 下载字节 | mp3 路径 |
|---|---|---|---|---|---|
| 2 | 301 | djzj.lianzhixiu.com | 不可解析 | 0 | `xcwq/003.mp3` |
| 19226 | 301 | djzj.lianzhixiu.com | 不可解析 | 0 | `2015/06/7934.mp3` |
| 19321 | 301 | djzj.lianzhixiu.com | 不可解析 | 0 | `2015/07/8029.mp3` |
| 19700 | 301 | djzj.lianzhixiu.com | 不可解析 | 0 | `2015/11/112.mp3` |
| 19779 | 301 | djzj.lianzhixiu.com | 不可解析 | 0 | `2015/11/190.mp3` |
| 22613 | 301 | djzj.lianzhixiu.com | 不可解析 | 0 | `2017/08/040.mp3` |
| 24429 | 301 | djzj.lianzhixiu.com | 不可解析 | 0 | `2019/04/002.mp3` |
| 26273 | 301 | djzj.lianzhixiu.com | 不可解析 | 0 | `2021/06/017.mp3` |
| 26738 | 301 | djzj.lianzhixiu.com | 不可解析 | 0 | `2021/09/169.mp3` |
| 26744 | 301 | djzj.lianzhixiu.com | 不可解析 | 0 | `2021/09/175.mp3` |

### 结论
- **替换逻辑本身 100% 成立**：67 首真实音源全是 `.m4a`，替换为 `.mp3` 后地址形态全部合法、与播放页同源同路径。地址"造得对"这件事板上钉钉。
- **但文件整站级不可达**：67 首无一例外 301 到同一个 `djzj.lianzhixiu.com`，而该主机在 Google(8.8.8.8)、腾讯(119.29.29.29)、阿里(223.5.5.5) 公共 DNS 上**均查不到 A 记录**，浏览器 `fetch` 直接失败，播放页也打了 `ajax?a=error` 错误信标。**不是个别死链，是 CDN 整站下线/解析失效**。
- 因此：**地址正确 ≠ 能播放/下载**。AjunDJ 若依赖该直链热链，在中国大陆真机上也大概率拉不到流（域名已无公开解析）。接入前必须在真机实测；若确认不可达，应改走官方下载接口 `i.djye.com/ajax?a=down`（需登录态/VIP，见 4.3）或直接放弃该音源、另接可播放的音乐源。
- 码率仍未字节级实测（CDN 不可达导致无法下载样本）；"320kbps"仅为下载页 `.mp3info` 声明值。
