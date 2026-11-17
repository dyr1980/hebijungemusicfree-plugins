# djmei.com（魅力DJ网）全站接口分析报告 —— 安卓接入版

> 分析日期：2026-08-11 ｜ 实测环境：PC站 + 移动端H5 + 官方API，所有接口均已真实请求验证

---

## 一、网站架构总览（重要结论先行）

| 端 | 地址 | 技术栈 | 接入价值 |
|---|---|---|---|
| PC 站 | `https://www.djmei.com/` | 帝国CMS（Empire CMS）+ jQuery + jPlayer，pjax 渲染 | HTML 解析可行，作为兜底 |
| 移动端 H5 | `https://m.djmei.com/` | **Vue SPA**（axios），纯 JSON 数据驱动 | ⭐ 背后是一套完整 JSON API，**安卓接入首选** |
| 官方 APK | `https://m.djmei.com/djmei_apk_v1.0.apk` | v1.0，2019-05 发布，6.6MB | 已过时，仅参考 |

**核心发现**：移动端 Vue 应用的全部数据来自 `https://www.djmei.com/api/` 下的 JSON 接口，**无需登录、无签名、无 token**（播放/列表/搜索/详情均可游客访问），是安卓 App 接入的最佳路径。

**两个必须明确的结论**：
1. **音乐视频：本站没有视频板块**。全站纯音频（mp3），首页/栏目/菜单中无任何 video/MV 入口，`/video/ /mv/` 等路径均 404。
2. **高品质链接：只有一种音质**。全部音频为 **MPEG1 Layer3 320kbps / 48kHz MP3**（已实测解析帧头），下载/播放接口不存在多清晰度选项（`pathid` 只有 0 有效）。

---

## 二、分类体系（全部类目 + 子类目）

来自实测接口 `GET /api/Song.php?enews=menu`：

| classid | 分类名 | 子分类（classid） |
|---|---|---|
| **1** | 舞曲串烧（父分类，聚合全部串烧） | 3=车载串烧、4=粤语串烧、5=歌曲串烧、6=蓝调串烧、7=前场串烧、8=包房串烧、9=酒吧串烧 |
| **2** | 中文DJ单曲（无子分类） | — |
| **10** | 专辑套曲（无子分类，多曲目合集） | — |

菜单接口返回原文（Unicode 解码后）：

```json
GET https://www.djmei.com/api/Song.php?enews=menu
{
  "1":  { "classid": "1",  "classname": "舞曲串烧",
          "minm": { "3":{"classid":"3","classname":"车载串烧"},
                    "4":{"classid":"4","classname":"粤语串烧"},
                    "5":{"classid":"5","classname":"歌曲串烧"},
                    "6":{"classid":"6","classname":"蓝调串烧"},
                    "7":{"classid":"7","classname":"前场串烧"},
                    "8":{"classid":"8","classname":"包房串烧"},
                    "9":{"classid":"9","classname":"酒吧串烧"} } },
  "2":  { "classid": "2",  "classname": "中文DJ单曲", "minm": [] },
  "10": { "classid": "10", "classname": "专辑套曲",   "minm": [] }
}
```

各分类实测曲库规模（`num` 字段）：中文DJ单曲 ≈7491 首、舞曲串烧 ≈5422 首、车载串烧 ≈1692 首。

---

## 三、JSON API 完整清单（安卓首选方案）

**Base URL：`https://www.djmei.com/api`**
通用请求头建议：`User-Agent`（任意移动端 UA 即可，实测无 Referer 校验、无防盗链）。

### 3.1 首页聚合数据（含排行榜）

```
GET /api/Song.php?enews=getIndexdata
```

返回结构（实测）：

| 字段 | 内容 | 对应 PC 首页板块 |
|---|---|---|
| `ad` | 轮播广告图（picurl + url） | 首页 Banner |
| `firsttitle` | 头条推荐（含大图封面，10条） | 焦点图 |
| `danqu` | 最新单曲（8条） | 最新单曲 |
| `chuanshao` | 最新串烧（8条） | 最新串烧 |
| `taoqu` | 最新套曲（8条） | 最新套曲 |
| `hotSong` | **新品排行榜 Top10** | 新品排行榜（与PC首页逐条一致） |

列表项统一结构：

```json
{
  "title":    "DJ龙二少 - 丽江打跳(DJ舞曲版)",   // 标题（歌手嵌在标题里，见第五章）
  "titlepic": "/file/danqu/2025-11-01/xxx.jpg", // 封面图相对路径，需拼域名
  "id":       "16071",                          // 歌曲ID
  "classid":  "2"                               // 所属分类
}
```

> 排行榜说明：本站**没有独立排行榜页面**（`/paihang/ /top/ /rank/` 均 404）。排行榜只有首页 widget：新品排行榜（=`hotSong`）、单曲排行榜、串烧排行榜、套曲排行榜（各 Top10）。安卓端可用 `SongList + orderby=click` 按分类模拟"单曲/串烧/套曲排行"。

### 3.2 分类列表（支持排序、分页）

```
GET /api/Song.php?enews=SongList&classid={classid}&page={page}&orderby={orderby}
```

| 参数 | 说明 |
|---|---|
| `classid` | 见分类表；传父类 1 返回全部串烧 |
| `page` | **从 0 开始**，每页固定 20 条 |
| `orderby` | 空/`new`=最新；`click`=按浏览量（热度）；`isgood`=推荐 |

返回（实测）：

```json
{
  "showmore": 1,                 // 1=还有下一页，0=已到末页（越界时 SongList 为空数组）
  "num": 7491,                   // 该分类总条数
  "classname": "中文DJ单曲",
  "SongList": { "1": { "title":"...", "titlepic":"...", "id":"16071", "classid":"2" }, ... }
}
```

> 注意：`SongList` 是对象（键为 "1","2",...），安卓端反序列化建议用 `Map<String,Item>` 或 `LinkedHashMap` 保序。

### 3.3 歌曲详情（字段最全的接口）⭐

```
GET /api/Song.php?enews=SongDetail&id={id}
```

实测返回（id=16070）：

```json
{
  "SongDetail": {
    "title":    "DJ龙二少 - 大悲咒(DJ加快版)",
    "titlepic": "/file/danqu/2024-04-27/428afd01a0bf719f9a5c57654c41b7e9.jpg",
    "id":       "16070",
    "classid":  "2",
    "onclick":  "801",            // 浏览量
    "songsay":  "",               // 歌曲描述/介绍（多为空）
    "timelong": "04分18秒",        // 时长（文本格式）
    "newstime": "1714148630",     // 更新时间：Unix 秒级时间戳（= 2024-04-27）
    "filesize": "10.10MB",        // 文件大小
    "filetype": "mp3"             // 文件类型
  },
  "MoreSong": [ ... ]             // 相关推荐 10 条（title/titlepic/id）
}
```

### 3.4 搜索

```
POST /api/Song.php?enews=search        ← 必须 POST（GET 会返回"请输入搜索关键词"）
Content-Type: multipart/form-data （或 x-www-form-urlencoded）

keyboard=邓紫棋
page=0            // 从 0 开始，每页 20 条
```

实测返回：

```json
{ "showmore": 1, "num": 58, "SongList": { "1": { "title":"邓紫棋_泡沫_[DJ贺仔_Rmx_2017]", "titlepic":"/file/xiao/358.jpg", "id":"7455", "classid":"2" }, ... } }
```

> 搜索为标题模糊匹配，无歌手/专辑独立检索维度；结果无时长/大小字段，需要的话拿 id 再调 SongDetail。

### 3.5 播放地址接口 ⭐（核心）

```
POST /api/play.php
a=play          // 单曲
q={id}          // 歌曲ID
n={index}       // 曲目序号，普通歌曲恒为 0；套曲为曲目下标（0 起）
```

实测返回（单曲）：

```json
{
  "title":    "DJ龙二少 - 大悲咒(DJ加快版)",
  "titlepic": "/file/danqu/2024-04-27/428afd01a0bf719f9a5c57654c41b7e9.jpg",
  "downpath": "/file/danqu/2024-04-27/5332c113a4dc65e5bcd68b52eeb55834.mp3"
}
```

**真实播放 URL = `https://www.djmei.com` + `downpath`**，已实测：
- HTTP 200，`Content-Type: audio/mpeg`，大小与详情页 filesize 一致
- **`Accept-Ranges: bytes` → 支持 Range 断点/拖动进度**，ExoPlayer/MediaPlayer 直接可播
- 实测帧头：MPEG1 Layer3，**320kbps，48kHz**

批量取播放地址（整页列表一次性拿 URL）：

```
POST /api/play.php
a=playlist
q=16070,7503      // 逗号分隔多个ID
n=0,0
→ 返回 JSON 数组：["/file/xxx.mp3","/file/yyy.mp3"]
```

### 3.6 下载接口

```
POST /api/down.php
i={id} & c={classid} & p={track_index}
```

实测返回（单曲，未登录即可）：

```json
{
  "state": "success",
  "link": "<a href='https://www.djmei.com/api/doaction.php?enews=DownSoft&classid=2&id=16070&pathid=0&pass=a80a7726b56770a60164314fad21d3fe&p=::::::'>点击下载</a><span class='tip'>今天你剩余次下载机会。</span>"
}
```

- 需从 `link` 的 HTML 里正则提取真实地址：`enews=DownSoft&classid=..&id=..&pathid=0&pass={md5}&p=::::::`
- 该地址实测 200，`Content-Disposition: attachment`，直接就是 mp3 文件；`pass` 由服务端签发、带时效，**不要缓存复用**
- **权限差异（实测）**：单曲（classid=2）游客可下载（有每日次数提示）；**套曲（classid=10）下载必须登录**，未登录返回 `你还没有登录！`

### 3.7 用户体系接口（可选功能）

```
POST /api/member-ajax.php
```

| enews | 参数 | 说明 |
|---|---|---|
| `login_ajax` | username, password | 登录，成功返回文本 `登录成功!`（Cookie 会话） |
| `loginout_ajax` | — | 退出 |
| `register_ajax` | username, phone, phoneCode, password, repassword | 注册（需短信验证码） |
| `getFava` | page | 获取收藏列表，返回 `{state:"success", FavaList:[...], user:{...}}` |
| `AddFava_ajax` | i, c, p | 添加收藏 |
| `js` | — | 查询登录状态（游客返回空） |

```
POST /api/sendSms.php        // phone=xxx&randCode=xxx&sendSms=reg 发送注册验证码
```

---

## 四、字段需求对照表（你要的每一项）

| 需求 | 来源字段 | 获取接口 | 备注 |
|---|---|---|---|
| 歌曲名/标题名 | `title` | 所有接口 | 唯一标题字段 |
| 歌手 | ⚠️ **无独立字段** | — | 命名惯例嵌在 title 中：`歌手_歌名[Remix作者]`（如"邓紫棋_泡沫_[DJ贺仔_Rmx_2017]"）；串烧多为 `DJ名_描述`；`-` 分隔也常见（"DJ龙二少 - 大悲咒"）。安卓端需自行按 `_` / `-` 规则拆分 |
| 封面图 | `titlepic` | 列表/详情/play | 相对路径，拼 `https://www.djmei.com` 前缀；小图目录 `/file/xiao/`，大图按日期目录 |
| 文件大小 | `filesize` | SongDetail | 文本如 `10.10MB`；**套曲为多值，用 `####` 分隔** |
| 歌曲时长 | `timelong` | SongDetail | 文本如 `04分18秒`、`1小时06分40秒`；**套曲为多值 `####` 分隔** |
| 更新时间 | `newstime` | SongDetail | Unix 秒级时间戳，自行转日期（PC 页显示 yyyy-mm-dd） |
| 浏览量 | `onclick` | SongDetail | PC 端另有计数器 `/e/public/ViewClick/?classid={c}&id={id}&addclick=1` |
| 文件类型 | `filetype` | SongDetail | 恒为 mp3 |
| 分类/子分类 | menu / `classid` | menu、各列表 | 见第二章 |
| 播放链接 | `downpath` | play.php | 320kbps 直链，支持 Range |
| 下载链接 | down.php→DownSoft | down.php | pass 带时效 |

---

## 五、专辑套曲（多曲目）特殊处理 ⭐

套曲（classid=10）是"一个 ID 下多首歌"的结构，接入时最容易踩坑：

1. **详情**：`timelong` / `filesize` 为 `####` 分隔的多值串，段数 = 曲目数。
   实测 id=16066：`timelong = "####06分36秒####03分04秒####03分49秒..."`（注意开头有一个空的 `####`，解析时过滤空段）
2. **曲目名**：JSON 接口不返回曲目列表，需从 PC 详情页 HTML 解析：
   `GET https://www.djmei.com/taoqu/{id}.html` → `ul.song_list li h2` 即每轨标题；
   每个播放按钮属性为 `q-data="{套曲id}" n-data="{轨号0起}"`
3. **播放单轨**：`play.php` 传 `q={套曲id}&n={轨号}`，实测返回该轨独立 mp3（如 `/file/taoqu/qexkfaf253x.mp3`）
4. **下载**：需登录（见 3.6）

---

## 六、PC 端 HTML 解析方案（备用/补充）

若 JSON 接口失效，可解析 PC HTML（帝国CMS 结构稳定）：

| 功能 | URL 规则 | 关键元素 |
|---|---|---|
| 分类列表 | `/danqu/`、`/chuanshao/chezai/` 等；翻页 `/{栏目}/index_{N}.html`（N 从 2 起，首页无后缀） | `li` 内含 `img`(封面)、`a[title]`(标题)、`.playnow[q-data]`(ID)、`.right_detail .long/.size/.time`(时长/大小/更新日期) |
| 详情页 | `/danqu/{id}.html`、`/chuanshao/{子类}/{id}.html`、`/taoqu/{id}.html` | `h1` 标题、`img.song_pic` 封面、`.single_minutiae .category/.value` 键值对（分类/时长/更新时间/大小/类型） |
| 搜索 | `POST /search/search.php`（keyboard/show=title/tempid=1）→ 302 到 `/search/?searchid={N}`；翻页 `/search/index.php?page={N}&searchid={N}` | 结果结构同分类列表 |
| 排行榜 | 仅首页 widget，无独立页 | `.index_title h3` 后的 `ul li a` |

PC 页按钮属性约定（与 API 参数一一对应）：
`playnow: q-data=id, n-data=轨号` ｜ `addfav/downloadnow: i-data=id, c-data=classid, p-data=轨号`

---

## 七、安卓接入工程建议

1. **数据层**：直接对接 `/api/Song.php`（GET）+ `play.php / down.php`（POST form）。OkHttp/Retrofit 均可，无需鉴权头，建议带移动端 UA。
2. **播放器**：ExoPlayer 直接消费 `https://www.djmei.com{downpath}`；服务端支持 Range，进度拖动/缓存无压力。
3. **图片**：Glide/Coil 加载 `https://www.djmei.com{titlepic}`；小图 `/file/xiao/*.jpg` 尺寸小适合列表，详情大图在日期目录下。
4. **歌手拆分**：按惯例正则拆 title：优先 ` - `（"歌手 - 歌名"），其次 `_`（"歌手_歌名[Remix]"），串烧类整体作为标题不拆。
5. **分页**：page 从 0 起、每页 20 条，用 `showmore` 判断是否加载下一页，`num` 做总数展示。
6. **下载**：每次现取 down.php 的 DownSoft 链接（pass 有时效），单曲免登录、套曲需登录态（Cookie：`/api/member-ajax.php enews=login_ajax` 登录后保持会话）。
7. **稳定性与合规**：
   - 该 API 是 H5 私有接口、非官方开放 API，字段可能随改版变化，建议做好降级（HTML 解析兜底）与字段容错。
   - 网站自身声明"内容整合自网络、非盈利分享"，曲库版权状态不明；**商用 App 接入前务必评估音乐版权风险**，建议控制请求频率、勿对音频做二次分发。
   - 音频仅 320kbps 单一音质；无 MV/视频内容可接。

---

## 八、实测 curl 速查（可直接验证）

```bash
# 分类菜单
curl "https://www.djmei.com/api/Song.php?enews=menu"
# 首页+排行榜
curl "https://www.djmei.com/api/Song.php?enews=getIndexdata"
# 单曲列表（最新）
curl "https://www.djmei.com/api/Song.php?enews=SongList&classid=2&page=0&orderby="
# 列表（热度排序）
curl "https://www.djmei.com/api/Song.php?enews=SongList&classid=2&page=0&orderby=click"
# 详情
curl "https://www.djmei.com/api/Song.php?enews=SongDetail&id=16070"
# 搜索
curl -X POST "https://www.djmei.com/api/Song.php?enews=search" -F "keyboard=邓紫棋" -F "page=0"
# 取播放地址（单曲）
curl -X POST "https://www.djmei.com/api/play.php" -F "a=play" -F "q=16070" -F "n=0"
# 取播放地址（套曲第2轨）
curl -X POST "https://www.djmei.com/api/play.php" -F "a=play" -F "q=16066" -F "n=1"
# 批量取播放地址
curl -X POST "https://www.djmei.com/api/play.php" -F "a=playlist" -F "q=16070,7503" -F "n=0,0"
# 下载链接（单曲免登录）
curl -X POST "https://www.djmei.com/api/down.php" -F "i=16070" -F "c=2" -F "p=0"
# 直链验证（支持 Range）
curl -I "https://www.djmei.com/file/danqu/2024-04-27/5332c113a4dc65e5bcd68b52eeb55834.mp3"
```
