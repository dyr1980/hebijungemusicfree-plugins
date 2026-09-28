# pgdjz.com（皮狗DJ站）接入注意事项（避坑清单）

> 分析时间：2026-08-09（基于 `https://wap.pgdjz.com/down/163225.html` 实测 + 既有《pgdjz.com_接口文档.md》核对）
> 分析对象：`pgdjz.com`（皮狗DJ站，CSCms 音乐系统）；WAP 子域 `wap.pgdjz.com`
> 用途：AjunDJ 接入该音源时的注意事项，全部基于实测（curl 直连 + WAP 下载页解析）
> 合规提醒：第三方 DJ 分享站，聚合音源前请确认授权与版权，仅做个人播放/学习用途

---

## 0. 已验证的架构事实（理解注意事项的前提）

| 项 | 结论（实测） |
|----|--------------|
| 站点 | `pgdjz.com`（皮狗DJ站）；WAP 子域 `wap.pgdjz.com` |
| 最终可达 | `https://www.pgdjz.com/`（跳转链：`pgdjz.com` 301 → `https://pgdjz.com` 301 → `https://www.pgdjz.com` 200） |
| 播放页 | `/dance/{id}.html`（SSR），但**直链不内联**，需接口换取 |
| **取链接口（核心）** | `POST https://www.pgdjz.com/ajax/danceplayer`，参数 `id={歌曲id}` → JSON `playurl`（m4a 直链） |
| 音频 CDN | `st.pgdjz.fun`（**单 IP `118.184.172.39`**，IIS/8.5） |
| 图片 CDN | `tp.pgdjz.fun`（与音频**同 IP** `118.184.172.39`） |
| www 主机 | `www.pgdjz.com` → **单 IP `156.238.247.233`** |
| WAP 主机 | `wap.pgdjz.com` → **Cloudflare 边缘（`172.67.195.25`）** |
| 格式/码率 | **m4a / AAC**（`Content-Type: audio/mp4`）；文件名里的 `Mp3 ` 前缀只是历史命名，实际是 m4a |
| 防盗链 | **无**：音频带/不带 Referer 均 `200/206` |
| 登录 | **试听免登录**；**下载** `POST /ajax/dancedown` 需登录（返回「需要登录才可以下载」） |
| 双斜杠 | CDN 域名后 `//` 是路径根（`st.pgdjz.fun//...`），**必须原样保留** |

> 实测 `id=163225`：`/ajax/danceplayer` 返回 `playurl = https://st.pgdjz.fun//zwdqdj/2026/Mp3 袁小袁 - 痴情犯的罪(DJ彭锐版).m4a`，HEAD `200`、`Content-Length: 1680977`（1.68MB）、`audio/mp4`、Range `206` 正常。

---

## 1. 502 根因：www + 音频 CDN 都是单点，不是「坏边缘」也不是「限流」⚠️

- **www 与音频 CDN 都是单 IP**：`www.pgdjz.com` 仅 `156.238.247.233`，`st.pgdjz.fun` 仅 `118.184.172.39`。**没有 bibdj 那种多边缘 CDN**，所以「IP 被钉在坏边缘」的机理**不适用**。
- **真实成因**：两台单点源站在高并发/自身负载高时返回 **502 / 000 / 超时**，属**服务端单侧不稳定**，所有访问者都会间歇遇到。
- **不是防盗链导致**：本站音频**免登录、不校验 Referer**，拿到 502 一定是源站/链路问题，不是缺头。
- **WAP 走 Cloudflare**：`wap.pgdjz.com` 是 Cloudflare 边缘（`172.67.195.25`），边缘层有韧性；但**源站仍是上面那两台单点**，源站抽风时 WAP 也会 502。AjunDJ 建议直接打 **www（非 WAP）**，少一层 Cloudflare 又绕不开单点，反而更可控。
- **结论**：502 与「你的 IP」无关，是 pgdjz 单点架构的固有问题；按 §5 退避重试即可。

---

## 2. 取链：走 `POST /ajax/danceplayer` 接口（无需渲染）✅

> pgdjz 与 dj14/qqdjs 不同——**直链不内联在 HTML**，而是 JS 运行时调接口返回。但只要调这个**对服务端完全开放的 POST 接口**即可，无需浏览器渲染、无需登录。

```bash
curl -s -X POST "https://www.pgdjz.com/ajax/danceplayer" --data "id=163225" \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['playurl'])"
# => https://st.pgdjz.fun//zwdqdj/2026/Mp3 袁小袁 - 痴情犯的罪(DJ彭锐版).m4a
```

提取要点：
- 接口：`POST /ajax/danceplayer`，表单参数 `id={歌曲id}`（来自列表/详情页链接 `/dance/{id}.html` 的末段数字）。
- 返回 JSON：`{"code":1,"playurl":"https://st.pgdjz.fun//...m4a","msg":"获取成功"}`，取 `playurl`。
- `playurl` 是 **JSON 转义**的（斜杠写成 `\/`），使用前先 `.replace("\\/","/")` 还原。
- **免鉴权**：不带 `X-Requested-With`、不带 Referer、未登录均返回 `code:1`（已实测）。
- 直链**无法由 id 反推**（含分类目录 `zwdqdj/ywdqdj` 与文件名字段），必须每次经接口换取；建议缓存 `id → playurl` 并加 TTL。

---

## 3. 音频直链规则（试听 m4a 免登录；下载需登录）

| 项 | 实测 |
|----|------|
| 试听直链（接口返回） | `https://st.pgdjz.fun//{分类}/{年}/{名}.m4a` → **200 / 206**（来自 `/ajax/danceplayer`，免登录） |
| 试听实际格式 | `file`/响应头识别为 `audio/mp4`（m4a/AAC），路径里的 `Mp3 ` 前缀是历史命名 |
| 试听防盗链 | **无**：带/不带 Referer 均 `200/206`；CORS 对 `<audio>` 直连无影响 |
| 高品/下载 | `POST /ajax/dancedown?id={id}` → **「需要登录才可以下载」**（登录门槛页）；登录后才下发下载链 |
| 下载门槛强度 | 比 dj14 轻：dj14 需 **VIP/会员+U币**，pgdjz 仅提示「需登录」——普通会员大概率可下（**未实测验证是否还要 VIP**） |
| WAP 下载页 | `wap.pgdjz.com/down/{id}.html`：**页内无内联 m4a**，靠 ajax 调 `/ajax/dancedown`（同样登录门槛）；`wap /ajax/danceplayer` 302 跳回 www |

> **关键结论**：
> - **试听/播放**：`/ajax/danceplayer` 免登录直拿 m4a，AjunDJ 匿名即可播。
> - **下载/高品**：`/ajax/dancedown` 需登录，匿名拿不到下载链；若要高品建议引导官方或接登录态（成本高于dj14之外的站点，且 WAP 版无额外优势）。
> - **优先用 www，别用 WAP**：WAP 是 Cloudflare 套壳、下载页还登录门槛、播放接口还 302 跳 www——直接打 `www.pgdjz.com/ajax/danceplayer` 最干净。

---

## 4. Referer / UA（无锁，最简）

- **音频不强制 Referer**：`st.pgdjz.fun` 带/不带 Referer 均 `200/206`（`Content-Type: audio/mp4`）。取链接口用正常浏览器 UA 即可。
- **CORS**：音频由 `<audio>` 直连播放不触发 CORS；接口为同源 POST，前端调用无跨域问题。
- **签名/登录（试听）**：无。`/ajax/danceplayer` 明文返回 `playurl`，无需 cookie、无需签名、无需 `X-Requested-With`。

---

## 5. 单点架构 + 超时重试 ⚠️

- www（取链接口）与音频 CDN 都是**单 IP**，偶发慢/抽风。客户端**读超时建议 ≥ 30s**。
- 遇 `502 / 503 / 000 / 连接超时` → **指数退避重试 + 抖动**。单点源站重试仍打同一台，但瞬时过载会过去，短退避有效；不要无脑猛刷。
- 音频 CDN `st.pgdjz.fun` 通常健康；若个别请求超时，重试有效。

---

## 6. 控制 QPS，避免触发限流 ⚠️

- 只预加载「**当前 + 下一首**」；失败后走指数退避，**不要硬重试成风暴**。
- 正常低频节奏实测安全，无签名/无频控，但仍建议限速 + 接口结果缓存（见 §2 TTL 建议）。

---

## 7. 取流字段提取（已验证）

| 字段 | 来源 / 提取方式 | 状态 |
|------|----------------|------|
| 歌曲 ID | URL `/dance/{id}.html` 或 `/down/{id}.html` 末段数字 | ✅ |
| 音频直链 | `POST /ajax/danceplayer` → `playurl`，还原 `\/`→`/`（含双斜杠 `//`） | ✅ 接口返回 |
| 歌名/歌手 | 详情页 HTML / 列表链接文本（接口不返回，需另扒页面） | ⏳ 页面解析 |
| 封面 | `tp.pgdjz.fun` 图片 CDN（接口不返回） | ⏳ 页面解析 |
| 高品质/下载直链 | `POST /ajax/dancedown`（**需登录**） | ⛔ 匿名不可得 |
| 文件有效性 | `file` 识别为 `MPEG-4, AAC` / `audio/mp4`（1.68MB 样本） | ✅ |

---

## 8. 接入 checklist（落地要点）

1. `GET /dance/{id}.html` 或搜索页解析出歌曲 `id`（路径末段数字）。
2. `POST https://www.pgdjz.com/ajax/danceplayer`，表单 `id={id}` → 取 `playurl`。
3. **还原 JSON 转义斜杠**（`\/"`→`"/`），**保留域名后双斜杠 `//`**，发请求时正确 percent-encode（中文/空格）。
4. 直连 `playurl` 喂播放器（`<audio src>` / ExoPlayer）；支持 Range，可拖动/断点续传。
5. **不拼高品质**（试听 m4a 是唯一免登录档；下载走 `/ajax/dancedown` 需登录）。
6. 取流**不带 Referer 也行**；读超时 ≥ 30s；遇 502/503/000 → 指数退避重试；QPS 克制。
7. **用 www 不用 WAP**：WAP 套 Cloudflare、下载页登录门槛、播放接口还 302 跳 www，无收益。
8. 直链含分类目录+文件名、无法 id 反推，缓存 `id→playurl` 并加 TTL；周期性 `nslookup st.pgdjz.fun` + 抽样 HEAD 做可用性探测（防 CDN 下线）。

---

## 附：与另五站对照（快速结论）

| 维度 | 55y | qqdjs | ddddj | dj14 | xxedm | **pgdjz（本站）** |
|------|-----|-------|-------|------|-------|-------------------|
| 502 主因 | 单点不稳 | 单点不稳 | 单点不稳 | www 单点不稳 | 共享 CDN 限流 | **www+CDN 均单点** |
| 取链方式 | 接口 DES 解密 | SSR→pan.urlkj | SSR→pan.urlkj | SSR 内联 `__PLAYLIST_DATA__` | SSR 内联 `<source>` | **接口 POST `/ajax/danceplayer`** |
| 直链是否内联 | 否(接口) | 是(var playurl) | 是(var playurl) | 是(全局变量) | 是(`<source>`) | **否(接口返回)** |
| 标准档 | ✅ | ✅ | ✅ | ✅(128k) | ✅ | ✅ m4a 免登录 |
| 高品质档 | down.cdn ✅ | 免登录替换 ✅ | pan down ✅ | 登录+VIP ❌匿名 | 免登录 play→down ✅ | **下载需登录（门槛较轻）** |
| 音频 Referer | 无强制 | 无锁 | 无锁 | 无锁 | 无锁 | **无锁** |
| 接入难度 | 低(签名可算) | 低(替换+去参) | 低 | 低(正则提 JSON) | 低(正则提 JSON) | **低（接口开放，无需渲染）** |

> 一句话：pgdjz 取链是**接口型**——`POST /ajax/danceplayer?id={id}` 直接拿 m4a 直链，**免登录、无防盗链、支持 Range、无需浏览器渲染**，是接入成本最低的一类；唯一门槛是下载/高品需登录（`/ajax/dancedown`）。注意两个坑：**CDN 域名后双斜杠 `//` 必须保留**，以及**文件名含中文/空格、发请求前必须正确 percent-encode**（否则会连接级失败 `000`）。建议直连 **www** 而非 WAP。
