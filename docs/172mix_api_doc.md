# 172Mix 舞曲音乐网 — 接口逻辑分析与接口文档

> 分析目标：`https://www.172mix.com`
> 分析时间：2026-08-09
> 方法：服务端渲染 HTML 抓包 + 内联脚本逆向 + 直链可访问性验证（curl 实测）
> 适用场景：为 AjunDJ（Jetpack Compose + Media3/ExoPlayer）做多音乐源接入提供对接依据

---

## 一、站点技术画像

| 维度 | 结论 | 证据 |
|------|------|------|
| 后端框架 | **Laravel（PHP）** | 响应头 `Set-Cookie: 172mix_session=...`（加密 session）、`<meta name="csrf-token">`、`XSRF-TOKEN` cookie |
| 前端技术 | jQuery + jPlayer 播放器 | `assets/js/player.min.js` 实为 **jPlayer 2.9.2**；页面大量 jQuery 事件 |
| 页面渲染 | **服务端渲染（SSR）** 为主，无前端 SPA 框架 | 数据直接出现在 HTML 中，无 XHR 拉取列表 |
| 边缘加速 | 阿里云 ESA（CDN + 安全） | `Server: ESA`、`Set-Cookie: acw_tc=...`、`cdn_sec_tc=...` |
| 音频存储 | 独立子域名 `mp3.172mix.com`（nginx） | 直链 `https://mp3.172mix.com/mp3/{date}/{hash}.m4a` |
| 移动端 | `m.172mix.com` 自动跳转，与 PC 共用同一套音频直链 | UA 切换脚本；实测 m 站音频 URL 与 PC 完全一致 |

**核心结论**：本站**没有公开、成体系的 REST/JSON API**。列表、详情、搜索全部是 SSR HTML 页面；数据接口仅极少量（侧边栏、OAuth 状态）。音频以静态文件直链形式分发，**这是接入的关键入口**。

---

## 二、域名与资源清单

| 域名 | 用途 | 可外部访问 |
|------|------|-----------|
| `www.172mix.com` | 主站，所有 SSR 页面与业务接口 | 是 |
| `m.172mix.com` | 移动站（自动跳转），逻辑同主站 | 是 |
| `mp3.172mix.com` | **音频文件 CDN（直链）** | ✅ **是，无 Referer 限制** |
| `rjsprbi.myalicdn.com` | 阿里云 RUM 前端监控脚本 | 无关 |
| `hm.baidu.com` / `zz.bdstatic.com` | 百度统计/推送 | 无关 |

---

## 三、接口总览

| # | 接口 | 方法 | 返回类型 | 说明 |
|---|------|------|---------|------|
| 1 | `/play/{id}` | GET | HTML（内联 `media` 对象） | **曲目播放详情页，含音频直链** |
| 2 | `mp3.172mix.com/mp3/{date}/{hash}.m4a` | GET | `audio/x-m4a` | **音频文件直链，无防盗链，支持 Range** |
| 3 | `/tracks/sidebar?id={id}` | GET | `application/json` | 侧边栏「在听榜/下载榜」HTML 片段 |
| 4 | `/searches?query={kw}&page={n}` | GET | HTML | 搜索结果，30 条/页 |
| 5 | `/genre/{code}?page={n}` | GET | HTML | 分类/风格列表 |
| 6 | `/ranks/sole?page={n}` | GET | HTML | 独家舞曲排行榜（分页） |
| 7 | `/album/{id}` | GET | HTML | DJ 专辑 / 作者主页 |
| 8 | `/down/{id}` · `/tracks/perform-down/{id}` | GET | HTML / 302→登录或MP3 | 高品质(320Kbps)下载页与下载触发接口（**需登录**） |
| 9 | `/oauth/auth` · `/oauth/auth-status` | GET | redirect / JSON | 第三方账号登录与状态 |
| 10 | `/account/*` · `/tracks/create` · `/gathers/create` · `/group-buys` | GET/POST | HTML | 会员中心、上传、套曲、打包下载（多需登录） |

---

## 四、核心接口详解

### 1. 曲目播放详情 — `GET /play/{id}`

这是**最重要的接入入口**。页面为 SSR HTML，曲目的音频地址以内联 JS 对象 `media` 直接写入：

```javascript
// 来自 /play/290506 实测
media = {
    id: "290506",
    name: "Dj炮仔-全国语ProgHouse音乐热播吹吹山顶的风专辑172Mix串烧",
    link: "https://www.172mix.com/play/290506",
    category: "中文串烧",
    src: "https://mp3.172mix.com/mp3/20260808/6a76080b3cb80.m4a",
};
```

**请求示例**
```bash
curl -s "https://www.172mix.com/play/290506" | grep -A6 'media = {'
```

**解析要点**（AjunDJ 接入时）：
- 用正则提取 `media = {...}` 块 → JSON 化 → 取 `src`（音频直链）、`name`（标题）、`category`（风格）、`id`。
- 播放页同时内联 jPlayer 初始化代码，但**直链已可直接交给 ExoPlayer 播放**，无需复刻 jPlayer。

**字段说明**

| 字段 | 含义 |
|------|------|
| `id` | 曲目数字 ID（即 URL 中的 `{id}`） |
| `name` | 完整曲目标题（通常含 DJ 名 + 描述） |
| `link` | 本站播放页地址 |
| `category` | 分类/风格名（如「中文串烧」） |
| `src` | **音频直链**（m4a/AAC） |

---

### 2. 音频直链 — `GET https://mp3.172mix.com/mp3/{YYYYMMDD}/{hash}.m4a`

**实测响应头**
```
HTTP/1.1 200 OK
Server: nginx
Content-Type: audio/x-m4a
Content-Length: 29064586
Accept-Ranges: bytes
```
- 格式：**M4A / AAC**（容器 MP4，编码 AAC-LC，容器后缀为 `.m4a`）。
- **防盗链验证**：分别用「无 Referer」「`Referer: https://www.172mix.com/play/290506`」请求，**均返回 200**，说明**无 Referer/Origin 校验**。
- **Range 支持**：`Accept-Ranges: bytes` → 支持断点续传与播放拖动（seek），ExoPlayer 可直接流式播放。
- 路径规律：`/mp3/{上架日期YYYYMMDD}/{随机32位hex}.m4a`；`hash` 不可预测，必须从 `/play/{id}` 解析获得。

> ✅ 结论：该直链可**被任意外部播放器直接集成**，无需 Cookie、无需登录、无需 Referer。AjunDJ 直接把 `src` 喂给 `Media3/ExoPlayer` 即可播放/下载。

---

### 3. 侧边栏热门榜 — `GET /tracks/sidebar?id={id}`  ⭐ 唯一确认的 JSON 数据接口

**实测**：返回 `Content-Type: application/json`，结构为两个 HTML 片段字段：

```json
{
  "playsContent": "<html片段：当前在听/播放榜列表>",
  "downsContent": "<html片段：下载榜列表>"
}
```

**请求示例**
```bash
curl -s "https://www.172mix.com/tracks/sidebar?id=290506"
```

**说明**：该接口为播放页右侧栏「在听榜 / 下载榜」提供数据，返回的是**已渲染好的 HTML 片段**（非结构化数据）。用途有限，仅作首页/详情页附属推荐，不承载核心曲库。

---

### 4. 搜索 — `GET /searches?query={kw}&page={n}`

- 表单：`action="/searches"`、`input name="query"`（首页顶部搜索框）。
- 返回 SSR HTML，每页 **30 条**，`?page=N` 分页（实测「dj」关键词结果超 **4480 页**）。
- 结果条目即 `/play/{id}` 链接 + 标题 + 封面。

```bash
curl -s "https://www.172mix.com/searches?query=dj&page=1" | grep -oE '/play/[0-9]+' | sort -u
```

---

### 5. 分类/风格列表 — `GET /genre/{code}?page={n}`

分类 `code` 来自导航，常用如下（实测存在）：

| code | 分类 | 子分类 code |
|------|------|------------|
| `cswq` | 串烧舞曲 | `zwcs` 中文串烧 / `ywcs` 英文串烧 / `zycs` 中英串烧 / `slcs` 实录串烧 / `jpcs` 精品串烧 |
| `zwwq` | 中文舞曲 | `lak` LakHouse / `proghouse` Prog House / `electro` Electro House / `extended` Extended House / `electromelbourne` / `club` Dance Club |
| `gwwq` | 国外舞曲 | — |
| `house` | 电音 House | — |
| `ygzq` | 越鼓专区 | `LakHouse` / `prod` PRODUCER VN |
| `jbcs` | 酒吧串烧 | — |

分页 `?page=N`（如 `cswq` 约 450 页）。返回 HTML 列表，条目含 `/play/{id}`。

---

### 6. 排行榜 — `/ranks/sole` 与首页 Tab 榜

- **独家舞曲榜**：`GET /ranks/sole?page=N`（独立分页页，实测 1311 页）。
- **串烧榜 / 单曲榜（日/周/月）**：**无独立 URL**（实测 `/ranks/skewers`、`/ranks/day` 等均 404）。这些数据在**首页通过 Bootstrap Tab 锚点**展示，SSR 已内联：
  - `#ranks-skewers`（串烧榜）、`#ranks-single`（单曲榜）
  - 各自含 `day`/`week`/`month` 子标签：`#ranks-day-skewers`、`#ranks-week-skewers`、`#ranks-month-skewers`、`#ranks-day-single`、`#ranks-week-single`、`#ranks-month-single`

> 接入建议：若需排行榜数据，抓首页 HTML 解析对应 Tab 区块即可；独家榜走 `/ranks/sole`。

---

### 7. 专辑/DJ 主页 — `GET /album/{id}`

按 DJ/作者聚合的曲目列表页（如 `/album/336` = Dj爱意）。返回 HTML，含该作者全部 `/play/{id}` 条目。

---

### 8. 高品质下载链路 — `/down/{id}` 与 `/tracks/perform-down/{id}`  ⭐ 需登录

「高品质 MP3 下载」入口在播放页（`/play/{id}` 内有「请点此处进入高品质MP3下载」指向 `/down/{id}`）。实测下载页明确标注 **「下载音质：320Kbps」**，即真正的无损/高品文件与试听直链（m4a/AAC）不同源。

**下载触发接口**：`GET /tracks/perform-down/{id}`
- 下载页「点击下载」按钮的 `href` 即为此地址（页面 JS 按登录态渲染：未登录显示「登录后再下载」+ 注册链接；已登录才露出下载按钮）。
- **鉴权实测**：未登录（含带匿名 `172mix_session` cookie）访问该接口，**一律 `302 Found` 重定向到 `https://www.172mix.com/login`**，不返回任何音频地址。

```http
GET /tracks/perform-down/290506  →  302 Found
Location: https://www.172mix.com/login
```

**结论**
- 试听直链（`media.src`，m4a/AAC）**免登录**即可播放/下载，用于播放器集成足够。
- 320Kbps 高品质 MP3 **在会员/M币墙后**：必须先登录拿到有效 `172mix_session`，再携带该 Cookie 请求 `/tracks/perform-down/{id}`，接口才会吐出真实 MP3 地址（预期为 `302` 跳转到 `mp3.172mix.com` 的 `.mp3` 直链，或直接返回文件流 `Content-Disposition: attachment`）。
- 未持有账号时，**无法在无登录态下获取高品质直链**（本站对下载做了统一登录拦截，非前端隐藏）。

---

### 9. 账号与上传（需登录，外部集成价值低）

| 接口 | 说明 |
|------|------|
| `/oauth/auth` | 第三方登录入口（redirect） |
| `/oauth/auth-status` | 登录状态查询（疑似 JSON） |
| `/authenticates` | 音乐人/作者认证 |
| `/account` · `/account/charge` · `/account/favorites` · `/account/downloads` | 会员中心/充值/收藏/下载记录 |
| `/tracks/create` · `/gathers/create` | 上传舞曲 / 发布套曲 |
| `/group-buys` | 打包下载 |

---

## 五、风控与注意事项

| 项 | 说明 | 对 AjunDJ 影响 |
|----|------|---------------|
| `acw_tc` / `cdn_sec_tc` Cookie | 阿里云 ESA 安全 cookie，首次访问下发 | 访问 `www` 页面时建议携带；**访问 `mp3` 直链不受影响** |
| `XSRF-TOKEN` | Laravel CSRF | 仅 POST 类操作需要；**播放/列表为 GET，无需** |
| 频率限制 | ESA 可能对高频请求限流 | 批量爬取 `www` 页面需限速 + 复用 Cookie；直链 CDN 较宽松 |
| 无结构化 API | 列表/搜索均为 HTML | 接入需**解析 HTML**（或解析 `media` 对象），非 JSON 反序列化 |

---

## 六、AjunDJ 接入方案建议

基于以上分析，推荐「**直链优先**」接入策略：

1. **播放/下载**：拿到曲目 ID → 请求 `/play/{id}` → 正则提取 `media.src` → 直接交给 `Media3/ExoPlayer` 播放或下载。**无需任何鉴权**。
2. **曲库浏览**：通过 `/genre/{code}`、`/ranks/sole`、`/searches?query=` 分页爬取 HTML，提取每页 `/play/{id}` 与标题/封面，构建本地索引。
3. **搜索**：复用 `/searches?query=` + `?page=N`，解析结果条目。
4. **格式兼容**：音频为 **M4A/AAC**，ExoPlayer（`Media3`）原生支持；如需统一为 MP3 可在端侧转码（非必须）。
5. **缓存与 Range**：直链支持 `Accept-Ranges`，ExoPlayer 默认启用渐进式/分片下载与拖动，体验良好。

**最小可用数据流**
```
用户选曲/搜索
   → GET /play/{id}  (或 /searches、/genre/{code})
   → 解析 media.src (m4a 直链)
   → ExoPlayer.setMediaItem(直链)  → 播放
```

---

## 七、合规提醒

- 本站音频为第三方版权内容（含「独家」「DJ 提供」标注）。接口分析仅用于**技术对接验证与学习**，批量下载/再分发需遵守相关版权与平台条款。
- 建议 AjunDJ 仅做「播放聚合/跳转」，避免大规模镜像曲库；接入前确认所用音乐源的授权范围。

---

## 附：实测证据索引（本机抓取样本）

| 文件 | 内容 |
|------|------|
| `play.html` | `/play/290506` 页面，含 `media` 对象与音频直链 |
| `sidebar.html` | `/tracks/sidebar?id=290506` 返回的 JSON（playsContent/downsContent） |
| `searches.html` | `/searches?query=dj` 搜索结果（30/页） |
| `ranks.html` / `genre.html` | 排行榜/分类分页页（验证 `?page=N`） |
| 音频直链 HEAD 探测 | `mp3.172mix.com/...m4a` → 200 / audio/x-m4a / Accept-Ranges: bytes / 无 Referer 限制 |
