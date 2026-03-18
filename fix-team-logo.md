---
description: 自动搜索、处理、上传缺失的球队 Logo
allowed-tools: Read, Write, Edit, Bash, WebSearch, WebFetch, AskUserQuestion, Task
---

你是球队 Logo 自动处理助手。根据用户输入，自动完成：搜索 Logo → 抠图 → 调整大小 → 压缩 → 上传更新。

$ARGUMENTS

---

# 搜索经验库（自我迭代）

**开始前必读**：`~/.claude/projects/-Users-mervin-galaxy-all/memory/team-logo-search.md`

这个文件记录了历次搜索的成功/失败经验、各来源的可靠性、各地区球队的搜索策略。搜索 Logo 时优先参考其中的经验，避免重复踩坑。

**结束后必更新**：每个球队处理完后（无论成功/失败），用 Edit 工具更新经验库：
- 成功 → 追加到「成功记录」表，如果发现新的有效来源则更新「搜索源优先级」和「按地区经验」
- 失败 → 追加到「失败记录」表，记录原因
- 踩坑 → 追加到「需要避免的坑」
- 新发现 → 更新对应的地区经验或图片格式经验

---

# 两种运行模式

- **模式 A（全自动）**：无参数 → 自动调 API 检测未来 7 天比赛中缺失 Logo 的球队（分 12 小时批次请求）
- **模式 B（指定文本）**：有参数 → 解析文本中的球队信息，仅处理指定球队

---

# Phase 0: 登录 & 环境准备

## 0.1 默认凭据

```
API_BASE=https://sport-dev.ra781.com/api/sport/admin
USERNAME=admin
PASSWORD=2!611VE1CPvCJr7J&..
```

直接使用以上默认值，无需询问用户。

## 0.2 登录

**重要**：密码含特殊字符（& 和 !），推荐用 Python requests 发请求，避免 shell 转义问题。

```python
import requests
resp = requests.post(f'{API_BASE}/login',
    json={'username': 'admin', 'password': '2!611VE1CPvCJr7J&..'},
    timeout=10)
d = resp.json()
token = d['token']  # { code: 200, token: "eyJ..." }
```

后续所有请求 header 加 `Authorization: {token}`（注意：不需要 Bearer 前缀）。

登录失败则停止并报错。

## 0.3 上传链路测试（首次运行）

登录成功后验证上传 API 可用：

```bash
# 用 Python 生成 1x1 透明 PNG 测试文件
python3 -c "
from PIL import Image
img = Image.new('RGBA', (1, 1), (0, 0, 0, 0))
img.save('/tmp/team_logo_test.png')
"

# 测试上传
UPLOAD_RESP=$(curl -s -X POST "$API_BASE/common/upload" \
  -H "Authorization: $TOKEN" \
  -F "file=@/tmp/team_logo_test.png;type=image/png")

rm -f /tmp/team_logo_test.png
```

验证返回 `code: 200` 且 `data.url` 有值。测试失败则停止并报错。

## 0.4 检测缺失 Logo

### 模式 A — 全自动

1. 分批调用赔率监控 API 获取未来 7 天比赛：

**关键**：
- `startTime` / `endTime` 必须是**字符串类型**，不是数字！传数字只返回极少比赛，传字符串才能拿到全部。
- 7 天数据量太大，必须**每 12 小时一批**请求，共 14 批，然后汇总去重。

```bash
# 用 Python 分批请求并汇总
python3 << 'PYEOF'
import json, subprocess, os
from datetime import datetime, timezone, timedelta

tz = timezone(timedelta(hours=8))
now = datetime.now(tz)
# 起点：当前时刻；终点：7 天后
range_start = now
range_end = now + timedelta(days=7)

API_BASE = os.environ.get("API_BASE", "https://sport-dev.ra781.com/api/sport/admin")
TOKEN = os.environ.get("TOKEN", "")

all_matches = []
batch = 0
cursor = range_start

while cursor < range_end:
    batch_end = min(cursor + timedelta(hours=12), range_end)
    batch += 1
    print(f"[批次 {batch}] {cursor.strftime('%m-%d %H:%M')} ~ {batch_end.strftime('%m-%d %H:%M')}")

    payload = {
        "page": 1,
        "size": 500,
        "startTime": str(int(cursor.timestamp() * 1000)),
        "endTime": str(int(batch_end.timestamp() * 1000)),
    }
    req_file = f"/tmp/monitor_req_{batch}.json"
    resp_file = f"/tmp/monitor_resp_{batch}.json"
    with open(req_file, "w") as f:
        json.dump(payload, f)

    result = subprocess.run([
        "curl", "-s", "-o", resp_file, "--max-time", "60",
        "-X", "POST", f"{API_BASE}/httpsse/data/oddsMonitor/list",
        "-H", f"Authorization: {TOKEN}",
        "-H", "Content-Type: application/json",
        "-d", f"@{req_file}"
    ], capture_output=True, text=True)

    try:
        with open(resp_file, "r") as f:
            resp = json.load(f)
        if resp.get("code") == 200 and isinstance(resp.get("data"), list):
            all_matches.extend(resp["data"])
            print(f"  -> 获取 {len(resp['data'])} 场比赛")
        else:
            print(f"  -> 响应异常: code={resp.get('code')}")
    except Exception as e:
        print(f"  -> 解析失败: {e}")

    # 清理批次临时文件
    os.remove(req_file)
    os.remove(resp_file)
    cursor = batch_end

# 按 matchId 去重
seen = set()
unique = []
for m in all_matches:
    mid = m.get("matchId")
    if mid not in seen:
        seen.add(mid)
        unique.append(m)

print(f"\n总计: {len(all_matches)} 场(去重后 {len(unique)} 场)")

# 保存汇总结果
with open("/tmp/monitor_all_matches.json", "w") as f:
    json.dump({"code": 200, "data": unique}, f)
PYEOF
```

后续解析统一读取 `/tmp/monitor_all_matches.json`。

2. 用 Python 解析比赛数据，检测每个球队 Logo 缺失：

**实际 API 响应格式**（`data` 直接是数组，不是嵌套对象）：
```json
{
  "code": 200,
  "data": [
    {
      "matchId": 29531,
      "startTime": 1773763200000,
      "homeTeamId": 35609,
      "homeTeamName": "Elana Torun",
      "homeTeamNameCN": "托伦",
      "homeTeamLogo": "",
      "awayTeamId": 162296,
      "awayTeamName": "Blekitni Stargard",
      "awayTeamNameCN": "斯塔加德什切青",
      "awayTeamLogo": "",
      "tournamentName": "III Liga, Group 2",
      "tournamentNameCN": "波兰足球丙级联赛",
      "status": "LIVE"
    }
  ]
}
```

从 `/tmp/monitor_all_matches.json` 读取汇总数据，遍历 `data` 数组中每场比赛，检查 `homeTeamLogo` 和 `awayTeamLogo`。

3. **全量 HTTP 可达性检测（必须！不能跳过！）**：

对**所有**有 Logo URL 的球队做 HTTP HEAD 检测，不能只检查字段是否为空。
后台前端通过 Image onError 回调检测加载失败的 Logo，我们必须用 HTTP 检测对齐这个逻辑。

```python
# 用线程池并发检测，20 线程约 1 分钟可检测 1500+ 个 URL
from concurrent.futures import ThreadPoolExecutor, as_completed

def check_logo(tid_info):
    tid, info = tid_info
    try:
        r = requests.head(info['logo'], timeout=8, allow_redirects=True,
                         headers={'User-Agent': 'Mozilla/5.0'})
        if r.status_code != 200:
            return tid, info, f'HTTP {r.status_code}'
        cl = r.headers.get('content-length', '')
        if cl and int(cl) == 0:
            return tid, info, 'Empty (0 bytes)'
    except:
        return tid, info, 'Error'
    return tid, None, None

with ThreadPoolExecutor(max_workers=20) as executor:
    # ... 并发检测所有有 URL 的球队
```

缺失判断标准：
- `!logo` → 无 Logo URL
- HTTP HEAD 返回非 200 → URL 失效
- content-length 为 0 → 空文件

4. 去重后生成缺失列表，格式同前端 Modal：
```
联赛名  ID:123  2025-03-18 20:00  主队名  主队名 vs 客队名
```

### 模式 B — 文本输入

解析用户提供的文本，每行格式：
```
联赛名  ID:123  2025-03-18 20:00  球队名  主队名 vs 客队名
```
正则：`^(.+?)\s+ID:(\d+)\s+(\S+\s+\S+)\s+(.+?)\s+(.+?)\s+vs\s+(.+)$`

## 0.5 获取球队完整信息

无论哪种模式，都需要调 team/list 获取球队详细数据（编辑时需回传所有字段）：

```bash
TEAM_DATA=$(curl -s -X POST "$API_BASE/data/team/list" \
  -H "Authorization: $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"id\":\"$TEAM_IDS\",\"page\":1,\"size\":100}")
```

其中 `$TEAM_IDS` 是逗号分隔的 ID 列表（如 `"123,456,789"`）。

保存每个球队的完整字段，后续编辑时只替换 logo。

---

# Phase 1: 搜索 Logo（逐个球队处理）

对每个缺失 Logo 的球队：

1. **提取英文名**：从 team/list 返回的 `languages` 数组中取 `type="en-US"` 的 `name`
2. **搜索优先级**（按顺序尝试）：
   - **官网**：WebSearch 搜 `"{英文球队名}" official site`，WebFetch 访问官网提取 header/logo 图片 URL
   - **TheSportsDB API**：`curl -s "https://www.thesportsdb.com/api/v1/json/3/searchteams.php?t=球队名"` → 取 `teams[0].strBadge`
   - **WebSearch 图片搜索**：搜 `"{英文球队名}" "{联赛名}" team logo transparent png`
   - **WebFetch 提取**：访问搜索结果中的 Logo 网站，提取实际图片 URL
3. **注意区分同名球队**：验证搜索结果中的球队确实是目标球队（对比联赛名）
4. **下载图片**：

```bash
curl -sL -o "/tmp/team_logo_${TEAM_ID}_raw.png" "$IMAGE_URL"
```

**支持 webp 格式**：如果下载的是 webp，Pillow 可以直接打开处理。

如果搜索多轮仍无合适结果，标记该球队为"失败"并继续下一个。

---

# Phase 2: 抠图（如需要）

用 Python + Pillow 检测是否已有透明背景：

```bash
python3 -c "
from PIL import Image
img = Image.open('/tmp/team_logo_${TEAM_ID}_raw.png').convert('RGBA')
extrema = img.getextrema()
has_transparency = extrema[3][0] < 128  # alpha 通道最小值 < 128
print('transparent' if has_transparency else 'opaque')
"
```

**如果不透明**，用 rembg 抠图：

```bash
python3 -c "
from rembg import remove
from PIL import Image
import io

input_img = Image.open('/tmp/team_logo_${TEAM_ID}_raw.png')
output_img = remove(input_img)
output_img.save('/tmp/team_logo_${TEAM_ID}_nobg.png')
"
```

> 首次使用需安装：`pip3 install rembg`（会下载约 170MB 模型）。
> 如果安装失败，提醒用户手动安装，或跳过抠图步骤让用户手动处理。

抠图后的文件：`/tmp/team_logo_{teamId}_nobg.png`
已透明的直接复制：`cp /tmp/team_logo_{teamId}_raw.png /tmp/team_logo_{teamId}_nobg.png`

---

# Phase 3: 调整大小（200x200 画布）

用 Pillow 将图片调整为 200x200 画布，内容约 180px 居中：

```bash
python3 -c "
from PIL import Image

img = Image.open('/tmp/team_logo_${TEAM_ID}_nobg.png').convert('RGBA')
img.thumbnail((180, 180), Image.LANCZOS)

canvas = Image.new('RGBA', (200, 200), (0, 0, 0, 0))
x = (200 - img.width) // 2
y = (200 - img.height) // 2
canvas.paste(img, (x, y), img)
canvas.save('/tmp/team_logo_${TEAM_ID}_resized.png')
"
```

---

# Phase 4: 压缩

用 pngquant 压缩（已安装在 `/usr/local/bin/pngquant`）：

```bash
pngquant --quality=65-80 --strip --force \
  "/tmp/team_logo_${TEAM_ID}_resized.png" \
  -o "/tmp/team_logo_${TEAM_ID}_final.png"
```

如果 pngquant 不可用，直接用 resized 文件作为 final。

---

# Phase 5: 用户确认

对每个处理完的球队 Logo：

1. 用 **Read 工具**展示 `/tmp/team_logo_{teamId}_final.png` 让用户预览图片
2. 显示信息：
   - 球队名（中英文）
   - 球队 ID
   - 所属联赛
   - 图片来源 URL
   - 文件大小
3. 用 **AskUserQuestion** 让用户选择：
   - **确认上传** — 继续 Phase 6
   - **跳过** — 跳过此球队
   - **重新搜索** — 回到 Phase 1 用不同关键词搜索

---

# Phase 6: 上传 & 更新

## 6.1 上传图片到 GCS

```bash
UPLOAD_RESP=$(curl -s -X POST "$API_BASE/common/upload" \
  -H "Authorization: $TOKEN" \
  -F "file=@/tmp/team_logo_${TEAM_ID}_final.png;type=image/png")
```

从响应中提取 `data.url`（相对路径如 `/profile/upload/2026/03/18/xxx.png`）。

**重要**：上传返回的是相对路径，team/edit 后端会自动拼接 GCS 前缀 `https://storage.googleapis.com/n3-dev-sport-assets/`，无需手动拼接。

## 6.2 更新球队 Logo

从 Phase 0.5 保存的球队数据中取必要字段，替换 `logo`：

```bash
curl -s -X POST "$API_BASE/data/team/edit" \
  -H "Authorization: $TOKEN" \
  -H "Content-Type: application/json" \
  -d "$TEAM_EDIT_JSON"
```

`$TEAM_EDIT_JSON` 需要写入文件（`-d @file`），包含以下字段：
```json
{
  "id": "123",
  "logo": "上传返回的相对路径",
  "languages": [...],
  "sportId": 1,
  "displayStatus": 0,
  "weight": 0,
  "tournamentIds": [...],
  "searchWord": [...]
}
```

**注意**：`id` 是字符串类型，不是数字。`name`/`nameCN`/`shortName` 不传也不会被清空。
```

## 6.3 清理临时文件

```bash
rm -f /tmp/team_logo_${TEAM_ID}_*.png
```

---

# 最终汇总

所有球队处理完后，输出汇总：

```
=== Logo 处理完成 ===
成功: 巴塞罗那(ID:123), 利物浦(ID:456)
跳过: 曼联(ID:789)
失败: 拜仁(ID:101) - 未找到合适的 Logo
```

---

# 本地工具依赖

| 功能 | 工具 | 说明 |
|------|------|------|
| 搜索图片 | WebSearch + WebFetch | Claude Code 内置 |
| 抠图 | `rembg` Python 包 | 首次需 `pip3 install rembg` |
| 调整大小 | `python3 + Pillow` | 已安装 |
| 压缩 | `pngquant` | 已安装 `/usr/local/bin/pngquant` |

---

# 注意事项

1. **逐个处理**：每次只处理一个球队，确认后再处理下一个
2. **错误容忍**：单个球队处理失败不影响其他球队
3. **Token 过期**：如果 API 返回 401，自动重新登录
4. **图片质量**：优先选择高清、透明背景的官方 Logo
5. **同名球队**：通过联赛名+比赛时间验证，避免选错同名球队的 Logo
6. **shell 特殊字符**：密码、JSON 数据等含 & ! 等字符时，用 Write 写入文件再 -d @file 传参
7. **大响应处理**：监控 API 响应可达 500KB+，必须 `-o file` 存文件再解析，不能 pipe
8. **调色板透明度**：P 模式图片可能自带透明度，转 RGBA 后再检测 alpha 通道
9. **webp 支持**：Pillow 直接支持打开 webp 文件
