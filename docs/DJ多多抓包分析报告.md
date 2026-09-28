# DJ多多（com.shoujiduoduo.dj）HTTPS 抓包分析报告

> 抓包时间：2026-08-17 23:00~23:26  
> 设备：Redmi M2103K19C（Android 13, Magisk 30.7 root）  
> 方法：iptables 透明拦截 + mitmproxy 解密

---

## 一、应用基本信息

| 项目 | 值 |
|------|-----|
| 包名 | `com.shoujiduoduo.dj` |
| 主 Activity | `com.duoduo.dj.MainActivity` |
| UID | 10211 |
| 版本 | 5.5.68 |
| 应用 ID | 6IUO2B |
| 加固方案 | 梆梆加固（StubApp 壳） |
| 插件框架 | libMEOW |
| 广告 SDK | 穿山甲(字节) v5568、快手 v4.6.30.1、友盟、腾讯 GDT/Bugly |

---

## 二、核心业务域名（按功能分类）

### 2.1 核心 API 服务器

| 域名 | 用途 | 关键接口 |
|------|------|----------|
| **new.dianyinduoduo.com** | ⭐ 主 API 服务器 | config, time, get_pkg_map, hot_v2 |
| api.dianyinduoduo.com | 旧版 API（可能废弃） | — |
| hscdn.dianyinduoduo.com | CDN / 资源分发 | 封面/图片等静态资源 |

### 2.2 音频资源链路（关键发现）

| 域名 | 用途 | 协议 |
|------|------|------|
| **down.bibdj.com** | ⭐ 音频下载入口（302 重定向服务） | HTTP 302 → OSS |
| **teambition-file.oss-cn-zhangjiakou.aliyuncs.com** | ⭐ 真实音频文件存储（阿里云 OSS） | HTTP 206 (Partial Content, audio/mpeg) |

**音频播放完整链路：**
```
app 请求歌曲列表 → new.dianyinduoduo.com (act=hot_v2)
                    ↓ 返回含 down.bibdj.com URL 的歌曲数据
app 请求音频 → down.bibdj.com//d//team//cs//mycs//{date}/{filename}.mp3
              ↓ HTTP 302 重定向 (Location: 阿里云 OSS URL)
app 下载音频 → teambition-file.oss-cn-zhangjiakou.aliyuncs.com/work/{path}/{file}.mp3
                                                         ?OSSAccessKeyId=LTAI...&Expires=...&Signature=...&tb-file-token=JWT...
              ↓ HTTP 206 Partial Content, Content-Type: audio/mpeg
              ↓ MP3 音频流开始播放
```

**OSS URL 特征：**
- 临时签名 URL（Expires 时间戳约 10 天有效期）
- `tb-file-token` 参数为 JWT 格式（包含 fileName、fileSize、scope 等元信息）
- 支持 206 Partial Content（可 seek/拖动进度条）

### 2.3 日志与统计

| 域名 | 用途 |
|------|------|
| log.ergeduoduo.com | 业务行为日志批量上报 (POST act=bulk) |
| cnlogs.umeng.com | 友盟统计（启动/推送事件） |
| errnewlog.umeng.com | 友盟错误日志 |
| tracking.miui.com | 小米系统追踪 |

### 2.4 广告 SDK 域名（非核心）

| 域名 | SDK |
|------|-----|
| api-access.pangolin-sdk-toutiao.com | 穿山甲/字节广告（get_ads） |
| open.e.kuaishou.com | 快手广告（univ） |
| api.bjzghd.com | 广告竞价（设备/广告位/出价 JSON） |
| adsmind.ugdtimg.com / pgdt.ugdtimg.com | 优量汇/GDT 广告素材 |
| sdk.e.qq.com / win.gdt.qq.com 等 | 腾讯广点通 |

---

## 三、核心 API 接口详解

### 3.1 new.dianyinduoduo.com 接口列表

所有接口使用 **GET** 方法，参数通过 Base64 编码的 `data=` 字段传递（内部为加密二进制，非纯文本）。

#### (1) 应用配置
```
GET /?act=config&data={Base64加密参数}
Host: new.dianyinduoduo.com
```

#### (2) 服务端时间同步
```
GET /?act=time&data={Base64加密参数}
Host: new.dianyinduoduo.com
```
> 用于防篡改的时间戳校验

#### (3) 包名映射表
```
GET /?act=get_pkg_map&data={Base64加密参数}
Host: new.dianyinduoduo.com
```

#### (4) ⭐ 热歌列表（首页数据源）
```
GET /?act=hot_v2&data={Base64加密参数}
Host: new.dianyinduoduo.com
```
> 返回热歌榜单数据（歌名、歌手、时长、播放量、音频 URL 等）

#### (5) 会话验证
```
GET /?Code={数字ID}
Host: new.dianyinduoduo.com
```
> 可能用于会话/token 校验

### 3.2 音频下载接口

#### bibdj 302 入口
```
GET //d//team//cs//mycs//{YYYYMMDDHHmm}/{URL编码的文件名}.mp3
Host: down.bibdj.com
→ 302 Location: {阿里云 OSS 签名 URL}
```

**URL 路径结构：**
- `/d/team/cs/mycs/` — 固定路径前缀
- `{YYYYMMDDHHMM}` — 日期时间目录（20260817010 = 2026-08-17 01:10）
- `{filename}.mp3` — URL 编码的中文文件名

**已确认的音频文件示例：**
1. `Dj聪仔-全粤语Electro音乐港派潮性男女情歌对唱串烧.mp3`
2. `全粤语ElertcoHouse专辑随缘选别急冻思维独飚嗨车载DJParts.mp3`
3. `Dj7索-沫仔为好基开善仔精选嘻哈夜雨蹦迪慢摇串烧2015.mp3`

#### 阿里云 OSS 实际存储
```
GET /work/{YYYY/MM/DD}/{uuid}/{timestamp}.mp3?OSSAccessKeyId={}&Expires={}&Signature={}&tb-file_token={JWT}
Host: teambition-file.oss-cn-zhangjiakou.aliyuncs.com
→ 206 Partial Content, Content-Type: audio/mpeg; charset=UTF-8
```

### 3.3 日志上报接口

```
POST /
Content-Type: application/x-www-form-urlencoded
Body: act=bulk&data={大量Base64编码的行为数据}
Host: log.ergeduoduo.com
```

---

## 四、请求特征总结

### 4.1 API 设计模式
- **操作码模式**: `act={config|time|get_pkg_map|hot_v2|bulk}`
- **参数加密**: `data=` 字段为 Base64 包装的加密二进制（非明文 JSON）
- **身份认证**: `Code={数字ID}` 或 Cookie/Token（未在 URL 中明文暴露）

### 4.2 音频分发架构
- **入口域名**: down.bibdj.com（302 跳转）
- **实际存储**: 阿里云 OSS（张家口区域）
- **鉴权方式**: OSS 临时签名 URL + JWT token
- **协议优化**: 206 Partial Content 支持边下边播/seek

### 4.3 反逆向措施
- 梆梆加固壳（dex 运行时解密）
- API 参数加密传输（非明文）
- SSL/TLS 证书固定（推测，但 mitmproxy 在系统 CA 注入后可解密）

---

## 五、第三方 SDK 完整清单

| SDK | 域名关键词 | 用途 |
|-----|-----------|------|
| 穿山甲(字节) | pangolin-sdk-toutiao, pglstatp-toutiao, bytedapm, byteimg | 广告展示 |
| 快手广告 | kuaishou.com, adukwai, kskwai | 广告展示 |
| 友盟统计 | umeng.com (cnlogs/errnewlog/ucc/utoken/audid) | 统计分析 |
| 腾讯广点通/GDT | gdt.qq.com, qzs.gdtimg.com, sdk.e.qq.com | 广告展示 |
| 腾讯 Bugly | bugly.qq.com | 崩溃上报 |
| 小米系统 | xiaomi.com, miui.com, micloud.net, hyperos.developer.xiaomi.com | 系统服务/推送/云服务 |
| MEOW 插件框架 | (代码层, 无独立域名) | 插件化架构 |

---

## 六、下一步建议

1. **深入分析 hot_v2 接口返回数据**: 抓取完整响应该解构歌曲对象（ID、标题、歌手、时长、封面 URL、音频 URL 字段）
2. **搜索接口**: 操作搜索功能捕获 `act=search` 或类似接口
3. **播放详情接口**: 点单首歌曲后可能有 `act=song_detail` 或 `act=get_play_url` 接口
4. **用户体系**: 登录/注册接口（如有）、用户收藏/历史记录接口
5. **bibjd.com 站点分析**: down.bibdj.com 是 302 跳转，直接访问 www.bibdj.com 可能获取更多音源信息
6. **dex 内存提取**: 从运行时 `/proc/PID/mem` 提取梆梆壳解密后的 dex（需 root），进一步反编译分析内部逻辑

---

*报告生成时间：2026-08-17 23:26*
*抓包工具链：iptables REDIRECT → fifo nc 中继 → adb reverse → host_relay.py → mitmproxy upstream → tunnel.py*
