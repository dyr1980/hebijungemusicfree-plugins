# DJMusic 音源接口文档

> 版本：2026-08-09
> 适用工程：`_shared/project_src`（包名 `com.djmusic.ajun`）
> 适用范围：App 内置全部 18 个音乐音源的接口契约与端点清单。
> 说明：本文档依据源码 `app/src/main/java/com/ajun/dj/data/remote/` 下各 `MusicSource` 实现逐条整理，端点均为 App 实际请求地址。

---

## 1. 统一接口契约（MusicSource）

所有音源实现同一接口 `MusicSource`（`MusicSource.kt`），上层 Repository / 播放 / 下载均通过当前活跃音源取数，新增站点无需改动上层业务。核心方法：

| 方法 | 说明 | 返回 |
|---|---|---|
| `getHomeList(page)` | 首页「最新」流（单曲页） | `List<Song>` |
| `getSongList(categoryId, type, page, sort)` | 曲库分类列表（`categoryId` 为站点分类标识，`type` 0=单曲/1=串烧） | `List<Song>` |
| `getSearchResults(keyword, page)` | 搜索结果 | `List<Song>` |
| `getRankList(rankId)` | 单个榜单歌曲 | `List<Song>` |
| `getRanks()` | 全部榜单（榜单页 Tab） | `List<Rank>` |
| `getGenres()` | 全部曲库分类（曲库页筛选） | `List<Genre>` |
| `getSongDetail(id)` | 歌曲详情（可选，默认 null） | `Song?` |
| `resolveAudioUrl(songId, quality)` | 解析可播放真实音频地址（交给 ExoPlayer） | `String` |
| `getLrc(songId)` | LRC 歌词文本（无则 null） | `String?` |

> 仅 djyule 额外实现了扩展能力：`getHotKeys()`（热搜词）、`getPlaylists()`、`getPlaylistSongs()`、`getAuthors()`、`getAuthorSongs()`、`getStations()`、`getStationSongs()`、`searchVideos()`、`getVideoDetail()`、`getVideoDownloadInfo()`、`getVideoDownloadUrl()`、`getExperienceSongs()`（体验精选 320k）。其余音源这些方法走默认空实现。

### 音质档位（全局常量）

`Constants.kt` 定义三档：`low` / `standard` / `high`，UI 显示「低品 / 标准音质 / 高品」。每个音源的「受支持档位 + 默认档」由 `SourceManager.SOURCE_QUALITY` 单表管理（播放与下载共用同一档案）：

| 音源（type） | 支持音质 | 默认 |
|---|---|---|
| djf5 | standard, high | standard |
| djyule | low, high | low |
| dj93 | low | low |
| bobidj | low | low |
| bibdj | standard | standard |
| hy57 | low | low |
| pgdjz | low | low |
| 55y | standard, high | high |
| dj14 | standard | standard |
| djuu | low | low |
| ddddj | standard, high | standard |
| xsdj123 | standard | standard |
| qqdjs | standard, high | high |
| xxedm | standard, high | high |
| 172mix | standard | standard |
| 36dj | standard | standard |
| spacedj | standard | standard |
| djyyy | standard | standard |

> 切换音源时，`SourceManager` 依据「省流量模式」自动对齐播放/下载音质（开=最低档，关=最高档），保证「播放音质 / 下载音质」弹窗始终只显示该源受支持的档。

### 分享链接解析（「我的 → 解析」）

`SourceManager.parseShareLink(url)` 按 host 识别音源并提取歌曲 id（正则见 `SHARE_LINK_SPECS`），再调用对应源的 `getSongDetail(id)`。各源识别规格如下（host 支持任意子域后缀匹配）：

| 音源 | host | id 提取正则 |
|---|---|---|
| djf5 | djf5.com | `play-(\d+)\.html` |
| djyule | djyule.com / sj.djyule.com | `player(?:\.asp|/)\?id=(\d+)` |
| dj93 | dj93.com | `play/(\d+)\.html` |
| bobidj | bobidj.com / www / m | `bobidj\.com/play/(?:index/music\?id=)?(\d+)` |
| bibdj | bibdj.com | `/play/(\d+)\.html` |
| hy57 | hy57.com | `/p/(\d+)\.html` |
| pgdjz | pgdjz.com | `/dance/(\d+)\.html` |
| 55y | 55y.top / dj92.cc | `(?:(?:song|play|detail)/\|id=)(\d+)` |
| dj14 | dj14.com | `/music/(\d+)\.html` |
| djuu | www/m/android.djuu.com | `play/(\d+)\.html` |
| ddddj | ddddj.com | `/play/(\d+)` |
| xsdj123 | xsdj123.com | `/music/info-(\d+)` |
| qqdjs | qqdjs.com | `/play/(\d+)\.html` |
| xxedm | xxedm.com | `/thread-(\d+)\.htm` |
| 172mix | 172mix.com | `/play/(\d+)` |
| 36dj | 36dj.com | `/play/(\d+)\.html` |
| spacedj | spacedj.cn | `music_detail/(\d+)` |
| djyyy | djyyy.com | `play/(\d+)\.html` |

---

## 2. 各音源接口明细

> 约定：`BASE` 为站点根；`{id}` 为歌曲/分类数字标识；`{page}` 为页码（多数从 1 起）。

### 2.1 DJ舞曲 (djf5)
- 实现：`DjF5Api.kt` ｜ BASE = `https://www.djf5.com`
- 首页最新：`GET {BASE}/dance/opt/new`
- 曲库分类：`GET {BASE}/dance/list-id-{categoryId}-{page}.html`
- 搜索：`GET {BASE}/dance/search?key={keyword}&page={page}`
- 榜单：`GET {BASE}/dance/opt/{rankType}`
- 歌曲详情：`GET {BASE}/dance/play-{songId}.html`
- 音频直链（播放）：`GET {BASE}/dance/djuq/playurl/{songId}`（返回真实地址；Referer=`{BASE}/`）
- 音频直链（下载）：`GET {BASE}/dance/down/gateway/{songId}`
- 封面兜底：`{BASE}/picdata/dance/images/{id}.jpg`
- 音质：standard + high

### 2.2 DJ娱乐网 (djyule)
- 实现：`DjYuleApi.kt` ｜ 主站 BASE=`https://www.djyule.com`，移动站 `https://sj.djyule.com`
- 接口集群（app 域名）：
  - `https://appso.djyule.com` — 歌曲/搜索/歌单/作者/电台主接口（`load_play_relevant.asp`、`load_hotkey.asp` 热搜、`video_play.asp` 视频）
  - `https://appsc.djyule.com` — 收藏/盒子
  - `https://apphy2.djyule.com` — 下载相关（`video_download_load.asp`、`video_download.asp`）
  - `https://appxml.djyule.com` — 配置/榜单/歌单 XML（`{key}_1.xml`、`tiyan320.xml`）
  - `http://img.djyule.com` — 封面/分类图标；`http://userimg.djyule.com` — 作者头像
- 音频解析：`GET {BASE}/musicUrl.asp?id={songId}`（须带 `Referer: {BASE}/player/?id={songId}`），响应 `musicUrl='https://w64-NN.djyule.com/'` → 映射到 `http://s64-NN.jyw8.com`
- 播放页：`{BASE}/player/?id={songId}`（移动站 `https://sj.djyule.com/player.asp?id={songId}`）
- 歌词：`GET {BASE}/showLRC.asp?id={songId}`
- 热搜词：`GET appso.djyule.com/load_hotkey.asp`（依赖 VIP 参数）
- 音质：low + high（含体验精选 320k 直链，PLAYURL 为完整 URL 可直接播放）

### 2.3 前沿DJ (dj93)
- 实现：`Dj93Source.kt` ｜ BASE=`https://www.dj93.com`，API=`https://www.dj93.com/api/mp`，IMG=`https://img.dj93.com`
- 首页最新：`GET {API}/songs/latest?size=20`（page≤1）｜ 分页 `GET {API}/songs?page={page}&size=20`
- 曲库分类：`GET {API}/songs/songs-by-subgenre/{genreId}?page={page}&size=20`（有分类）｜ 否则 `songs?page=`
- 搜索：`GET {API}/songs/search?keyword={keyword}&page={page}&size=20`
- 榜单列表：`GET {API}/songs/top-rankings` ｜ 榜单歌曲：`GET {API}/songs/top-ranking-songs?rank_id={rankId}&page=1&size=30`
- 分类列表：`GET {API}/songs/genres`
- 歌曲详情：`GET {API}/songs/{id}`
- 音频直链：`GET {API}/songs/play-url?id={songId}&quality=96`（仅 96 → low）
- 扩展：专辑 `albums/hot`、`albums/detail/{id}`；作者 `artists/hot`、`artists/{id}`；电台 `radio/hot`、`radio/{id}`
- 音质：low only

### 2.4 波比DJ (bobidj)
- 实现：`BobiDjSource.kt` ｜ BASE=`https://m.bobidj.com`，PLAY_BASE=`https://www.bobidj.com`
- 首页/榜单：`GET {BASE}/process/newlist?musicid={ANCHOR}`
- 曲库分类：`GET {BASE}/process/{path}?musicid={ANCHOR}`
- 搜索：`GET {BASE}/search/so?skeys={keyword}&byid=0&page={page}`
- 分类名：`GET {BASE}/process/getmusicname?musicid={ids}`
- 歌曲详情：`GET {BASE}/play/index/music?id={id}`
- 音频直链：`AES` 解密播放页内联密文 → `https://p2.bobidj.com/{date}/{BBxxxx}_{hash}.mp4?auth_key=...`（AAC ~64k 试听，免登录）
- 音质：low only

### 2.5 宝贝DJ (bibdj)
- 实现：`BibiDjSource.kt` ｜ BASE=`https://www.bibdj.com`
- 曲库分类：`GET {BASE}/sort/{seg}-0-0-0-{page}.html`
- 搜索：`GET {BASE}/search?key={keyword}&page={page}`
- 歌曲详情：`GET {BASE}/play/{id}.html`
- 音频直链：`GET {BASE}/play/{songId}.html`（SSR 内联 `var playurl` → `down.bibdj.com`）；另有 `GET {BASE}/play/ajax/gmap`
- 音质：standard only

### 2.6 黑色音频DJ (hy57)
- 实现：`Hy57Source.kt` ｜ BASE=`https://www.hy57.com`
- 首页推荐：`GET {BASE}/index.php/dance/vvvdj/getreco` ｜ 首页兜底 `GET {BASE}/index.html`
- 曲库分类：`GET {BASE}/p/list-id-{catId}-1.html`
- 搜索：`GET {BASE}/dance/search?key={keyword}`
- 歌曲详情：`GET {BASE}/p/{id}.html`
- 音频直链：`GET {BASE}/index.php/dance/play/url/jp`
- 音质：low only

### 2.7 皮狗DJ站 (pgdjz)
- 实现：`PgdjzSource.kt` ｜ BASE=`https://www.pgdjz.com`，WAP=`https://wap.pgdjz.com`
- 首页：`GET {BASE}/`
- 曲库分类：`GET {WAP}/dance/lists/{key}.html`（`key`=hits/down 等）
- 搜索：`GET {BASE}/dance/search.html?key={keyword}`
- 榜单：`GET {BASE}/`（rankId==1 首页），其余 emptyList
- 歌曲详情：`GET {BASE}/dance/{id}.html`
- 音频直链：`GET {BASE}/ajax/danceplayer?id={songId}`
- 音质：low only

### 2.8 55音乐 (55y)
- 实现：`Five5ySource.kt` ｜ BASE=`https://www.55y.top`，API=`https://www.55y.top/pc-api`
- 全部业务接口 **POST** 到 `{API}/{module}/{action}`（带 `Referer: {BASE}/`）
- 音频直链：响应 `row.downurl`（high）/ `row.playurl`（standard），服务端返明文 `http://`（Android 9+ 需 cleartext 放行）
- 分支站点：`dj92.cc/djplay/{id}.html` 与 `55y.top` 共用歌曲 id
- 音质：standard + high（默认 high）

### 2.9 DJ14水晶舞曲 (dj14)
- 实现：`Dj14Source.kt` ｜ BASE=`https://dj14.com`
- 首页：`GET {BASE}/`
- 曲库分类：`GET {BASE}/genre/c{catId}/0-0-0-{page}-1.html`
- 搜索：`GET {BASE}/search?keys={keyword}`
- 榜单：首页 `{BASE}/` ｜ 热门 `{BASE}/top/mixes`
- 歌曲详情：`GET {BASE}/music/{id}.html`
- 音频直链：`GET {BASE}/music/{songId}.html` 提取 `mp3` 字段 → `https://cdn.dj1198.com/uploads/play/128k/<YYYYMMDD>/<32hex>.mp3`（须带 `Referer: http://dj14.com/`）
- 音质：standard only

### 2.10 DJ叮叮 (djuu)
- 实现：`DjuuSource.kt` ｜ BASE=`https://android.djuu.com/app/v1`（逆向 app 接口）
- 首页/榜单：`GET {BASE}/music/playRankings?rankType=1&page={page}&row=30`
- 曲库分类：`GET {BASE}/music/playRankings?rankType=1&categorie_pid={pid}&page={page}&row=30`
- 搜索：`GET {BASE}/music/search?keywords={keyword}&page={page}&row=30`
- 榜单（按类型）：`GET {BASE}/music/playRankings?rankType={rt}&page=1&row=30`
- 分类字典：`GET {BASE}/dict`
- 歌曲详情：`GET https://m.djuu.com/play/{id}.html`（SSR，含歌手名）
- 音频直链：`GET {BASE}/music/listen?id={songId}`
- 音质：low only

### 2.11 清纯DJ (ddddj)
- 实现：`DdddjSource.kt` ｜ BASE=`https://www.ddddj.com`
- 首页：`GET {BASE}/`
- 曲库分类：`GET {BASE}/genre/{slug}/0-0-0-0-{page}.html` ｜ 热门 `/top/{slug}.html` ｜ 精选 `/ranks/sole/{slug}-0-0-0-{page}.html`
- 搜索：`GET {BASE}/search.html?keys={keyword}&page={page}`（表单 `/search/selbox/`）
- 歌曲详情：`GET {BASE}/play/{id}`
- 音频直链：**POST** `{BASE}/ajax/wap/play`（表单，Referer=`{BASE}/play/{id}`）→ 返回 url
- 音质：standard + high

### 2.12 先上DJ (xsdj123)
- 实现：`XsdjSource.kt` ｜ BASE=`https://www.xsdj123.com`
- 首页：`GET {BASE}/music/list-1-0?type=new&page={page}`
- 曲库分类：`GET {BASE}/music/list-{catId}-0?type=new&page={page}`
- 搜索：`GET {BASE}/music/find?keyword={keyword}&page={page}`（亦支持 `type=` 参数）
- 歌曲详情：`GET {BASE}/music/info-{id}`
- 音频直链：详情页内联 `<audio src="https://listen.dj1234.com/...aac">`（Referer 由 SourceHeaders 自动注入）
- 音质：standard only

### 2.13 QQDJ舞曲 (qqdjs)
- 实现：`QqdjsSource.kt` ｜ BASE=`https://www.qqdjs.com`
- 首页：`GET {BASE}/genre/xccs/0-0-0-0-{page}.html`
- 曲库分类：`GET {BASE}/genre/{slug}/0-0-0-0-{page}.html`
- 搜索：page≤1 `GET {BASE}/search/?keys={kw}` ｜ 否则 `GET {BASE}/search/page/{page}.html?keys={kw}&cid=0&type=1`
- 榜单（sole）：`GET {BASE}/ranks/sole/0-0-0-0-{page}.html`
- 歌曲详情：`GET {BASE}/play/{id}.html`
- 音频直链：`GET {BASE}/play/{songId}.html` → 去 `?t=&sign=&kid=` 参数并改 `http://`（保留 `/play/` 路径）
- 音质：standard + high

### 2.14 小熊电音 (xxedm)
- 实现：`XxedmSource.kt` ｜ BASE=`http://xxedm.com`（**注意为 HTTP**）
- 首页/曲库：`GET {BASE}/forum-{fid}.htm`（分页 `forum-{fid}-{page}.htm`）
- 搜索：`GET {BASE}/search-{kw}.htm`
- 歌曲详情：`GET {BASE}/thread-{id}.htm`
- 音频直链：`<source src="https://pan.urlkj.com/play/...mp3">`（回退 `<audio src>`），免登录、无 Referer 防盗链
- 音质：standard + high

### 2.15 172Mix 舞曲网 (172mix)
- 实现：`Mix172Source.kt` ｜ BASE=`https://www.172mix.com`
- 曲库分类：`GET {BASE}/genre/{code}?page={page}`
- 搜索：`GET {BASE}/searches?query={keyword}&page={page}`
- 榜单：`GET {BASE}/ranks/sole?page=1`
- 歌曲详情：`GET {BASE}/play/{id}`
- 音频直链：`GET {BASE}/play/{songId}` 提取 jPlayer `setMedia({ m4a: "https://mp3.172mix.com/mp3/{date}/{hash}.m4a" })`
- 音质：standard only

### 2.16 36DJ 舞曲网 (36dj)
- 实现：`Dj36Source.kt` ｜ BASE=`https://www.36dj.com`，CDN=`https://tn1.72djapp.cn/`
- 首页：`GET {BASE}/html/new/`（分页 `index_{p}.html`）
- 曲库分类：`GET {BASE}/{code}/`（分页 `index_{p}.html`）
- 榜单：`/html/taldown.html?action=1`、`/html/month.html?action=down`、`/html/diggtop.html?action=hot`
- 搜索：**POST** `{BASE}/e/search/?searchget=1`（表单）→ `GET {BASE}/e/search/result/index.php?page={p}&searchid={lastSearchId}`
- 歌曲详情：`GET {BASE}/play/{id}.html`
- 音频直链：`GET {BASE}/play/{songId}.html` 提取 `playurl` → 拼接 `{CDN}{playurl}`（m4a，支持 Range）
- 音质：standard only

### 2.17 SpaceDJ 音乐网 (spacedj)
- 实现：`SpacedjSource.kt` ｜ BASE=`https://www.spacedj.cn`
- 首页：`GET {BASE}/music/cycle?page={p}`
- 曲库分类：`GET {BASE}/music/cycle?style_name={code}&page={p}`（`code` 取自内置 `CATEGORIES`：早场/中场/主场/后场舞曲）
- 搜索：`GET {BASE}/search?keywords={keyword}&type=single&page={p}`
- 歌曲详情：`GET {BASE}/music_detail/{id}`
- 音频直链：`GET {BASE}/music_detail/{songId}` 提取 `.mp3` → 相对路径 `/uploads/attachment/...` 补 `{BASE}`；`downloads.spacedj.cn` 已是绝对 URL
- 音质：standard only
- 注意：当前测试环境（宿主机/模拟器）对 `spacedj.cn` 的 443 连接超时（服务端不可达），列表为空属环境限制，非代码缺陷。

### 2.18 DJ耶耶耶网 (djyyy)
- 实现：`DjyyySource.kt` ｜ BASE=`https://djyyy.com`
- 首页：`GET {BASE}/`（分页 `{BASE}/x_{p}.html`）
- 曲库分类：`GET {BASE}/{code}/x_{p}.html`（`code` 取自内置 `CATEGORIES`：hanmai=喊麦 / chuanshao=串烧 / manyao=慢摇 / jiaoyi=交友 / jinbao=金包 / mange=曼歌）；默认 `{BASE}/x_{p}.html`
- 搜索：`GET {BASE}/search.php?ac=dj&key={keyword}&page={p}`
- 歌曲详情：`GET {BASE}/play/{id}.html` ｜ 关联列表 `GET {BASE}/ajax/?ac=play_musiclist&id={id}`
- 音频直链：`GET {BASE}/play/{songId}.html` 提取 token → `GET {BASE}/t/{token}`（带 Referer）解析 `playurl='https://oo.haiqu.vip/...m4a'`（`\/`→`/` 反转义，再解码 `\uXXXX` Unicode 转义得到含中文直链）
- 音质：standard only

---

## 3. 各音源「按音质」获取音频直链逻辑（resolveAudioUrl 详解）

`resolveAudioUrl(songId, quality)` 是上层播放/下载取链的唯一入口，`quality ∈ {low, standard, high}`。各音源实现分两大类：

### 3.1 单音质源（忽略 quality，直接返回唯一直链）

这类源站点本身只提供一种规格，无论 `quality` 传入什么，**统一返回同一真实音频地址**（UI 弹窗也只会显示该源 `SOURCE_QUALITY` 中声明的那一档，不会让用户选到不存在的档）。实现上 `quality` 参数被直接忽略：

| 音源 | 唯一音质 | 取链方式（已实现） | 备注 |
|---|---|---|---|
| dj93 | low(96K) | `GET {API}/songs/play-url?id={id}&quality=96` → `play_url`（列表已预填，零二次解析） | 硬编码 `quality=96`，无 high 入口 |
| bobidj | low(AAC 试听) | AES 解密播放页内联密文 → `https://p2.bobidj.com/{date}/{BBxxxx}_{hash}.mp4?auth_key=...` | 免登录试听 |
| bibdj | standard | `GET {BASE}/play/{id}.html` 提取 `var playurl`（→`down.bibdj.com`）；失败兜底 `GET {BASE}/play/ajax/gmap` | 双路径兜底 |
| hy57 | low(标准 m4a) | `GET {BASE}/index.php/dance/play/url/jp` → JSON `playurl` | 注释标 low，站点单规格 |
| pgdjz | low(m4a) | `GET {BASE}/ajax/danceplayer?id={id}` → JSON `playurl` | 直链含目录+文件名，无法由 id 反推 |
| dj14 | standard(128K) | `GET {BASE}/music/{id}.html` 提取 `mp3` 字段 → `https://cdn.dj1198.com/uploads/play/128k/...`（Referer=`http://dj14.com/`） | 无高品 |
| djuu | low(m4a) | `GET {BASE}/music/listen?id={id}` → listenurl（无 302、无 Referer 校验） | 逆向 app 接口 |
| 172mix | standard | `GET {BASE}/play/{id}` 提取 jPlayer `setMedia({m4a:"https://mp3.172mix.com/mp3/{date}/{hash}.m4a"})` | 免登录、支持 Range |
| 36dj | standard | `GET {BASE}/play/{id}.html` 提取 `var info={"playurl":...}` → 拼接 `{CDN}{playurl}`（`tn1.72djapp.cn`，m4a） | 支持 Range |
| spacedj | standard | `GET {BASE}/music_detail/{id}` 提取 `.mp3` → 相对 `/uploads/attachment/...` 补 BASE | 当前环境不可达（站点问题） |
| djyyy | standard | `GET {BASE}/play/{id}.html` 提取 token → `GET {BASE}/t/{token}`（Referer）→ 解析 `playurl='https://oo.haiqu.vip/...m4a'`（反转义+解码 Unicode） | 无高品 |

### 3.2 多音质分叉源（按 quality 选择不同取链/变换）

以下 7 个源在 `resolveAudioUrl` 内对 `quality` 做了真实分叉。**UI「播放音质 / 下载音质」弹窗显示的档位，正是 `SOURCE_QUALITY[type].supported` 所声明的集合**（见 §1 音质表），取链逻辑与之一一对应。

#### 3.2.1 DJ舞曲 (djf5) — standard + high
- 入口统一：`GET {BASE}/dance/djuq/playurl/{id}`（`Referer={BASE}/`）→ **302 重定向**到 `pan.urlkj.com` 真实直链（路径含 `/play//`），用 HEAD 拿 Location（不跟随），失败指数退避重试。
- **标准(128K)**：`resolveStandardAudioUrl` 直接采用该 `/play//` 直链。
- **高品(320K)**：`resolveHighAudioUrl` 把标准直链路径中 **`/play//` 精确替换为 `/down//`**（用精确路径替换，避免误命中 URL 中其它 play 子串）。
- 低品：djf5 无此档，`SOURCE_QUALITY` 未声明，UI 不会出现；即便传入也走 `else` 分支返回标准直链（安全回退）。

#### 3.2.2 DJ娱乐网 (djyule) — low + high（无 128K 档）
- 先取 PLAYURL **相对路径** `path`（优先列表/搜索解析写入的内存缓存 `playPathCache`；miss 则兜底抓 `{MOBILE}/player.asp?id={id}` 或 `{BASE}/player/?id={id}` 提取）。
- 按音质拼接 CDN 前缀：
  - **高品(320K)** `QUALITY_HIGH`：`probe320k(path)` —— 在 320K 主机池（`jrid28/fuuw27/ngrw1/st320.jyw8.com` 等）按相对路径发 Range 请求探测，命中即返回完整直链；探测失败回退 64K。
  - **低品(64K)** `QUALITY_LOW` / **标准 `QUALITY_STANDARD`**（djyule 无 128K 档，自动回退 64K）：`build64k(path)` —— 三路降级：
    1. 按路径前缀本地选 s64 主机（零网络）：`YYYYMMDD-` 年≥2022→`s64-20`、2019–2021→`s64-19`、更早→`s64-14`；`up<日期>-<尾号>` 尾号=2→`s64-2`，否则 `s64-14`；
    2. 兜底 `GET {BASE}/musicUrl.asp?id={id}`（须 `Referer:{BASE}/player/?id={id}`）解析 `musicUrl='https://w64-NN.djyule.com/'` → 映射 `http://s64-NN.jyw8.com`；
    3. 最终兜底池 `s64-1/2/14/19/20` 逐个 Range 探测。
  - 64K 解析失败则反向回退 320K，保证至少一档可播；未知 quality 视为低码。
- 协议：64K 用 `http://s64-NN.jyw8.com`（明文，已放行），320K 同域探测。

#### 3.2.3 55音乐 (55y) — standard + high（默认 high）
- 统一入口 **POST** `{API}/music/info`（即 `pc-api/music/info`，`Referer={BASE}/`）→ 解密响应 `row`。
- **高品(high)**：取 `row.downurl`（服务端返明文 `http://`，Android 9+ 需 cleartext 放行）。
- **标准(standard)**：取 `row.playurl`（以 `//` 开头，需补 `https:`）。
- 任一为空则互相兜底（high 空用 playurl；playurl 空用 downurl）；最终协议归一：以 `//` 开头补 `https:`，以 `http://` 开头升 `https://`。

#### 3.2.4 清纯DJ (ddddj) — standard + high
- 先 **POST** `{BASE}/ajax/wap/play`（表单，`Referer={BASE}/play/{id}`）→ 响应 `data.url`（pan.urlkj.com 签名直链）。
- 再 `toFreeUrl(signed, high)` 变换：
  - **标准(standard)**：**保留签名参数**（`?t=&sign=&kid=`）——实测去掉签名会被限速（≈40KB/s），保留才走全速通道；协议降 `http`。
  - **高品(high)**：**去除签名参数**，并把路径 **`/play/` → `/down/`**（高品文件约为标准 3.36 倍，码率显著提升）；签名与 `/down/` 高品通道不兼容，必须去签名再换路径；协议降 `http`。

#### 3.2.5 QQDJ舞曲 (qqdjs) — standard + high
- 取链：优先用列表解析缓存的 `data-audio` 直链（`playUrlCache`）；为空则抓详情页 `GET {BASE}/play/{id}.html` 提取内联 `var playurl`（注意 JS 转义的 `\/` 须先还原成 `/`，否则直链带 `\` 会 404）。
- `toFreeUrl(url, high)` 变换（与 ddddj 同款逻辑）：
  - **标准(standard)**：去 `?t=&sign=&kid=` 参数 + 改 `http`。
  - **高品(high)**：在去参数基础上把路径 **`/play/` → `/down/`**。

#### 3.2.6 先上DJ (xsdj123) — standard（高品回退）
- 取链：`GET {BASE}/music/info-{id}` 正则提取 `<audio ... src="https://listen.dj1234.com/...aac">` 直链填入 `playUrl`。
- 文档指出高品(320K)需登录后 POST `/music/down` 现取阿里云 `req_auth` 签名 URL；本项目**不做登录**，无法免登录构造，故 `quality == HIGH` 时**同样回退到标准 .aac 直链**（仅音质降低，链接可用）。即 UI 虽按 `SOURCE_QUALITY` 声明只显示 standard，即使传入 high 也安全降级。

#### 3.2.7 小熊电音 (xxedm) — standard + high（HTTP 站点）
- 取链：`GET {BASE}/thread-{id}.htm`（`BASE=http://xxedm.com`）提取 `<source src="https://pan.urlkj.com/play/...mp3">`（回退 `<audio src>`）。
- **标准(standard)**：去查询参数 + 改 `http`（`stdHttp`）。
- **高品(high)**：在去参数基础上：
  1. 去码率后缀（`_NNkbps`）；
  2. 路径 **`/play/` → `/down/`**；
  3. 按 `structure_c3` 提示的扩展名（wav/flac/mp3）构造候选，**逐个降速 HEAD 探活**（间隔 800ms，避免并发扫扩展名触发共享 CDN 限流）；
  4. 全部失败回退 `stdHttp`。
- 协议统一 `http://`（明文，已放行）。

### 3.3 取链层通用约定
- **缓存**：djyule（`playPathCache`/`s64HostCache`）、qqdjs（`playUrlCache`）、dj93（`urlCache`）均按 `songId` 缓存，避免重复请求。
- **防盗链**：取流请求统一注入各源 `Referer`（djf5=`{BASE}/`、djyule=`{BASE}/player/?id=`、55y=`{BASE}/`、dj14=`http://dj14.com/`、ddddj=`{BASE}/play/{id}`、djyyy=`{BASE}`）；spacedj 无 `Referer`。
- **cleartext**：55y / xxedm / dj14 返回或要求 `http://` 明文，依赖 `network_security_config` / `cleartextTrafficPermitted` 放行。
- **降级总原则**：高品取不到时回退标准（djf5/djyule/55y/qqdjs/xsdj123/xxedm 均如此），保证「至少一档可播」；单音质源忽略 quality 始终返回唯一档。

---

## 4. 通用请求约束（BaseHttpSource）

- 统一 HTTP 客户端：`BaseHttpSource.httpGetText`（含重试、UA、Referer、Accept 容错）。
- 防盗链：播放/下载流默认注入各源 `Referer`（详见各源 `SourceHeaders` / `*_REFERER` 常量）。
- DNS/host 识别：分享链接采用「归一化 host + 子域后缀匹配」，支持 www./m./play. 等任意子域。
- cleartext：部分源（55y、xxedm、dj14）返回或要求 `http://` 明文链接，需 `AndroidManifest` 的 `cleartextTrafficPermitted` 放行。

## 5. 备注

- 文档端点提取自 `2026-08-09` 源码快照；若后续新增/调整音源，请同步更新 `SourceManager`（映射表、音质表、分享链接规格）。
- 本次实测环境：djyyy / djyule / 多数源可达；spacedj.cn 服务端不可达（环境限制）。
