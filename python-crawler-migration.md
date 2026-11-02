# Python 爬虫旧 API 迁移

本文档适用于本次移除旧 Python API 后的源代码。这些异动尚未对应已发布的模块版本；实际使用需搭配包含此异动的 Python 模块与兼容 App，建置方式见[运行环境模块](runtime-modules.md)。爬虫仍使用 CPython 3.10，入口维持同步 `def`。

## AI 迁移顺序

先读[替换清单](#替换清单)、[爬虫入口](#爬虫入口)、[完整 ID 集合](#完整-id-集合)与[验证](#验证)，再依来源使用的功能阅读其余段落。本文档是修改规则，不代表所有使用端都已完成迁移。

1. 从配置的 sites／lives 追查本地及远程入口、必要 helper、ext 格式与配套 App／模块；取不到的远程源代码列为未验证。先区分 App 爬虫与独立配置产生工具；后者不注入 self.net，不改造成 Spider。
2. 保存既有分类、筛选、搜索、列表 ID、每集播放值与登录流程，确认请求的正文、编码、Cookie、签名及错误处理。
3. 依替换清单修改已移除 API，保留来源业务逻辑；不添加同名兼容层，也不直接依赖 app.py 或 runtime 的内部方法。
4. 核对每个入口的参数与回传，特别是 detailContent 的完整 ids、playerContent 的 vipFlags，以及搜索默认字符串页码 "1"。传入正确但使用端只读 ids[0]，仍未完成迁移。
5. 仅在语意相同时改用 net.json、net.batch／map、Local 或 TTL 缓存；保留需要 HTTP 状态、原始正文与来源指定代理的分支。
6. 核对 Session、reader、WebSocket、弹幕停止回呼及自建资源的释放，不关闭共用站点 net。最后以验证表逐项回报，未测项目明列，不以语法或单笔成功宣称全面完成。

共用字段与可选功能见 CatVodSpider 的 `docs/development.md` 与 Python Demo。核对实作时，TV repo 的 `chaquo/src/main/python/base/spider.py` 定义基底，`chaquo/src/main/python/app.py` 负责入口转接，`chaquo/src/main/python/runtime/net.py` 定义网络合约；这些是阅读依据，不是爬虫应导入的内部 API。

## 替换清单

| 旧写法 | 现行写法 | 差异 |
| --- | --- | --- |
| `self.fetch()`、`self.post()` | `self.net.req()`、`self.net.json()` | 使用宿主注入的网络，不再回传 requests.Response |
| `self.str2json()`、`self.json2str()` | `json.loads()`、`json.dumps()` | 明确 import json；入口直接回传 dict／list |
| `getCache()`、`setCache()`、`delCache()` | `self.local` 或 `self.net` 的 TTL 缓存 | 依持久化设置或限时数据选择；这三个旧方法已在先前共享 API 改版移除 |
| 共用 `cache/py`、`SourceFileLoader` 加载 | `self.loadModule()`、`self.loadSpider()` | 方法保留，只使用站点隔离的 Runtime |
| `Runner`、`Trigger` | 继承 `base.spider.Spider`，由 App 调用 | 旧转接档已移除，不自行包装宿主桥接 |
| `getName()` | 移除覆写，站点名称使用配置的 name | 不再是宿主入口，也没有基底默认方法 |
| `self.getProxyUrl(params)` | `self.proxyUrl(params)` | 字符串键值参数；local 默认 True，没有 getProxyUrl 旧别名 |
| `self.getProxyUrl(False)` | `self.proxyUrl(local=False)` | 布尔位移至 local，False 使用局域网 IP |
| `localProxy(param)` | `proxy(params)` | 三语言统一入口，接收扁平字符串键值集合 |
| `self.getDanmakuUrl(id)` | `self.danmakuUrl(id)` | 由 App 加入站点识别，没有旧别名 |
| `self.dialog.show(options, callback)`、options 内的 `onAction` | `self.dialog(options, callback)` | callback 是选填第二个参数，options 只放数据 |
| `self.html(content)` | `lxml.etree.HTML(content)` | 自行 import lxml，不再经基底转调用 |
| `self.regStr(reg, src, group)` | `re.search()` 后取得群组 | 调用端保留找不到时回传空字符串等规则 |
| `self.removeHtmlTags(src)` | 解析 HTML 后截取文本 | 原方法只用正则删除标签，与解析 HTML 的结果可能不同 |
| `self.cleanText(src)` | 来源自己的 `re` 过滤规则 | 原方法只删除部分 Emoji 区段，没有通用文本清理合约 |

`getDependence()`、`loadModule()`、`loadSpider()` 仍可使用。上述四个 HTML／文本工具已从基底移除，lxml 与标准库 re 仍可直接 import。不要添加同名旧方法或捕捉 AttributeError 后改走另一套兼容流程。

宿主桥接集中在 `runtime.loader`、`runtime.net`、`runtime.dialog`、`runtime.format`，取代原先的 `_tv_*` 模块名称。Dialog 直接注入为 `self.dialog`，使用 `self.dialog(options, callback)`；callback 收到 `{button, values}`，同步回传更新 dict 或 None。最终回传 `{button, values}`，取消时为 None，错误向外抛出。options 内的 callback／onAction 字段不会注册回呼。

`base` 只保留爬虫使用的 `spider.py`。公开入口与共用 API 使用 `homeContent`、`proxyUrl` 等 camelCase 名称，宿主内部 helper 采 snake_case；爬虫不直接依赖 `runtime` 或 `app.py` 的内部函数名称。

原先注入的 `_tv_runtime` 字段改为 `runtime`，由宿主管理加载与释放。来源不覆写此字段，依赖与子爬虫仍通过 `loadModule()`／`loadSpider()` 使用。

## 爬虫入口

入口类别维持 `from base.spider import Spider as BaseSpider` 与 `class Spider(BaseSpider)`，由 App 注入资源后调用同步 `def`。需要初始化时覆写 `init(self, extend="")`，否则可省略；App 不等待 coroutine，入口不使用 async def。

以下参数均省略 self；公开名称维持原有 camelCase，自订 helper 使用 snake_case，私有 helper 可保留前置底线。

| 入口 | 参数与回传 |
| --- | --- |
| `init(extend="")` | extend 是来源设置字符串，需要 JSON 时由来源解析；回传 None |
| `homeContent(filter)` | filter 是 bool；回传含 class、可选 filters 的 dict |
| `homeVideoContent()` | 回传含 list 的 dict |
| `categoryContent(tid, pg, filter, extend)` | pg 保留调用值，extend 是筛选 dict；回传当页 list 与分页字段 |
| `detailContent(ids)` | ids 是完整 ID list；回传含完整详情 list 的 dict |
| `searchContent(key, quick, pg="1")` | quick 是 bool；双参数调用默认 pg="1"，回传当页 list 与分页字段 |
| `playerContent(flag, id, vipFlags)` | id 是每集播放值，vipFlags 是完整 list，含空 list；回传播放 dict |
| `liveContent(url)` | 回传直播 dict／list，或原有 TXT／M3U 字符串 |
| `proxy(params)` | params 是字符串键值 dict；回传 `[status, mime, body, headers]`，headers 可省略 |
| `manualVideoCheck()`、`isVideoFormat(url)` | 回传 bool，不回传 JSON 字符串 |
| `action(action)` | action 是操作字符串；回传操作结果 dict／list |
| `danmaku(id, emit)` | 用 emit(item) 发送数据，回传可调用的停止函数；不回传 coroutine |
| `destroy()` | 释放自行创建的资源；回传 None |

配置中的 ext 即使原本是 JSON 对象，到 init 仍是 JSON 字符串，需要时由来源 json.loads；categoryContent 的 extend 已是筛选 dict，不重复解析。只覆写来源需要的入口；基底默认首页空 class、列表空 list、播放／操作空 dict、直播空字符串、代理 404 与空正文。

## 网络请求

`self.net` 在 `init()` 前由 App 注入。请求使用 options 字典，时间单位为毫秒；可省略的 options 接受 None，视为空 dict，与 JS 的 null 一致。

| 原 requests 用法 | 迁移方式 |
| --- | --- |
| `response.json()` | 直接使用 `self.net.json(url, options)` |
| `response.text` | `self.net.req(url, options)["content"]`，先检查 error 与 code |
| `response.content` | options 设 `buffer: 3`，成功回应的 content 为 bytes |
| `response.status_code` | `response["code"]`；网络错误先检查 error |
| `response.headers` | `response["headers"]`；重复 header 的值可能为数组 |
| `params={...}` | options 的 `params`；保留既有 query，字典值可用数组发送同名参数，None 会略过 |
| `timeout=5` | `timeout: 5000`；整次请求期限另用 `callTimeout` |
| `allow_redirects=False` | `redirect: 0`；默认跟随重新导向 |
| `cookies={...}`、requests.Session | 使用 Cookie API 或 `self.net.session()` |
| `stream=True` | `self.net.openStream()`；以 with／finally 关闭 reader |
| 固定 HTTP 批量 | `self.net.batch(requests, 4)`，每笔为 `{"url": url, "options": options}`，逐一检查 error／code |
| 每笔有解析或签名 | `self.net.map(items, mapper, 4)`；mapper 接收 item、index，保留输入顺序 |
| `verify=False` | 没有同名 options；TLS 行为由宿主网络层决定，不原样搬入此参数 |

新 `timeout` 默认为 10000 毫秒，与旧 fetch／post 的 5 秒不同。需要维持旧期限时明确传 5000；`timeout` 管理连接／读写，`callTimeout` 管理整次请求。

### GET 与 JSON

只需要 JSON 时直接取得解析结果。网络失败、HTTP 非 2xx 或无效 JSON 会抛出例外，不再额外调用 `.json()` 或 `json.loads()`。

```python
data = self.net.json(url, {
    "params": {"keyword": key, "page": pg},
    "headers": headers,
    "timeout": 5000,
})
```

需要状态码、错误正文或文本时使用 req。HTTP 非 2xx 仍保留回应；不要把登录页或错误伪装成成功空列表。

例如需区分 403 的会员限制，或正文还需解密时，保留 req 与来源自己的解析流程。req 后使用 json.loads 本身不是过时写法，不能只因回应是 JSON 就机械替换成 net.json。

```python
response = self.net.req(url, {"headers": headers, "timeout": 5000})
if response.get("error"):
    raise RuntimeError(response["error"])
if not 200 <= response["code"] < 300:
    raise RuntimeError(f"HTTP {response['code']}")
text = response["content"]
```

req 默认回传文本，编码依 HTTP 回应处理，不再由基底强制指定 UTF-8。需要自行解码时使用 `buffer: 3`，检查成功后对 bytes 调用 `decode()`。

| buffer | content |
| --- | --- |
| `0` 或省略 | str，依回应编码解码 |
| `1` | 有号 byte 的整数 list，值为 -128～127 |
| `2` | Base64 字符串 |
| `3` | bytes；QuickJS 对应 ArrayBuffer |

buffer 只接受 0～3 的整数，net.json 使用文本模式。一般二进位用 3；若把 buffer: 1 的结果用于 body，先将每个值以 `value & 255` 转成 0～255，作 proxy 正文则先组成 bytes。`buffer: 2` 仍可使用，与已移除的 proxy 第五格旗标不同。

### POST 与正文

旧 `post(data={...})` 通常送出表单，新网络 API 的 `postType` 默认为 json。迁移表单时必须明确指定 form。

```python
data = self.net.json(url, {
    "method": "POST",
    "postType": "form",
    "data": {"name": name, "page": pg},
    "headers": headers,
    "timeout": 5000,
})
```

| 正文格式 | options |
| --- | --- |
| JSON 值 | `method: "POST", postType: "json", data: value` |
| 表单 | `method: "POST", postType: "form", data: {...}` |
| multipart 文本字段 | `method: "POST", postType: "form-data", data: {...}` |
| 已编码文本 | `method: "POST", body: text`，并提供 Content-Type |
| 原始 bytes | `method: "POST", body: list(payload)`，并提供 Content-Type |

`data` 与 `body` 互斥；GET／HEAD 不接受正文。form／form-data 的 data 必须是对象，字段经共用网络层转字符串并去除前后空白。需要重复表单字段或精确保留文本时，自行编码后以 body 发送；form-data 此处只提供文本字段，不对应 requests 的 files 参数。

```python
response = self.net.req(url, {
    "method": "POST",
    "headers": {"Content-Type": "application/octet-stream"},
    "body": list(payload),
    "buffer": 3,
})
```

`buffer` 只控制回应格式；原始二进位正文仍须转成 0～255 整数数组。req 的网络与 HTTP 错误检查方式与 GET 相同。

### Cookie、Session 与生命周期

`self.net.getCookie(url)`／`setCookie(url, cookie)` 读写 WebView Cookie，写入接受单笔 Set-Cookie 格式；请求设 `cookie: True` 才会自动带入与更新。App 的 User-Agent 可由 `self.net.getUserAgent()` 取得，再按来源需要放入 headers，不会自动套用到所有请求。

需要隔离 Cookie 时创建 Session，请求会自动使用该 Session 的 Cookie，并在结束时关闭。

```python
with self.net.session() as session:
    session.json(login_url, {
        "method": "POST",
        "postType": "form",
        "data": credentials,
    })
    data = session.json(data_url)
```

Session 关闭只释放自己的请求、Cookie 与缓存，不关闭父站点。站点的 self.net 由 App 管理；子爬虫不要自行关闭共用 self.net。串流以 `reader.read(size)` 取得 bytes，结束时回传 None，reader 也须以 with／finally 关闭。

`self.net.ws(url, options)` 是单次 WebSocket 交换，成功 code 为 101，收到首个完整消息后关闭。持续收讯使用 `self.net.connect(url, options, callback)`，立即回传 socket，callback 接收 `(socket, event)`；message 的 content 是 str 或 bytes。使用回呼传入的 socket，每次 open 重新订阅；reconnect: True 时在 retry 停止心跳并清除残包，停止或 destroy 时 close／cancel 连接。

共用 net 目前没有逐次请求指定代理的 options。来源若明确支持配置代理，不能直接删除既有 requests 分支，或把 proxies 塞进 options 当成已迁移；保留其设置、错误与 Session 释放，并列出尚未由共用 API 涵盖的能力。既有登录 Cookie／token 也不能在迁移时改为访客流程。

## Local 与 TTL 缓存

小型持久化设置使用 Local，Python 直接存 JSON 值，不先串行化成字符串。

```python
self.local.set("preferences", {"category": "movie", "page": 1})
preferences = self.local.get("preferences", {})
self.local.delete("preferences")
```

Local 按 Site.key 隔离；不存在时回传 None，或 get 的第二参数。set／delete 回传 None，调用时的例外会向外传递，不能再比较旧的 `"succeed"`／`"failed"`。App 清缓存不会删除 Local，改 Site.key 后也不会自动沿用原站点数据。

旧 `/cache` 与 Local 是不同数据空间，不会自动搬移。Local 不解析 `expiresAt`，即使字段时间已过仍原样回传；不要直接搬入旧数据并假定具有过期判断。重要设置需由来源明确处理迁移或通过配置重新提供，不加入全域旧缓存 fallback。

需要期限的数据使用 cached，由 loader 产生 JSON 值。

```python
catalog = self.net.cached(
    "catalog",
    {"ttl": 60000, "stale": False},
    lambda: self.net.json(catalog_url),
)
```

- ttl 为必填毫秒数；0 不缓存。stale 默认 True，可能先回过期值再背景更新；需要等待新数据时设 False。
- persist 默认 False；只为可公开持久化的数据激活，Session 不支持 persist。Cookie、token、签名播放网址不写入长期 Local 或持久缓存。
- `self.net.peek(key)` 可读到过期值，不能拿来取代有效期判断。`self.net.clearCache(key)` 清单笔，省略 key 清整个站点缓存。
- `self.net.refresh(key, options, loader)` 立即返回并背景更新；`self.net.cachedText(key, options, loader)` 回传同项目的 JSON 文本，loader 仍回传 JSON 数据，每次调用仍检查 TTL。
- cached 不是 setCache 的同参数替代；保存设置用 Local，限时数据才交给 loader 与 TTL。

## 依赖与子爬虫

保留 loadModule 回传的模块对象，不再加载后用 `importlib.import_module("helper")` 找全域同名模块，也不手动修改 sys.path、sys.modules 或共用 cache/py。

```python
def init(self, extend=""):
    self.helper = self.loadModule("helper")
    self.child = self.loadSpider("child")
    self.child.init(extend)
```

loadModule／loadSpider 的名称相对于入口爬虫来源解析，模块与文件缓存按站点、来源及加载世代隔离。loadSpider 不会自动调用子爬虫 init；在父爬虫 init 中加载时，子爬虫共用目前站点的 net、local、js 与 Dialog，不创建另一份站点资源。

`self.js.eval(source, {"globals": values, "timeout": 5000})` 同步回传最后一个表达式的 JSON 值，需要配套 App 与 QuickJS 模块。每次运行环境独立，不沿用爬虫的变量、net 或模块；数据以 globals 传入，不拼接到代码，也不能覆盖内置 md5X。timeout 为正整数毫秒，默认 5000；脚本结果不能是 Promise。

getDependence 仍可声明依赖。若使用一般 import 且名称可能与全域模块碰撞，在 getDependence 声明后于 init 中 import，或直接采用 loadModule。

```python
def getDependence(self):
    return ["helper"]

def init(self, extend=""):
    import helper
    self.helper = helper
```

不要直接建构脱离 App Runtime 的爬虫，再期待 loadModule／loadSpider 从旧共用目录加载。Runner／Trigger 不再使用；以 `from base.spider import Spider as BaseSpider` 导入基底并定义 `class Spider(BaseSpider)`，由 App 完成加载、注入、调用与释放。

## JSON 与入口回传

只在解析正文或创建指定格式的请求时使用标准库 JSON。

```python
import json

data = json.loads(text)
body = json.dumps(data, ensure_ascii=False)
```

首页、分类、搜索、详情、播放与操作结果直接回传 dict／list，由宿主统一串行化，不回传额外 json.dumps 后的字符串。仅在解析正文或创建请求时使用标准库 json，不依赖宿主内部 helper。JSON 结果不接受 NaN／Infinity 或未完成的 coroutine。

错误消息使用 `{"msg": str(error)}`，不直接回传 Exception 对象；dict 内的值同样必须能串行化成 JSON。

布尔接口回传 bool，proxy 回传 `[status, mime, body, headers]`；只有 liveContent 另接受 TXT／M3U 字符串，不套用数据入口的 JSON 包装。

### 完整 ID 集合

旧实作若只处理 ids[0]，先把单笔解析搬到 `_detail(self, video_id)`，把其中对 ids[0] 的使用换成 video_id，并保留既有请求、签名、缓存、线路、集数与附加字段。helper 回传含 list 的 dict，再由入口合并：

```python
def detailContent(self, ids):
    if not ids:
        return {"list": []}
    responses = [self._detail(video_id) for video_id in ids]
    return {**responses[0], "list": [item for response in responses for item in response["list"]]}
```

保留输入顺序，不自行去重；空集合不发请求，单笔错误向外传递，不捕捉后略过该 ID。范例沿用第一笔的顶层字段；有批量消息或其他来源语意时明确处理，不只取 list 而丢失必要字段。原 API 若直接支持批量 ID，优先使用其批量接口。

来源原本以空 list 与 msg 回报错误时，可在整个批量外层保留这个错误格式，失败时不交付已累积的部分结果。未上架或没有免费集数等合法空结果可与其他详情合并，但须保留各笔提示，不能只沿用第一笔的 msg。

默认依序处理。各笔解析确定不修改共用登录、Session 或暂存状态时，才改用 `self.net.map(ids, lambda video_id, index: self._detail(video_id), 4)`；mapper 接收 item、index 两个参数。固定请求可用 net.batch，每笔仍需检查 error 与 code。不以无上限线程或 asyncio 取代此合约。

## HTML 与文本处理

HTML 解析直接使用 lxml，并处理空内容无法创建树的情况。

```python
from lxml import etree

html_text = "<div>你好 &amp; <b>世界</b></div>"
root = etree.HTML(html_text)
plain_text = "" if root is None else "".join(root.itertext())
```

这也可取代需要截取文本的 removeHtmlTags 用法。解析器会处理 `&amp;` 等实体，与原本 `re.sub('<.*?>', '', src)` 的结果不完全相同；script／style 内容与段落空白需按来源需求处理。

正则截取由来源明确指定群组及无匹配结果，以下保留旧 regStr 找不到时回传空字符串的行为。

```python
import re

source = "ID:123"
match = re.search(r"ID:(\d+)", source)
value = match.group(1) if match else ""
```

cleanText 的过滤范围由来源决定。确实需要原有行为时，可直接使用以下明确规则；它只涵盖旧方法的四段 Unicode 范围，不是完整的 Emoji 过滤。

```python
import re

title = "电影😀"
pattern = re.compile('[\U0001F600-\U0001F64F\U0001F300-\U0001F5FF\U0001F680-\U0001F6FF\U0001F1E0-\U0001F1FF]')
title = pattern.sub("", title)
```

## 代理网址

第一个参数传入业务参数集合，是否使用本机地址以 local 指定。

```python
local_url = self.proxyUrl({"id": "a+b &=中文"})
lan_url = self.proxyUrl({"id": "a+b &=中文"}, local=False)
lan_base = self.proxyUrl(local=False)
```

`proxyUrl(params=None, local=True)` 只接受字符串键值 dict，不自动转型；省略 params 或传入 None 时视为空 dict，local 只接受 bool。App 自动加入 do／siteKey 并编码 query，params 不可覆写它们，不自行拼接路由。

`proxy(params)` 收到一个扁平字符串键值 dict，与 Java／JS 一致。HTTP 入口依序合并 query、headers、文件信息，后者覆写同名字段，do／siteKey 也会传入；query 已解码，不重复 unquote。

回传 `[status, mime, body, headers]`，headers 可省略或为 None，额外字段忽略。status 是 100～599 的整数，MIME 是字符串；body 可为文本、bytes、bytearray 或 InputStream。headers 的值用字符串或字符串串行，多值以 `, ` 合并并保留空白。HTTP server 管理 Content-Length／Transfer-Encoding，不转送来源的这两个 header。

```python
def proxy(self, params):
    result = self.net.stream(params["url"])
    return [result["code"], result["contentType"], result["stream"], result["headers"]]
```

成功交付后由 App 关闭 stream；不交付时自行关闭。分块读取使用 self.net.openStream，放在 with／finally 中；reader.read(size) 回传 bytes，EOF 为 None，size 默认 65536，范围 1～1048576，同一 reader 不并行读取。自订正文用 self.net.stream(generator())，每块输出 bytes／bytearray／连续 memoryview，不传 HTTP options；reader 在产生器内打开并释放，不累积整段直播。

第五格 Base64 旗标已移除；Base64 与 data URL 字符串都当成普通文本。需要二进位正文时先解码成 bytes，不添加旧旗标处理。

## 验证

先逐一盘点入口与必要依赖中的 fetch／post、str2json／json2str、旧 Cache API、html／regStr／removeHtmlTags／cleanText、getName、localProxy、getProxyUrl／getDanmakuUrl、dialog.show、第五格 Base64 旗标、Runner／Trigger、_tv_*、SourceFileLoader 与手动 sys.path 修改。搜索命中只是定位，必须阅读所属函数；正常的 json.loads、私有 helper 底线及保留 API 不列为失败。

| 检查 | 通过条件 |
| --- | --- |
| 语法与加载 | Python 3.10 下 py_compile 通过，配套 App 可加载、注入与 init；入口没有 coroutine |
| 详情集合 | 测 `[]`、`["a"]`、`["a", "b"]`、重复 ID 及单笔失败；核对顺序、完整字段与错误传递 |
| 参数 | 测 init 的 JSON／网址字符串、双参数搜索与显式页码、空／多笔 vipFlags、分类 extend dict |
| 列表与播放 | 分类／搜索两页、尾页、无结果及筛选正确；逐一接通列表 ID → 详情 → 每集播放值 → 播放结果 |
| HTTP 与登录 | 表单／JSON／原始 bytes、毫秒期限、非 2xx、登录页与格式错误符合来源规则，不变成成功空列表 |
| 资源与隔离 | 实际使用的 Cookie、Session、Local／TTL、依赖、Dialog、proxy 与停止／destroy 流程保留隔离及释放 |
| 代理 | 测空参数、None、字符串键值、本机／区网 URL、headers、UTF-8 与含 0／128／255 的 bytes；第五格不解码，串流交接后可读取并由 App 关闭 |

完整清单才使用 self.page；外站已分页就保留当页，不再切片，有精确总笔数才填 total。App、Python 模块与来源须配套；主机适配器通过不代表 Android 注入、缓存或取消已通过。

完成回报列出已修改文件、保留分支及原因、实际运行的检查与未测项目。分开说明语法、App 加载、采样详情／播放网址及实际设备播放；没有检查所有适用入口与必要依赖，不宣称每个爬虫都已全面迁移。
