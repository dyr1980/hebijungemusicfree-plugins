# qqdjs.com 接口文档（QQDJ舞曲网）

> 分析时间：2026-08-08
> 分析方法：Playwright（Chromium 渲染）抓包 + `curl` 只读 GET 实测
> 结论先行：**可正常播放。音频托管在 `pan.urlkj.com`（与 ddddj.com / xxedm.com 同一 CDN 体系），免授权直链规则与 ddddj 完全一致；SSR 纯 HTML，音频直链内联在详情页 `var playurl`，无加密、无 JS 反爬。**
> 合规提醒：第三方 DJ 分享站，聚合音源前请确认授权与版权，仅做个人播放/学习用途。

---

## 1. 站点概况

| 项目 | 内容 |
|------|------|
| 主站 | `https://www.qqdjs.com/`（强制 https；`http://` 301 跳 https） |
| 前端模板 | `dj_msc`（`/template/web/dj_msc/...`，与 ddddj 同源同构） |
| 渲染方式 | SSR 纯 HTML，列表/详情均服务端输出，无 XHR 拉取音频 |
| 音频播放器 | `/template/web/dj_msc/assets/js/player/player.js` + HTML5 `<audio id="audio-player">` |
| 音频 CDN | `pan.urlkj.com`（带 `t`/`sign`/`kid` 签名参数，但签名**非强制**） |
| 音频格式 | MP3（`Content-Type: audio/mpeg`） |

---

## 2. 核心接口清单（全部 SSR HTML，无加密）

| 接口 | 方法 | 说明 |
|------|------|------|
| `/` | GET | 首页（最新/推荐/榜单入口） |
| `/genre/{风格}/` | GET | 分类（按音乐风格）列表页，末段追加伪静态分页 |
| `/play/{id}.html` | GET | **单曲详情/播放页**，内联 `var playurl = "..."` 即真实音频直链 |
| `/ranks/sole.html` | GET | 独家推荐榜（sole），伪静态分页 |
| `/ranks/chart?period=total` | GET | 排行榜，period：`total`/`day`/`week`/`month` |
| `/songs.html` | GET | 最新舞曲列表 |
| `/album.html`、`/album/{id}/` | GET | 专辑频道 |
| `/radio/`、`/radio/{id}/` | GET | 电台/歌单频道 |
| `/djshow/`、`/djshow/{id}/` | GET | DJ 制作人主页 |
| `/search/?keys={kw}` | GET | 搜索（关键词 GET 参数） |
| `/search/page/{N}.html?keys={kw}&cid={cat}&type={type}` | GET | 搜索结果分页（参数化，区别于列表伪静态） |
| `/user/account.html`、`/user/charge.html`、`/user/music/down.html` | GET | 用户中心 / 充值 / 下载记录 |

### 2.1 分类（genre）枚举（实测抓取到的风格标识）

中文串烧类：`xccs`(粤语串烧) `zwcs`(中文串烧) `wycs`(外语串烧) `zycs`(综艺串烧) `zwwq`(中文舞曲) `ywwq`(英文舞曲)
越南/国际类：`wwBounce` `wwClub_Dance` `wwDisco` `wwExtended` `wwLakHouse` `wwVinaHouse` `wyElectro` `wyFunky` `wysProg` `vinahouse` `VINAEDM` `lakhouse` `VibeDrop`
电音/浩室类：`BigRoom_EDM` `China_Trend` `Club_Dance_pop` `Contrast_Rank_Transition` `DanceClub` `Deep_House_Nu_Disco` `Dubstep_Trap_DubMix` `Electro_House` `Funky_Prog` `Future_Bass` `Hip-Hop` `KoreaStyle_Bounce` `Mashup_PartyBreak` `Pop_Music` `Trance_PSY` `bootleg` `breakbeat` `dissco` `electro` `extended` `funkyhouse` `hardbounce` `hardstyle` `melbourne` `original` `other` `proghouse` `reboedm` `techno`

> 风格 URL 示例：`/genre/xccs/`、`/genre/zwcs/`、`/genre/hardstyle/`、`/genre/vinahouse/`

---

## 3. 列表 / 搜索结果提取（歌曲名·歌手·文件大小·播放时长·封面图）

> 用于「搜索弹窗列表 / 分类列表」展示：每条结果需填 **歌曲名、歌手、文件大小、播放时长、封面图**。下面给出入口 URL 与逐字段提取规则。全部基于 SSR HTML，正则/属性即可，无需渲染。
> **本站特点（最规整）**：qqdjs 每条列表项把 **歌名、歌手、封面、播放直链、ID** 全放在 `data-*` 属性里，**文件大小/时长/码率** 在 `xiu-genre-song-meta` 的固定 `SIZE`/`TIME`/`Kbps` 文本里——几乎不用猜，正则一抓即得。

### 3.1 列表 / 搜索入口

| 场景 | URL | 翻页 |
|------|-----|------|
| 搜索 | `GET /search/?keys={kw}` | 参数化：`GET /search/page/{N}.html?keys={kw}&cid=0&type=1` |
| 分类 | `GET /genre/{风格}/{lang}-{tag}-{月/年}-{地区}-{page}.html`（末段=页码） | 末段 `{page}` 递增 |
| 榜单 | `GET /ranks/sole/{lang}-{tag}-{月/年}-{地区}-{page}.html`、`GET /ranks/chart?period=total` | 末段 `{page}` 递增 |

### 3.2 单条 item 的真实 HTML（节选，来自 `/genre/xccs/`）

```html
<div class="xiu-genre-song-row song-row sbg" data-id="301706">
  <div class="xiu-genre-song-row__inner">
    <div class="xiu-genre-col-cover cover-box isgood_list dj-genre-song-cell"
         data-id="301706"
         data-name="乐清DJ章恒-深情的东南亚上头的迷幻咚咚FK"
         data-artist="乐清DJ章恒"
         data-url="/play/301706.html"
         data-cover="https://www.qqdjs.com/cover/579.jpg"
         data-audio="https://pan.urlkj.com/play/260716/...mp3?t=...&sign=...&kid=...">
      <a class="playnow xiu-genre-cover-link" href="javascript:void(0);">
        <img src="https://www.qqdjs.com/cover/579.jpg" alt="" width="48" height="48"></a>
    </div>
    <div class="xiu-genre-col-info">
      <p class="xiu-genre-song-title">
        <a href="/play/301706.html" class="xiu-genre-song-title-link" title="乐清DJ章恒-...">乐清DJ章恒-...</a></p>
      <p class="xiu-genre-song-meta">
        <span>ID 301706</span><span>TIME 62:46</span><span>SIZE 144.74 MB</span><span>322 Kbps</span></p>
    </div>
    ...
  </div>
</div>
```

### 3.3 逐字段提取规则

| 字段 | 列表是否直接有 | 来源 / 提取方式 | 样例 |
|------|----------------|----------------|------|
| **歌曲 ID** | ✅ 是 | `div.xiu-genre-song-row` 的 `data-id`，或 `data-url="/play/{id}.html"` → `play/(\d+)`，或 meta `ID (\d+)` | `301706` |
| **歌曲名** | ✅ 是 | `data-name` 属性（或 `.xiu-genre-song-title-link` 的文本/title） | `乐清DJ章恒-深情的东南亚上头的迷幻咚咚FK` |
| **歌手名** | ✅ 是 | **`data-artist` 属性（独立字段，最干净）** | `乐清DJ章恒` |
| **文件大小** | ✅ 是 | `.xiu-genre-song-meta` 中 `SIZE ([\d.]+ ?MB)` | `144.74 MB` |
| **播放时长** | ✅ 是 | `.xiu-genre-song-meta` 中 `TIME (\d{1,2}:\d{2})`（mm:ss） | `62:46` |
| **封面图** | ✅ 是 | **`data-cover` 属性**（即最终地址，已含 https 主站域名；等同 `<img src>`） | `https://www.qqdjs.com/cover/579.jpg` |
| **播放直链** | ✅ 是 | `data-audio`（pan.urlkj.com play URL 带签名；建议去参数改 http 见 §4） | — |
| **码率** | ✅ 是 | `.xiu-genre-song-meta` 中 `(\d+ Kbps)`（原版标注码率） | `322 Kbps` |

### 3.4 提取示例（Python 正则）

```python
import re

ROW = re.compile(r'<div class="xiu-genre-song-row[^"]*"[^>]*?(.*?)</div>\s*</div>\s*</div>', re.S)
def parse_list(html: str):
    items = []
    # 直接按 data-* 整块抓（每行一个字段，顺序固定）
    for blk in re.findall(r'data-id="(\d+)"\s*data-name="([^"]*)"\s*data-artist="([^"]*)"\s*'
                          r'data-url="([^"]*)"\s*data-cover="([^"]*)"\s*data-audio="([^"]*)"', html):
        tid, name, artist, url, cover, audio = blk
        # meta 里的 TIME / SIZE / Kbps（每个 row 出现一次，顺序取）
        items.append({
            "id": tid, "title": name, "artist": artist,
            "cover": cover, "play_url": audio,
        })
    # 时长/大小用全局顺序匹配（与上面 item 顺序一致）
    times = re.findall(r'TIME\s+(\d{1,2}:\d{2})', html)
    sizes = re.findall(r'SIZE\s+([\d.]+ ?MB)', html)
    for i, it in enumerate(items):
        it["duration"] = times[i] if i < len(times) else None
        it["size"]     = sizes[i] if i < len(sizes) else None
    return items
```

### 3.5 注意事项

- qqdjs 是最规整的一站：歌手、封面、歌名、时长、大小均有**独立字段**，无需启发式分词。
- 时长格式 `mm:ss`；大小格式 `MB`；如需秒数/字节自行换算（`parse_size_to_bytes` 见通用工具）。
- 封面 `data-cover` 即最终地址，可直接用（已含 `https://www.qqdjs.com`）。
- 播放直链 `data-audio` 带时效签名；接入时用「去参数 + 改 http」得到免签标准链（见 §4）。
- 列表无需登录、无 Referer 锁。

---

## 4. 音频资源规律

1. 单曲详情页 `/play/{id}.html` 的 HTML 内联变量：
   ```js
   var playurl = "https://pan.urlkj.com/play/{YYMMDD}/{歌曲名}.mp3?t={时间戳}&sign={哈希}&kid={站点key}";
   ```
2. CDN 域名：`pan.urlkj.com`（与 ddddj.com / xxedm.com 共用同一对象存储）。
3. 路径结构：`/play/{YYMMDD}/{名称}.mp3`（标准品质）、`/down/{YYMMDD}/{名称}.mp3`（高品质）。
4. `{YYMMDD}` 为上传日期目录（如 `260716` = 2026-07-16），`{名称}` 为 URL 编码的歌曲名。
5. 页面里的 `?t=&sign=&kid=` 是**防盗链签名，但对播放非强制**——去掉参数改用 `http://` 仍可正常拉流。

---

## 5. 音质规则：标准档 vs 高品质档

> 规则原文：「播放直链去掉 `.mp3` 后的参数，改 `http` 访问 → 拿到免授权**标准品质**；在此基础上把 `play` 替换成 `down` → 获取**高品质**。」

以单曲 `乐清DJ章恒-深情的东南亚上头的迷幻咚FK.mp3`（id=301706，目录 `260716`）为例：

| 变体 | URL | HTTP | Content-Length | Content-Type |
|------|-----|------|----------------|--------------|
| 页面原始（带签名 https） | `https://pan.urlkj.com/play/260716/...mp3?t=...&sign=...&kid=520ae49c` | 206（浏览器实测） | — | audio/mpeg |
| **标准品质（去参数 + http）** | `http://pan.urlkj.com/play/260716/...mp3` | 200 / 206 | **45,193,307 B（≈45.2 MB）** | audio/mpeg |
| **高品质（play→down + http）** | `http://pan.urlkj.com/down/260716/...mp3` | 200 / 206 | **151,765,958 B（≈151.8 MB）** | audio/mpeg |

> 高品质 ≈ 标准品质的 **3.36 倍**（151.8 / 45.2），确认 `down` 即无损/高码率版本。

**验证结论**

- ✅ 规则完全成立：去参数改 http = 标准品质；`play`→`down` = 高品质。
- ✅ 真实可播放：200/206，`Accept-Ranges: bytes`（支持 Range 拖动）。
- ✅ **无 Referer 锁**：不带 Referer 直接请求即 200/206。
- ✅ **CORS 开放**：`Access-Control-Allow-Origin: *`，WebView / fetch 跨域可直接用。
- ⚠️ 页面内签名 `?t=&sign=&kid=` 可省略；若保留，签名有效期很短（数十秒级），过期会 403，**接入优先用免签名的 http 标准/高品质链**。
- ⚠️ 高频连发同 CDN 可能触发**临时限流（偶发 403 / 连接抖动）**，稍后重试即恢复。

**转换代码（供 AjunDJ）**

```python
def qqdjs_to_links(page_playurl: str) -> dict:
    """返回标准/高品质免授权直链（http，去参数）。"""
    base = page_playurl.split("?")[0]        # 去掉 ?t=&sign=&kid=
    std = base.replace("https://", "http://")  # 标准品质
    high = std.replace("/play/", "/down/")     # 高品质
    return {"standard": std, "high": high}
```

### 5.1 关于「页面下载跳转链接」＝高品质直链（重点澄清）

很多同学会去抓详情页「我要下载 / 下载这首歌曲」按钮的跳转地址。分两层：

1. **下载按钮跳转页**：`onclick="DownLoad({id})"` → 打开 iframe 弹窗，内容指向 `/down/{id}`（如 `https://www.qqdjs.com/down/301706`）。该页是**登录门槛页**：未登录只显示「请登录后再下载 / 下载地址有效期 3 天 / 需验证码 / 消耗 M币」等提示，静态 HTML 不含裸直链；登录后由服务端动态下发一个有效期 3 天的签名下载地址。匿名请求拿不到可用高品质 URL。
2. **该页最终服务的高品质原版直链**：即 `http://pan.urlkj.com/down/{YYMMDD}/{名称}.mp3`。下载页标注本曲为 144.74 MB / 322 Kbps 原版，与上面实测高品质直链（151.8 MB）一致。**实测该直链无需登录、无需签名、支持 Range 即可 200/206 拉取**，所以 AjunDJ 接入「高品质/下载」时**直接构造 `play`→`down` 的 CDN 直链即可，不必走 `/down/{id}` 门槛页**。

> 一句话：详情页 `var playurl` 把 `play` 换成 `down`（并去参数改 http）得到的，就是「下载跳转」背后真正的高品质原版 MP3 直链。

---

## 6. 播放 / 取流请求链

```
1) GET  /play/301706.html                         → SSR，内联 var playurl = "https://pan.urlkj.com/play/260716/...mp3?t=...&sign=...&kid=..."
2) GET  http://pan.urlkj.com/play/260716/...mp3   ← 播放（去参数改 http 即 200/206, audio/mpeg）
3) GET  http://pan.urlkj.com/down/260716/...mp3   ← 高品质（play→down，约 3.36×）
```

- 取链：请求详情页 HTML → 正则 `var playurl\s*=\s*["']([^"']+)["']` 提取 → 用 §5 转换。
- 无需接口调用、无需登录、无需渲染。

---

## 7. 反爬 / 防盗链 / 鉴权

| 机制 | 是否存在 | 实测说明 |
|------|----------|----------|
| Cookie / JS 校验 | 否 | 纯 `curl` 拿完整 HTML 与 `var playurl` |
| 加密 / 签名（播放） | 否（签名非强制） | 去 `?t=&sign=&kid=` 后正常拉流 |
| Referer 防盗链 | 否 | 带/不带 Referer 均 200 |
| CORS | 否（反而全开） | `Access-Control-Allow-Origin: *` |
| 登录门槛（下载页） | 是 | `/down/{id}` 需登录才下发签名下载地址，但**高品质原版 CDN 直链本身免登录** |

---

## 8. 可访问性测试结论

```bash
# 1) 取详情页内音频直链
curl -sk -A "Mozilla/5.0" "https://www.qqdjs.com/play/301706.html" | grep -oE "var playurl = '[^']+'"

# 2) 标准品质（免授权 http）
curl -sk -A "Mozilla/5.0" -r 0-1023 "http://pan.urlkj.com/play/260716/...mp3" -o /dev/null -w "%{http_code}\n"
# -> 200

# 3) 高品质（play -> down）
curl -sk -A "Mozilla/5.0" -I "http://pan.urlkj.com/down/260716/...mp3" | grep -iE 'HTTP/|content-length|content-type|access-control-allow-origin'
# -> HTTP/1.1 200 | audio/mpeg | 151765958 | Access-Control-Allow-Origin: *
```

| 维度 | 结果 |
|------|------|
| 音频直链状态码 | 200（整文件）/ 206（Range 分段） |
| 防盗链(Referer) | 无 |
| CORS | `Access-Control-Allow-Origin: *` |
| Range 拖拽 | 支持（Accept-Ranges: bytes，206） |
| 高品质直链 | 可用 —— `play`→`down` 即高码率原版（约 3.36×） |

**综合判定：现在能播放（能）。** CDN DNS 正常、无防盗链、CORS 全开、支持 Range。高品质免登录直取，无需走下载门槛页。

---

## 9. 给 AjunDJ 的接入提示

1. **取直链流程**：列表页 `/genre/...` 或 `/ranks/...` 拿到 `/play/{id}.html` → 请求详情页 HTML → 正则提取 `var playurl = "(.+?)"` → 用 §5 `qqdjs_to_links()` 转标准/高品质。
2. **优先用免签 http 链**：避免处理 `t/sign` 时效签名；`down` 高品质约 3.36× 体积，按需切换清晰度。
3. **播放器选择**：`pan.urlkj.com` 返回 `audio/mpeg` 且 `Access-Control-Allow-Origin: *`，ExoPlayer / MediaPlayer / WebView `<audio>` 均可直接播放，无需额外代理。
4. **分页抓取**：
   - 分类/榜单：`{lang}-{tag}-{月/年}-{地区}-{page}` 末段递增翻页（每页约 38~40 首）。实测 `/genre/xccs/0-0-0-0-2.html` → 200。
   - 搜索：`/search/page/{N}.html?keys=&cid=0&type=1`。
5. **与 ddddj / xxedm 共用同一解析器**：三者均为 `dj_msc` 模板 + `pan.urlkj.com` CDN + 同一套去参数/play→down 规则，可抽象为一个通用 `UrlkjSource` 适配器（xxedm 高品质扩展名需额外按 `structure_c3` 处理，见 xxedm 文档）。
6. **注意点**：`http://pan.urlkj.com` 偶发连接抖动（沙箱侧或 CDN 侧），生产环境加失败重试。
