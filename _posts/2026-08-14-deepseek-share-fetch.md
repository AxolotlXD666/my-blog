---
title: DeepSeek 分享对话抓取方法
date: 2026-08-14 21:30:00 +0800
tags: [技术, Python, DeepSeek]
---

# DeepSeek 分享对话抓取方法

## 问题

DeepSeek 的分享链接（`https://chat.deepseek.com/share/<对话ID>`）打开后是一个纯前端页面（React SPA），对话内容在运行时才通过接口加载。直接抓取 HTML 只能拿到 `og:description` 里被截断的一小段摘要，拿不到完整对话。

## 思路

SPA 的数据一定来自某个后端接口。所以方法很简单：**找到那个接口，直接调用它**，绕过页面渲染。

## 步骤

### 1. 找到前端 JS 包

打开分享页，查看页面源码里引用的脚本，例如：

```html
<script src="https://fe-static.deepseek.com/chat/static/main.69ea66451d.js"></script>
```

（这个哈希文件名可能随版本变化，以页面实际引用为准。）

### 2. 在 JS 包里搜索 API 路径

下载 `main.js`，用正则搜出所有带 `share` 的接口路径：

```python
import re, requests

js = requests.get("https://fe-static.deepseek.com/chat/static/main.69ea66451d.js").text
for m in sorted(set(re.findall(r'["\'](/api/[^"\']*share[^"\']*)["\']', js))):
    print(m)
```

输出（节选）：

```text
/api/v0/share/content
/api/v0/share/create
/api/v0/share/delete
/api/v0/share/fork
/api/v0/share/list
```

### 3. 调用内容接口

`/api/v0/share/content` 就是拿对话内容的接口，参数是 `share_id`：

```python
import requests

share_id = "对话分享ID"  # 即 URL 中 share/ 后面那段
headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0",
    "Accept": "application/json, text/plain, */*",
    "Referer": f"https://chat.deepseek.com/share/{share_id}",
}
r = requests.get(
    f"https://chat.deepseek.com/api/v0/share/content?share_id={share_id}",
    headers=headers, timeout=15,
)
data = r.json()
```

### 4. 解析返回结果

返回的 JSON 里，对话在 `data.biz_data.messages`，每条消息含 `role`（`USER` / `ASSISTANT`）和 `content`：

```python
for m in data["data"]["biz_data"]["messages"]:
    print("=" * 50)
    print(f"[{m['role']}]")
    print(m.get("content", ""))
```

## 完整脚本

```python
# fetch_deepseek_share.py
# 用法：python3 fetch_deepseek_share.py <分享ID>
import sys, requests

share_id = sys.argv[1]
headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0",
    "Accept": "application/json, text/plain, */*",
    "Referer": f"https://chat.deepseek.com/share/{share_id}",
}
r = requests.get(
    f"https://chat.deepseek.com/api/v0/share/content?share_id={share_id}",
    headers=headers, timeout=15,
)
data = r.json()
if data.get("code") != 0:
    print("请求失败：", data.get("msg"))
    sys.exit(1)
for m in data["data"]["biz_data"]["messages"]:
    print("=" * 50)
    print(f"[{m['role']}]")
    print(m.get("content", ""))
```

## 注意点

- **Referer 头要带上**，请求头不完整可能被拒绝。
- 参数是查询参数 `share_id`，不是路径参数。
- `code: 0` 表示成功；对话在 `data.biz_data.messages`。
- 页面 `og:description` 里有对话开头摘要，可以先快速预览，再决定要不要抓全量。
- 分享出去的对话本来就是公开的，此方法只是读取公开数据，不涉及任何隐私绕过。
