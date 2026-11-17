# bobidj.com（波比DJ音乐网）接入注意事项（避坑清单）

> 分析时间：2026-08-09（基于 `https://www.bobidj.com/play/142433.html` 实测 + 播放器 JS `enc.js` / `player.js` / `bobidj.js` 反编译）
> 分析对象：`bobidj.com`（波比DJ音乐网，nginx + 百度云加速 CDN `p2.bobidj.com`，音频 URL **AES-128-CBC 加密 + auth_key 鉴权**）
> 用途：AjunDJ 接入该音源时的注意事项（避坑清单），全部基于实测
> 合规提醒：第三方 DJ 分享站，聚合音源前请确认授权与版权，仅做个人播放/学习用途

---

## 0. 已验证的架构事实（理解注意事项的前提）

| 项 | 结论（实测） |
|----|--------------|
| 站点 | `bobidj.com`（波比DJ音乐网）；裸域 301 → `www.bobidj.com`（nginx） |
| 列表/作品集页 | `/ablum/info/{id}.html`（DJ 作品集，含多首，每首链到 `/play/{id}.html`） |
| **单首播放页** | `/play/{id}.html`（如 `/play/142433.html`）——**音频直链在此页经 JS 解密得出** |
| **取链机制（核心）** | 播放页内联**密文** + `decrypt()` 客户端解密：<br>`playurl = decrypt("{AES密文}", "8m2LsoZHlMUXAt22", "KNS8U4QvCKdkZI5D")` |
| 加密算法 | **CryptoJS AES-128-CBC / PKCS7**（`enc.js` 内 `CryptoJS.AES.decrypt(word, key, iv)`）；key/iv 静态写在 `enc.js` |
| 解密后直链 | `https://p2.bobidj.com/{yyyy}/{m}/{d}/{BBxxxx}_{hash}.mp4?auth_key={expire}-{rand}-0-{md5}` |
| 音频 CDN | `p2.bobidj.com`（百度云加速，DNS `*.bdydns.com`，**多边缘**） |
| 试听格式/码率 | **AAC-in-MP4**（`.mp4` 实为 AAC 音频），~64 kbps、44.1kHz（样本 7:10 / 3.48MB）——试听档 |
| **auth_key 鉴权** | URL 带 `?auth_key=...`：**必带**，篡改/缺失 → **403**（防盗链）；但同曲 auth_key 跨多次抓取一致、数小时后仍有效，疑似长有效期/非严格过期 |
| 试听 Referer | **无锁**：带/不带 Referer 均 200 |
| 试听 Range | 支持：`Accept-Ranges: bytes`、中段 `206`、`X-Cache-Status: HIT` |
| 试听登录 | **免登录**：AES 解密纯客户端，无需 cookie |
| 下载接口 | `.icodown` 点击 → `bobidj.download()` → iframe `/down/index?id={id}` |
| 下载门槛 | **需登录 + VIP/B币**：导航含「升级VIP/充值B币/下载明细」，高音质 MP3 匿名不可得 |

> 实测 `id=142433`：`decrypt(...)` 还原出 `https://p2.bobidj.com/2026/8/3/BB147559_40c51.mp4?auth_key=1786238486-91762271-0-782261c33ff9727c0cb5ca264ce42d9d`；直连 200、`video/mp4`、`Content-Length: 3477808`、Range 206、ffprobe 确认 `aac`/~64k/44.1kHz。篡改 auth_key → 403，去 auth_key → 403。

---

## 1. 502 根因：音频 CDN 是多边缘（百度云加速），非单点⚠️

- **音频 CDN `p2.bobidj.com` 多边缘**：DNS 经 `*.bdydns.com`（百度云加速），有多节点，**不是单 IP 死点**。偶发 502 多为「被钉在坏边缘」或边缘抖动，退避重试通常换健康边缘恢复（类似 bibdj 的坏边缘机理）。
- **www 也是 CDN**：`bobidj.com` 经 `cdnhwcgqa21.com` 多 IP，播放页本身韧性好。
- **两段链路分开判断**：
  1. 播放页 `/play/{id}.html` 502 → www CDN 边缘问题。
  2. 解密出的 `p2.bobidj.com/...mp4` 403/502 → 先看 **auth_key 是否完整**（403=鉴权失败，不是宕机）；带对 auth_key 仍 502 才是 CDN 边缘问题，退避重试。
- **auth_key 是 CDN 边缘统一校验**：密钥全边缘共享，不会因为换了边缘就 403（所有边缘认同一个 auth_key）。

---

## 2. 取链：抓播放页 → 提密文 → AES-128-CBC 解密（需复现解密）✅

> bobidj 与 dj14（SSR 内联变量 + DeCode）/ pgdjz（接口）/ dj93（同源 302）/ xsdj123（`<audio>` 内联）/ xxedm（`<source>` 内联）都不同——它的**音频 URL 是 AES 加密的**，明文不内联、不靠接口，而是页面内联密文 + `enc.js` 的 `decrypt()` 客户端还原。AjunDJ 必须**复现 AES 解密**才能拿直链。

```js
// 伪代码（Node 验证可用）：
// 1) GET https://www.bobidj.com/play/{id}.html
// 2) 正则提取：decrypt("<密文>", "<key>", "<iv>")
//    密文 = 页面内 <script> 里的 decrypt(...) 第一参（base64）
//    key   = "8m2LsoZHlMUXAt22"   // 16B，静态（enc.js）
//    iv    = "KNS8U4QvCKdkZI5D"   // 16B，静态（enc.js）
// 3) AES-128-CBC / PKCS7 解密（key/iv 用 UTF-8 字节） → 得到带 auth_key 的 mp4 URL

const crypto = require('crypto');
function decrypt(ct, key, iv) {
  const d = crypto.createDecipheriv('aes-128-cbc',
    Buffer.from(key, 'utf8'), Buffer.from(iv, 'utf8'));
  return d.update(ct, 'base64', 'utf8') + d.final('utf8');
}
// decrypt(pageCiphertext, "8m2LsoZHlMUXAt22", "KNS8U4QvCKdkZI5D")
// => https://p2.bobidj.com/2026/8/3/BB147559_40c51.mp4?auth_key=1786238486-...-0-782261c33ff9727c0cb5ca264ce42d9d
```

提取要点：
- 播放页：`GET /play/{id}.html`（`id` 来自 `/ablum/info/` 列表里每首的 `/play/{id}.html` 链接，或内联 `var playingid={id}`）。
- 密文/密钥：`playurl = decrypt("{密文}", "{key}", "{iv}")` 三个字符串参数，正则 `decrypt\(\s*"([^"]+)"\s*,\s*"([^"]+)"\s*,\s*"([^"]+)"\s*\)` 提取。
- **key/iv 目前静态**（写在 `enc.js`，两次抓取一致）——可硬编码，但稳妥起见建议从 `enc.js` 现取，防站方轮换。
- **解密结果含完整 `?auth_key=...` 查询串，必须原样保留**（见 §3/§4），不要当多余参数剥离。
- 解密纯客户端、免登录、无需 cookie/签名头。

---

## 3. 音频直链规则（AAC 试听免登录；auth_key 必带；下载需 VIP）

| 项 | 实测 |
|----|------|
| 试听直链 | `https://p2.bobidj.com/{日期}/{BBxxxx}_{hash}.mp4?auth_key=...` → **带 auth_key 即 200/206**（AES 解密得出，免登录） |
| 试听格式 | **AAC-in-MP4**（`.mp4` 实为 AAC 音频，无视频轨）；ffprobe：`aac` / ~64 kbps / 44.1kHz / 7:10 |
| **auth_key 鉴权** | **必带**：篡改/缺失 → **403**；同曲 auth_key 跨抓取一致、数小时后仍有效（长有效期/非严格过期），可缓存，遇 403 刷新 |
| 试听 Referer | **无锁**：带/不带 Referer 均 200 |
| 试听 Range | `Accept-Ranges: bytes`、中段 `206`、`X-Cache-Status: HIT`（CDN 缓存命中） |
| 试听登录 | **免登录**：AES 解密客户端完成 |
| 下载接口 | `.icodown` → `bobidj.download()` → iframe `/down/index?id={id}` |
| 下载门槛 | **需登录 + VIP/B币**：导航含「升级VIP/充值B币/下载明细」，高音质 MP3 匿名不可得 |

> **关键结论**：
> - **试听/播放**：抓 `/play/{id}.html` → AES-128-CBC 解密 → 带 `auth_key` 的 `p2.bobidj.com/...mp4`（AAC）喂播放器；**免登录、无 Referer 锁、支持 Range**。
> - **auth_key 是头号坑**：它**不是普通查询参数**，而是 CDN 防盗链签名——**必须原样带在 URL 里**，否则 403。但它又不严格过期（实测可缓存），所以取一次解密结果可复用，遇 403 再 re-fetch+解密。
> - **下载/高品 MP3**：`/down/index?id={id}` 弹窗 + 登录+VIP 门，匿名拿不到；若要 MP3 需登录+VIP/B币，或用 ffmpeg 把免登录 AAC 转码（音质封顶 ~64k）。
> - **AES 密钥静态**：解密算法固定（CryptoJS AES-128-CBC），key/iv 写死在 `enc.js`，可稳定复现。

---

## 4. Referer / UA / auth_key（无 Referer 锁，但 auth_key 必须在 URL）

- **音频不强制 Referer**：`p2.bobidj.com` 带/不带 Referer 均 `200/206`（`Content-Type: video/mp4`）。取链/取流用正常浏览器 UA 即可——比 xsdj123 宽松。
- **auth_key 不是头，是 URL 查询串**：它必须**随 URL 一起发送**（就是 URL 的一部分），没法用请求头绕过；剥离/篡改 → 403。这是 bobidj 唯一的「防盗链」手段（Referer 不管，但 auth_key 管）。
- **CORS**：`<audio>` 直连播放不触发 CORS 问题（auth_key 已在 URL 内）。
- **签名/登录（试听）**：无。AES 解密纯客户端，无 cookie、无 `X-Requested-With`、无 Referer 要求。

---

## 5. 多边缘 CDN + auth_key 重试 ⚠️

- **音频 CDN 多边缘**（百度云加速）：遇 502/超时 → 指数退避重试，CDN 通常换健康边缘。
- **auth_key 失败 ≠ 宕机**：返回 **403** 才是 auth_key 问题（缺失/篡改/边缘校验异常），**不是 502**。若 403：先确认解密出的 URL 完整保留了 `?auth_key=...`；仍 403 则 **re-fetch 播放页 + 重新解密**（少数情况 auth_key 可能刷新），不要当宕机猛刷。
- 播放页 `/play/{id}.html` 自身走 www CDN（多边缘），读超时 ≥ 30s，退避重试。

---

## 6. 控制 QPS，避免触发限流 ⚠️

- 只预加载「**当前 + 下一首**」；失败后走指数退避，**不要硬重试成风暴**。
- 解密出的带 auth_key URL **可缓存**（实测长有效期），不必每播都解密；遇 403 再刷新。
- 试听免登录、无签名、低频安全；建议限速 + 缓存 `id→解密URL`。

---

## 7. 取流字段提取（已验证）

| 字段 | 来源 / 提取方式 | 状态 |
|------|----------------|------|
| 歌曲 ID | `/play/{id}.html` 末段；或页面 `var playingid={id}`；或 `/ablum/info/` 列表里的 `/play/{id}.html` | ✅ |
| **音频密文** | 播放页内联 `decrypt("{密文}","key","iv")` 第一参 | ✅ 页面内联 |
| **AES key/iv** | 静态 `enc.js`：`8m2LsoZHlMUXAt22` / `KNS8U4QvCKdkZI5D`（建议从 enc.js 现取） | ✅ 静态 |
| **试听直链** | AES-128-CBC 解密密文 → `p2.bobidj.com/...mp4?auth_key=...`（**auth_key 必带**） | ✅ 解密得出 |
| 试听格式 | `.mp4` → ffprobe 确认 AAC ~64k（MP4 容器纯音频） | ✅ |
| 歌名/歌手 | `/ablum/info/` 列表链接文本、`/play/` 页面标题 | ✅ |
| 封面 | `img.bobidj.com/static/upload/playbg/...`（播放页内联 `dj_{id}.jpg` 等） | ✅ |
| 文件大小/时长 | 播放页 `structure`/详情区（HTML 含，需解析）；时长可 ffprobe | ⏳ 页面解析 |
| **MP3 下载直链** | `/down/index?id={id}` 弹窗 + **登录 + VIP/B币**，无内联直链 | ⛔ 匿名不可得 |

---

## 8. 接入 checklist（落地要点）

1. `GET https://www.bobidj.com/play/{id}.html` 抓播放页。
2. 正则提取 `decrypt("{密文}","{key}","{iv}")` 三参；用 **AES-128-CBC / PKCS7**（key/iv 取 UTF-8 字节）解密 → 得到 `p2.bobidj.com/...mp4?auth_key=...`。
3. **解密结果原样用，保留 `?auth_key=...` 完整查询串**（剥离/篡改 → 403）。
4. 直连喂播放器（`<audio src>` / ExoPlayer）；AAC-in-MP4，支持 Range，可拖动/断点；**不拼高品**（试听 AAC ~64k 是唯一免登录档）。
5. **无需 Referer**（带不带都 200）；读超时 ≥ 30s；遇 502 → 退避重试（多边缘 CDN）；遇 **403 → 先查 auth_key 是否完整，再 re-fetch+解密**，别当宕机。
6. 解密 URL 可缓存复用；遇 403 刷新。密钥静态，可直接硬编码或从 `enc.js` 现取。
7. 下载/MP3：匿名不可得（需登录+VIP/B币）；要么接登录态，要么 ffmpeg 把免登录 AAC 转码（音质封顶 ~64k），要么换 mp3 原生站（xxedm/qqdjs/ddddj/55y/dj14）。
8. 列表先抓 `/ablum/info/{djid}.html` 拿 `/play/{id}.html` 链接，再逐首进播放页取密文解密。

---

## 附：与另九站对照（快速结论）

| 维度 | 55y | qqdjs | ddddj | dj14 | xxedm | pgdjz | hy57 | dj93 | xsdj123 | **bobidj（本站）** |
|------|-----|-------|-------|------|-------|-------|------|------|---------|--------------------|
| 502 主因 | 单点 | 单点 | 单点 | www单点 | 共享单点+限流 | www+CDN均单点 | www单点 | www+CDN双单点 | 音频多边缘/www单点 | **音频多边缘(百度云)** |
| 取链方式 | 接口DES | SSR变量 | SSR变量 | SSR变量+DeCode | SSR内联`<source>` | 接口POST | 接口JS片段 | 同源302 | HTML内联`<audio>` | **AES加密+解密** |
| 直链是否内联 | 否 | 是 | 是 | 是 | 是 | 否 | 否 | 否 | 是 | **否(密文内联)** |
| 音频主机 | — | pan.urlkj | pan.urlkj | play.dj1198 | pan.urlkj | st.pgdjz | m4a.hy57 | play.dj93 | listen.dj1234 | **p2.bobidj(百度云)** |
| 试听格式 | m4a | m4a | m4a | mp3 128k | MP3常64k | m4a | m4a | MP4纯AAC~100k | AAC~110k | **MP4纯AAC~64k** |
| 试听 Referer | 无强制 | 无锁 | 无锁 | 无锁 | 无锁 | 无锁 | 无锁 | 无锁 | ⚠️白名单必带 | **无锁** |
| 试听登录 | 免登录 | 免登录 | 免登录 | 免登录 | 免登录 | 免登录 | 免登录 | 免登录 | 免登录 | **免登录(解密即可)** |
| **额外鉴权** | 无 | 无 | 无 | 无 | 无 | 无 | 无 | 无 | 无 | **⚠️auth_key必带** |
| 高品/MP3 | down.cdn | 替换 | pan down | 登录+VIP | play→down匿名 | 下载需登录 | 全开放 | 登录+VIP+验证 | 登录+VIP | **登录+VIP/B币 ❌匿名** |
| 接入难度 | 低 | 低 | 低 | 低 | 低(降速) | 低 | 低 | 低 | 低(Referer必带) | **中(AES解密+auth_key)** |

> 一句话：bobidj 取链是**加密型**——播放页内联 AES-128-CBC 密文，需复现 `enc.js` 的 `decrypt()`（key=`8m2LsoZHlMUXAt22`、iv=`KNS8U4QvCKdkZI5D`，静态）才能得到直链 `p2.bobidj.com/...mp4?auth_key=...`。试听**免登录、无 Referer 锁、支持 Range、AAC ~64k**，但 **URL 必带 `auth_key` 防盗链签名（篡改/缺失→403）**——这是本站头号坑。密钥静态可复现，auth_key 实测长有效期可缓存（遇 403 再 re-fetch+解密）。音频 CDN 是百度云加速多边缘，502 退避重试即可。下载高音质 MP3 需登录+VIP/B币，匿名只有 AAC 试听流。
