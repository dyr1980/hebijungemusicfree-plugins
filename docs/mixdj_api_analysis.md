# MixDJ（mixdj.cn）网站接口与页面解析 · 安卓接入全景分析

> 分析目标站：https://www.mixdj.cn （「全球DJ舞曲同步网」）
> 分析方式：真实浏览器抓包 + 源码逆向（JS / HTML 结构 / 网络请求验证）
> 适用场景：安卓 App（Jetpack Compose + Media3/ExoPlayer）接入搜索、榜单、分类、播放、高品下载
> 分析日期：2026-08-11

---

## 0. 一句话结论

MixDJ **没有对外 REST/JSON API**，它是一个 **ThinkPHP 服务端渲染（SSR）站点**。绝大多数数据（搜索结果、榜单、分类列表、歌曲详情）都直接返回 **HTML 页面**，需要在安卓端用 **Jsoup 解析 HTML** 来取数；少量交互（播放计数、波形图、收藏、下载）走 **jQuery AJAX POST** 返回 JSON。

但好消息是：**歌曲试听 MP3 是可直接流式播放的静态直链**，无需登录即可给 ExoPlayer 播放（48k 试听码率）；320k 高品下载需登录+金币走鉴权接口。

---

## 1. 基础信息与路径规律

| 项 | 值 |
|---|---|
| 主域名 | `https://www.mixdj.cn` |
| 技术栈 | ThinkPHP 3（`Application/Home/View` 目录结构）、jQuery 3.1.1、Layui、jPlayer |
| 渲染方式 | 服务端渲染 HTML 为主，Layui laypage 做 AJAX 分页 |
| 歌曲详情基地址 | `/music/info`（由隐藏域 `base_info_url` 提供） |

**核心资源路径规律（可直接拼接）：**

| 资源 | URL 规律 | 示例 |
|---|---|---|
| 歌曲试听 MP3（48k） | `/MusicData/play/{YYYYMM}/{hash}.mp3` | `/MusicData/play/202608/6a7976f733c37.mp3` |
| 歌曲封面图 | `/Uploads/Music/thumb_0/{YYYYMM}/{hash}.{ext}` | `/Uploads/Music/thumb_0/202607/6a45720624413.jpg` |
| 分类封面图 | `/Uploads/Musiccategory/thumb_0/{YYYYMM}/{hash}.{ext}` | `/Uploads/Musiccategory/thumb_0/202607/6a57687ac2468.jpg` |
| 波形图（Bowen） | `/Uploads/Bowen/{timename}/{uid}.png` | `/Uploads/Bowen/202608/56859.png` |

> 注意：封面图 hash 与 MP3 hash **互相独立**（不是同一个 hash），两者仅共享 `YYYYMM` 月份目录。封面必须**从列表/详情 HTML 里解析 `<img>` 的 `src`**，不能由 MP3 路径推算。

---

## 2. 接口总览表

| 功能 | 方法 | 地址 | 返回 | 是否需要登录 |
|---|---|---|---|---|
| 搜索/全部单曲列表 | GET | `/music/single.html` | HTML | 否 |
| 单曲排序/榜单 | GET | `/music/single/sorting/{n}.html` | HTML | 否 |
| 风格分类列表 | GET | `/music/style/id/{id}.html` | HTML | 否 |
| 套曲/歌单列表 | GET | `/music/scene.html`、`/music/sstq.html`、`/music/source.html` | HTML | 否 |
| 歌曲详情 | GET | `/music/info/id/{id}.html` | HTML | 否 |
| 套曲/歌单详情 | GET | `/album/lists/id/{id}.html` | HTML | 否 |
| 播放计数 +1 | POST | `/music/getplay.html` | JSON | 否 |
| 波形图状态 | POST | `/index/bowen.html` | JSON | 否 |
| 收藏歌曲 | POST | `/member/getfavorite.html` | JSON | 是 |
| 高品(320k)下载 | POST | 由 JS 动态设置的 `data-url`（鉴权下发） | JSON→下载地址 | **是（需金币）** |

---

## 3. 搜索接口（重点）

### 3.1 请求

```
GET https://www.mixdj.cn/music/single.html
```

| 参数 | 说明 | 示例 |
|---|---|---|
| `keyword` | 关键词（歌名/歌手，模糊匹配） | `techno` |
| `min_bmp` | BPM 下限（可选） | `120` |
| `max_bmp` | BPM 上限（可选） | `128` |
| `key` | 音调 KEY（可选，如 `6A`） | `6A` |
| `now_page` | 页码（AJAX 分页参数，从 1 起） | `2` |
| `page_size` | 每页条数（10/20/30/40/50） | `10` |
| `start-time` / `end-time` | 更新日期区间（Layui 日期选择器） | `2026-08-01` |

> 完整分页 URL 实测可用：`/music/single.html?keyword=techno&now_page=2&page_size=10`
> （首页默认 `now_page=1&page_size=10`，实测「techno」共 **481 条**结果）

### 3.2 返回解析（HTML → 数据）

每首歌是一个 `<div class="item">`，结构固定，见第 6 节。搜索结果无需登录即可拿到：**封面、歌名、BPM/KEY/金币/日期、风格、MP3 试听直链、uid**。

### 3.3 AJAX 分页机制（重要）

列表底部翻页是 **Layui laypage**，点击「下一页」本质是前端拼出
`baseurl + 'keyword=' + keyword + '&page_size=' + limit + '&now_page=' + curr` 后 **AJAX 拉取整页 HTML 并替换 `.song-list-all` 容器**。

→ 安卓端**不需要模拟这个 AJAX**：直接按 `now_page` 拼 URL 用 `GET` 请求整页 HTML 即可，参数完全一致。

---

## 4. 榜单 / 排行榜

榜单本质是「单曲列表的不同排序视图」，入口都是 `/music/single/sorting/{n}.html`：

| sorting 值 | 含义（实测推断） | 首项示例（日期） |
|---|---|---|
| `1` | 最新（按更新日期倒序） | 26.08.10 |
| `2` | 较早（按日期升序） | 19.05.31 |
| `3` | **单曲下载排行**（首页「单曲下载排行 TOP Music」指向此） | 26.06.27 |
| `4` | 另外一种排序（按 ID/热度） | 23.05.09 |
| `5` | 同最新（验证用） | 26.08.10 |

> 说明：sorting 数字与含义是**经验映射**，站点未提供文字标签；最稳妥的「排行榜」就是 `sorting/3.html`（下载榜）。

其它榜单入口：
- **歌单下载排行 / 现場歌單**：`/music/scene.html`
- **私售套曲**：`/music/sstq.html`
- **美金包（付费资源）**：`/music/source.html`
- **免费曲包**：`/news/index.html`
- **百大艺人榜单 / 各类 MIX 榜单**：以「套曲/歌单」形式存在，如 `/album/lists/id/2829.html`（BOUNCE TECHNO 百大艺人榜单）

---

## 5. 分类 / 子分类

站点分类是**两层结构**：先选「内容板块」，再选「音乐风格」。

### 5.1 一级板块（主导航）

| 板块 | 地址 |
|---|---|
| 全部單曲 | `/music/single.html` |
| 私售套曲 | `/music/sstq.html` |
| 現場歌單 | `/music/scene.html` |
| 美金包 | `/music/source.html` |
| 免費曲包 | `/news/index.html` |

### 5.2 二级风格（共 24 个，扁平列表，无更深子树）

悬浮「STYLE」菜单或任意风格分类页左侧列出，全部为 `/music/style/id/{id}.html`：

| id | 风格 | id | 风格 |
|---|---|---|---|
| 11 | Dubstep丨Drum&Bass丨DubMix | 780 | Tech House |
| 6 | Mashup丨Bootleg | 787 | Hard Trap |
| 786 | 霓虹小厅House | 665 | Minimal |
| 962 | DJ TONG BOOTLEG & MASH UP | 674 | Future Bass |
| 673 | Future House | 679 | 越南鼓/Bounce/Melbourne/Minimal/korea Bounce |
| 677 | Hard Dance | 9 | 跳舞俱乐部丨HIPHOP丨说唱 |
| 8 | Dance丨Electro pop | 781 | DJ Tools |
| 1081 | Afro House 热播 | 671 | Big Room |
| 675 | PSY Trance | 1187 | China Remix 中文歌 |
| 7 | Electro House | 497 | Deep丨Progressive丨House |
| 782 | House | 672 | Bass House |
| 1198 | Techno (Peak Time / Driving) | 676 | Trance |

> 风格分类页本身的列表结构与搜索结果**完全一致**（同样支持 `now_page`/`page_size` 分页），只是预置了风格筛选。

---

## 6. 列表项结构（解析模板 · 最关键）

下面是搜索/分类/榜单列表里**每一首歌**的真实 HTML（已脱敏简化），安卓端按这些选择器解析：

```html
<div class="item">
  <div class="group group1">
    <div class="number">01/</div>
    <div class="song-img">
      <img src="/Uploads/Music/thumb_0/202607/6a45720624413.jpg" ...>  <!-- 封面图 -->
    </div>
  </div>
  <div class="group group2">
    <div class="set set1">
      <a class="song-name play-item" title="歌手 - 歌名 (Remix)">歌手 - 歌名 (Remix)</a>
      <div class="bottom">
        <p>BPM 155 | KEY 5A | 金币 4 | 日期 26.07.02</p>  <!-- BPM/KEY/金币/更新日期 -->
      </div>
    </div>
    <div class="set set2">
      <p class="p2"><a href="/music/style/id/677.html">Hard Dance</a></p>  <!-- 风格分类 -->
    </div>
  </div>
  <div class="group group3">
    <div class="bfl">
      <!-- 播放图标：所有核心数据都在这里 -->
      <div id="jp-play_56642" data-download="/music/info/id/56642.html"
           data-type="0" class="icon" uid="56642"
           data-cate_name="Hard Dance"
           data-info=" | BPM 155 | KEY 5A | 金币 4"
           data-name="歌手 - 歌名 (Remix)"
           timename="202607" crimp="1"
           url="/MusicData/play/202607/6a4571e1c87ee.mp3"></div>
    </div>
  </div>
</div>
```

### 选择器 → 字段 映射

| 你要的字段 | 选择器 | 取值方式 |
|---|---|---|
| 歌曲 uid | `.bfl .icon` 的 `uid` | 直接属性 |
| **播放/试听直链** | `.bfl .icon` 的 `url` | 直接属性（MP3 地址） |
| **歌名（曲名）** | `.group2 .song-name` 文本；或 `data-name` | 文本 / 属性 |
| **歌手** | 由歌名按 ` - ` 分割取前半段 | 站点**无独立歌手字段**，歌手嵌在歌名里 |
| **封面图** | `.group1 .song-img img` 的 `src` | 直接属性（需拼主域） |
| **风格/分类** | `.group2 .set2 .p2 a` 文本 + `href` | 文本 / id |
| BPM | `.group2 .bottom p` 文本中 `BPM xxx` | 正则提取 |
| KEY（音调） | 同上 `KEY xxx` | 正则提取 |
| 金币 | 同上 `金币 x` | 正则提取 |
| **更新时间** | 同上 `日期 yy.mm.dd` | 正则提取（转成 yyyy-mm-dd） |
| 详情页地址 | `.bfl .icon` 的 `data-download` | 直接属性 |

> **歌手 vs 歌名**：MixDJ 没有单独「歌手」字段。歌名格式统一为 `歌手A & 歌手B - 曲名 (Remix) ...`。安卓端建议按第一个 ` - ` 切分：左边=歌手，右边=标题名（曲名）。

---

## 7. 歌曲详情页（全字段）

`GET /music/info/id/{uid}.html`，实测字段（`uid=56859` 示例）：

```html
<div class="songs-infos">
  <h3> Fatboy Slim & Riva Starr - Eat Sleep Rave Repeat (Mike Renza Remix) EXTENDED 推荐单曲！</h3>
  <ul>
    <li><strong>歌曲编号：</strong> 56859</li>
    <li><strong>歌曲风格：</strong> House</li>
    <li><strong>歌曲名称：</strong> Fatboy Slim & Riva Starr - Eat Sleep Rave Repeat ...</li>
    <li><strong>歌曲BPM：</strong> 130</li>
    <li><strong>歌曲音调：</strong> 6A</li>
    <li><strong>歌曲金币：</strong> 6</li>
    <li><strong>更新日期：</strong> 2026-08-10</li>
    <li><strong>歌曲音质：</strong> 320 k</li>      <!-- 高品(下载)码率 -->
    <li><strong>试听音质：</strong> 48 k</li>        <!-- 试听(播放)码率 -->
    <li><strong>试听次数：</strong> 345</li>
    <li><strong>上传作者：</strong> 站长</li>
    <li><strong>下载次数：</strong> 181</li>
    <li><strong>收藏次数：</strong> 0</li>
  </ul>
  <div class="songs-player">
    <img class="oscillogram" src="https://www.mixdj.cn/Uploads/Bowen/202608/56859.png">  <!-- 波形图 -->
  </div>
</div>
```

| 你要的字段 | 来源 |
|---|---|
| 歌曲编号(uid) | `歌曲编号` |
| **标题名(曲名)** | `<h3>` 或 `歌曲名称` |
| 风格/分类 | `歌曲风格` |
| BPM / KEY / 金币 | 对应 `<li>` |
| **更新时间** | `更新日期`（已是 `yyyy-mm-dd`） |
| **高品码率** | `歌曲音质`（如 320 k）→ 对应下载品质 |
| **试听码率** | `试听音质`（48 k）→ 对应试听直链品质 |
| 试听/下载/收藏次数 | 对应 `<li>` |
| 波形图 | `img.oscillogram` 的 `src` |

> **文件大小**：详情页**不直接给文件大小文本**。做法二选一：
> 1. 请求 MP3 直链读 `Content-Length` 响应头（最直接，见第 8 节）；
> 2. 用「码率 × 时长」估算（320k → 约 2.4MB/分钟；48k → 约 0.36MB/分钟）。
>
> **歌曲时长**：详情页播放器初始显示 `00:00`，真实时长**不在 HTML 里**，由 jPlayer 读取音频元数据得到。安卓端用 `MediaMetadataRetriever` 对 MP3 直链取时长即可。

---

## 8. 播放 / 试听（MP3 直链，可直接播）

- **试听直链**（48k）：列表项 `url` 属性，如
  `https://www.mixdj.cn/MusicData/play/202608/6a7976f733c37.mp3`
- 实测响应：`Content-Type: audio/mpeg`，`Content-Length: 2362718`（≈2.25MB），**支持 Range 请求（206）**，`accept-ranges: bytes`，**无需 Referer / 无防盗链**。
- → 直接丢给 **ExoPlayer / Media3** 即可流式播放、拖动进度。

**播放计数接口**（点击播放时站点会调，可选对接）：
```
POST /music/getplay.html
body: id=56859
响应: {"state":1,"num":"347"}   // 返回最新试听次数
```

**波形图接口**（可选，做播放进度波形 UI 用）：
```
POST /index/bowen.html
body: id=56859
响应: {"status":1}
实际波形 PNG: /Uploads/Bowen/{timename}/{uid}.png
```

---

## 9. 高品链接（320k 下载）· 限制说明

- 详情页标注 **歌曲音质 320 k** 即高品；但**下载不是静态 URL**，且**无法从试听链接推导**。
- **关键结论（已实测验证）**：高品 320k 链接**不能**由试听直链 `/MusicData/play/{YYYYMM}/{hash}.mp3`（48k 转码文件）通过"换目录 / 换后缀"推导。证据如下：
  1. **穷举 15 种同 hash 路径规律全部 404**（带 / 不带 `Referer` 均如此）：`/MusicData/download/...`、`/MusicData/320/...`、`/MusicData/high/...`、`/MusicData/flac/...`、`/MusicData/wav/...`、`/MusicData/source/...`、`/MusicData/play/320/...`、`..._320.mp3`、`320_....mp3`、`..._192.mp3` 等 —— 试听与高品是**不同文件、不同 hash**，路径树根本不重合。
  2. 下载按钮 `#downBtn` 的真实属性为 `data-url=""`（HTML 中空白）、`href="javascript:;"` —— 地址是点击后由 JS **动态填充**的，并非写死的静态文件。
  3. 未登录态实际点击「我要下载」：点击前后 `data-url` 始终为空，页面含登录引导文案，被登录态拦截，**不发任何下载请求**（localStorage 无 `musicuid`）。
  4. `music.box.js` 中仅有播放计数 `$.post(playUrl)`、波形 `$.post(crbowen)` 两个 AJAX；下载地址**无静态端点**，须经登录 + 扣金币后由内部接口返回带签名/token 的临时下载 URL 才塞入 `data-url`。
- 前端机制（从源码还原）：播放器栏 `#downBtn` 的 `data-url` 由 JS 在已登录且有金币时动态设置；下载按钮点击后走
  `$.ajax({type:"POST", url: data-url, data:{id, type:0}, dataType:"json"})`，返回 JSON 后再拿到真实下载地址。
- **结论**：320k 高品下载**需要登录 + 消耗金币**，无公开静态直链，**无法从试听 URL 推导**。安卓端若要支持高品下载，必须：
  1. 实现站点登录（账号体系在 `/user/`，注册在 `/user/doreg.html`）；
  2. 复刻该 AJAX 调用拿下载地址；
  3. 遵守金币/版权规则。

> 对纯播放型 DJ App，建议先用 48k 试听直链做流畅播放；高品下载作为「登录后增值能力」后续接入。

---

## 10. 音乐视频（重要结论）

**MixDJ 没有独立「音乐视频」流媒体板块**。核查结果：
- 探测 `/music/video.html`、`/video.html`、`/music/mv.html` 等均 **404**；
- 专辑/歌单详情页**无 `<video>` 标签**；
- 「视频」仅作为**部分套曲的附属下载素材**出现（如歌单标题「含开场视频素材」），是随套曲打包的可下载文件，不提供在线播放。

→ 安卓端若要做「音乐视频」模块，当前站点的可行路线是：解析带「视频素材」描述的套曲详情，提供**该套曲的下载入口**（同样受登录/金币限制），而非视频流播放。

---

## 11. 安卓集成建议（落地架构）

```
┌─────────────┐   Jsoup (HTML 解析)   ┌──────────────────┐
│  MixdjRepository  │ ───────────────► │  mixdj.cn 各 HTML 页 │
│  (Kotlin 协程)    │ ◄─────────────── │  (搜索/榜单/分类/详情)│
└─────────────┘   映射为数据类        └──────────────────┘
       │
       ├─► ExoPlayer / Media3  ◄── 直连 /MusicData/play/*.mp3 (48k 试听)
       ├─► Coil/Glide          ◄── 加载 /Uploads/Music/thumb_0/*.jpg (封面)
       └─► Retrofit (JSON)     ◄── /music/getplay.html、/index/bowen.html、登录下载
```

**推荐技术选型：**
- **网络/解析**：`Jsoup`（解析 HTML 列表与详情） + `Retrofit`（仅用于那几个 JSON 接口）
- **播放**：`androidx.media3.exoplayer`（直连 MP3 直链，支持 Range 拖动）
- **图片**：`Coil` 加载封面/波形图
- **并发**：`Kotlin 协程 + Flow`

**数据模型（Kotlin 草稿）：**
```kotlin
data class Track(
    val uid: String,
    val title: String,          // 曲名（歌名按 " - " 取右半）
    val artist: String,         // 歌手（歌名按 " - " 取左半）
    val style: String,          // 风格/分类
    val styleId: String,        // 风格 id
    val coverUrl: String,       // 封面图
    val previewUrl: String,     // 48k 试听直链
    val bpm: Int?,
    val key: String?,
    val coins: Int?,
    val updateDate: String,     // 更新时间
    val quality: String,        // 高品码率，如 "320 k"
    val playCount: Int?,
    val downloadCount: Int?,
    val bowenUrl: String        // 波形图
)
```

**分页加载示例（搜索）：**
```kotlin
suspend fun search(keyword: String, page: Int, size: Int = 20): List<Track> {
    val url = "https://www.mixdj.cn/music/single.html" +
              "?keyword=${encode(keyword)}&now_page=$page&page_size=$size"
    val doc = Jsoup.connect(url).get()
    return doc.select(".song-list-all .item").map { parseTrack(it) }
}
```

**播放（ExoPlayer）：**
```kotlin
val mediaItem = MediaItem.fromUri("https://www.mixdj.cn/MusicData/play/202608/6a7976f733c37.mp3")
exoPlayer.setMediaItem(mediaItem)
exoPlayer.prepare(); exoPlayer.play()
```

---

## 12. 注意事项 / 踩坑点

1. **无 JSON 列表 API**：所有列表/搜索/榜单/详情都是 HTML，必须 Jsoup 解析，且选择器可能随改版变动，需做容错。
2. **歌手无独立字段**：歌手嵌在歌名 `歌手 - 曲名` 中，按首个 ` - ` 切分；注意歌名里可能含 ` - `（如 `A - B (C - D Remix)`），切分只取第一个。
3. **封面与 MP3 不同 hash**：封面只能从 HTML `<img src>` 取，不能由 MP3 路径推算。
4. **详情页无封面**：详情页只显示波形图，不显示封面；封面请从列表/搜索结果里带过来缓存。
5. **时长/大小不在 HTML**：时长用 `MediaMetadataRetriever` 取；大小读 MP3 `Content-Length` 或按码率估算。
6. **高品下载需登录+金币**：无静态直链，纯播放用 48k 试听即可。
7. **无音乐视频流**：视频仅作套曲附属下载素材，不提供在线播放。
8. **UA/频率**：虽试听直链无 Referer 校验，但高频抓取建议带正常 `User-Agent` 并控制并发，避免被限流。
9. **日期格式**：列表里是 `yy.mm.dd`（如 `26.07.02`），详情页是 `yyyy-mm-dd`，解析时注意转换。

---

## 13. 实测验证记录（真实样本）

| 验证项 | 结果 |
|---|---|
| 试听 MP3 直链可访问 | `200 audio/mpeg` 2,362,718 字节，支持 Range(206) |
| 封面图 URL | `200`（如 `/Uploads/Music/thumb_0/202607/6a45720624413.jpg`） |
| 波形图 URL | `200`（`/Uploads/Bowen/202608/56859.png`） |
| `POST /music/getplay.html` | `{"state":1,"num":"347"}` |
| `POST /index/bowen.html` | `{"status":1}` |
| 搜索分页 `now_page=2` | 生效，首项变为第 11 首 |
| 风格分类总数 | 24 个 |
| `sorting/3` 下载榜 | 首页「单曲下载排行 TOP Music」指向 |
| 高品静态下载路径 | `404`（需鉴权） |
| 音乐视频独立页 | `404`（无独立视频流） |

---

*本分析基于 2026-08-11 对 mixdj.cn 的真实抓包与源码逆向，接口路径与选择器可能随站点改版变化，接入时建议对关键选择器加监控与降级。*
