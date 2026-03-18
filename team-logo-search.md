# 球队 Logo 搜索经验库

每次 `/fix-team-logo` 运行后自动更新此文件，记录搜索经验供下次参考。

## 搜索源优先级（按成功率排序）

1. **TheSportsDB API** — `https://www.thesportsdb.com/api/v1/json/3/searchteams.php?t=球队名` → `teams[0].strBadge`，适合有一定知名度的球队。注意球队名拼写变体！
2. **Transfermarkt CDN** — `https://tmssl.akamaized.net/images/wappen/head/{TM_ID}.png`，覆盖面最广，包括极小的俱乐部都有。需要先搜索获取 TM ID
3. **Wikipedia API** — 检查球队 Wikipedia 页面的 images 列表，筛选含 logo/crest/badge 的文件名
4. **iconape.com** — `https://iconape.com/{slug}-logo-logo-icon-svg-png.html`，PNG 在 `/wp-content/files/` 下
5. **footylogos.com** — 大型联赛球队透明 PNG，CDN URL 格式 `https://cdn.prod.website-files.com/...`
6. **球队官网** — 最可靠但耗时，Logo 常在 `/wp-content/uploads/` 路径下

## 缺失检测策略（重要！）

**必须全量检测**，不能只看字段是否为空：
1. 字段为空/null → 缺失
2. 有 URL 但 HTTP HEAD 返回非 200 → 缺失
3. 有 URL 但 content-length=0 → 缺失

后台前端 (odds-monitor-v2) 的检测方式是 `!team.logo || failedLogos.has(team.logo)`，
它通过 Image 的 onError 回调捕获加载失败的 URL。
我们的脚本也必须用 HTTP HEAD 检测所有有 URL 的球队，用线程池并发（20线程）可以在1分钟内检测完1500+个。

## 搜索策略最佳实践

### 名称变体生成
搜不到时必须尝试变体：
- 去掉前缀：FK/FC/KF/AS/US/CD/ASD/USD/GKS/KS/BTS/OFK/TJ
- 去掉后缀：Reserve/U19/U23/FC/SC
- 城市名替代：如 "Klithorps Toun" → "Cleethorpes Town"
- 本地拼写：如 "OFK Belgrade" → "OFK Beograd"，"Besa Kavaje" → "Besa Kavajë"

### Transfermarkt 搜索（万能兜底）
- 搜索页面：`https://www.transfermarkt.us/schnellsuche/ergebnis/schnellsuche?query=球队名`
- 从 HTML 中提取 `/verein/(\d+)` 获取 ID
- Logo URL：`https://tmssl.akamaized.net/images/wappen/head/{ID}.png`
- **覆盖率极高**：连菲律宾联赛、马里联赛、巴西州级联赛的球队都有
- 图片通常 139x181 RGBA PNG

### TheSportsDB 批量搜索技巧
- 第一轮用原始英文名搜索
- 第二轮用名称变体（去前缀、去后缀、本地拼写）
- `lookupteam.php?id=` 接受 TheSportsDB 自己的 ID，不是我们的 teamId

## 按地区/联赛的经验

### 波兰 (Poland)
- 波兰球队通常有自己的官网，格式多为 `球队名.pl`
- ✅ Elana Torun → 官网 `elanatorun.com`
- ✅ Błękitni Stargard → 官网 `blekitni.stargard.pl`
- ✅ Pniówek Pawłowice → TheSportsDB（搜 "Pniowek Pawlowice"）
- ✅ Rekord Bielsko-Biala → iconape.com
- ✅ Starovice → Transfermarkt（TM ID: 134036）

### 阿尔巴尼亚 (Albania)
- TheSportsDB 覆盖不错
- ✅ Laçi, Luftëtari, Kukësi → TheSportsDB 第一轮
- ✅ Besa Kavajë, Apolonia Fier, Korabi Peshkopi → TheSportsDB 第三轮（需名称变体）

### 塞尔维亚 (Serbia)
- ✅ Čukarički, Voždovac, Grafičar → TheSportsDB
- ✅ OFK Belgrade → TheSportsDB（搜 "OFK Beograd"）

### 菲律宾 (Philippines)
- TheSportsDB 不覆盖
- **Transfermarkt 是唯一来源**
- ✅ Mendiola FC 1991 (TM:27359), Tuloy FC (TM:100532), Manila Digger FC (TM:106490), Davao Aguilas (TM:59419)

### 非洲 (Mali, Cameroon, Kenya, Botswana)
- TheSportsDB 覆盖极少
- **Transfermarkt 是主要来源**
- ✅ US Bougouni (TM:81770), Derby Academie (TM:78820), AS Fortuna Mfou (TM:25372)
- ✅ PWD Bamenda → TheSportsDB
- ✅ Unisport de Bafang → Wikipedia（JPG 90x90）
- ✅ Nairobi United (TM:108005), Lerumo Lions (TM:125314)

### 美国 (US Open Cup)
- 小俱乐部 TheSportsDB 不覆盖
- **Transfermarkt 有效**
- ✅ West Chester United (TM:46343), Azteca FC (TM:59798), San Ramon FC (TM:72981), Northern Virginia FC → TheSportsDB

### 意大利 (Serie D)
- TheSportsDB 覆盖不错
- ✅ Pistoiese, Lentigione, Varesina, Castellanzese → TheSportsDB

### 大型俱乐部
- ✅ Inter Milan, Sporting CP, Red Star Belgrade → footylogos.com CDN（1500x1500 高清）
- ✅ Real Madrid, FK IMT → Wikipedia
- ✅ Benfica → TheSportsDB

## 需要避免的坑

| 问题 | 说明 |
|------|------|
| **API startTime/endTime 必须是字符串** | OddsMonitorReqVo 中定义为 String，传数字只返回极少比赛 |
| **密码特殊字符** | `2!611VE1CPvCJr7J&..` 含 `&!`，必须写 JSON 文件再 `curl -d @file` |
| **API 响应过大** | 监控 API 响应 500KB+，必须 `curl -o file` 保存后再解析 |
| worldsoccerpins 同名混淆 | 搜 "US Bougouni" 返回了 "US Bougouba"，必须验证 Logo 上的文字 |
| seeklogo.com 下载需特殊 URL | 直接 curl seeklogo 页面返回 HTML，需 WebFetch 提取真实下载链接 |
| sofascore/flashscore CDN 403 | 这两个站的 Logo CDN 直接请求返回 403 |
| Wikipedia Commons-logo.svg | 搜索 Wikipedia images 时会返回 "File:Commons-logo.svg"，必须过滤 |
| SVG 转换困难 | macOS 上 cairo 库可能架构不兼容（x86 vs arm64），SVG→PNG 转换可能失败 |
| TheSportsDB ID ≠ 我们的 teamId | lookupteam.php 用的是 TheSportsDB 自己的数据库 ID |
| team/edit 需传完整字段 | 缺失字段会被置 null，从 team/list 获取完整数据仅替换 logo |
| upload 返回相对路径 | 上传返回 `/profile/upload/...`，后端 team/edit 自动加 GCS 前缀 |
| TheSportsDB/TM 图片可能为 0 字节 | 有些 URL 返回 HTTP 200 但 content 为空，必须检查 `len(content) > 100` |
| API 路径变更 | 2026-03 起 API 前缀从 `/admin/` 改为 `/api/sport/admin/`，team/list 响应用 `content` 而非 `list` |
| 用 Python requests 替代 curl | 密码特殊字符用 Python `requests.post(json=...)` 最可靠，避免 shell 转义 |

## 图片处理经验

- Transfermarkt: 139x181 RGBA PNG，需 resize
- TheSportsDB: 通常 512x512 透明 PNG
- footylogos.com: 1500x1500 调色板 PNG，需 convert('RGBA')
- Wikipedia: 尺寸不一，可能是 JPG（需 rembg 抠图）或 PNG
- iconape.com: 600x600 RGBA PNG
- 处理流程: thumbnail(180,180) → 居中贴到 200x200 透明画布 → pngquant 压缩

## 批量处理统计

### 第三次批量（2026-03-18 第三轮，全量检测后补漏）
| 来源 | 成功数 | 说明 |
|------|--------|------|
| Transfermarkt | 11 | 包含新出现的比赛球队 |
| TheSportsDB | 5 | 含缅甸、印度等小联赛 |
| **总计** | **16** | 16/19 新缺失球队 |

全量检测 1636 个有 URL 球队 → 0 个失效。仅 19 个无 URL（处理后剩 3 个）。
未找到：Lemense FC SP, FC Mali Coura, BOHFS St. Louis

### 第二次批量（2026-03-18 第二轮）
| 来源 | 成功数 | 说明 |
|------|--------|------|
| Transfermarkt | 107 | 含名称变体深搜，覆盖最广 |
| TheSportsDB | 24 | 3 轮搜索 + 名称变体 |
| **总计** | **131** | 131/136 = 96.3% 成功率 |

### 第一次批量（2026-03-18 第一轮）
| 来源 | 成功数 | 说明 |
|------|--------|------|
| TheSportsDB | 27 | 3 轮搜索，名称变体很重要 |
| Transfermarkt | 14 | 万能兜底，覆盖最广 |
| Wikipedia | 8 | 包含直接搜索和 API 搜索 |
| footylogos.com | 3 | 大俱乐部高清 Logo |
| iconape.com | 2 | Digenis Morphou, Rekord |
| 球队官网 | 3 | Elana Torun, Błękitni Stargard, AS Real Bamako |
| **总计** | **57** | 57/57 = 100% 成功率 |
