# JavaScript 爬虫旧 API 迁移

本文档适用于本次 QuickJS 爬虫 API 清理后的源代码，尚未对应已发布的模块版本；实际使用需搭配包含此异动的 QuickJS 模块与兼容 App；wrapper Java 与 JNI 必须来自相同来源，旧 JNI 不能搭配添加的对象绑定 API。建置方式见[运行环境模块](runtime-modules.md)。

清理依据是重复入口、隐含行为与旧协定的维护成本。方法较旧、名称较短或标准 API 能完成类似工作，都不足以单独构成移除理由。

## AI 迁移顺序

先读[替换清单](#替换清单)、[爬虫入口](#爬虫入口)、[完整 ID 集合](#完整-id-集合)与[验证](#验证)，再依来源使用的功能阅读其余段落。本文档是修改规则，不代表所有使用端都已完成迁移。

1. 从配置的 sites／lives 追查本地及远程入口、必要 helper、ext 格式与配套 App／wrapper／runtime；取不到的远程源代码列为未验证。区分 QuickJS 爬虫与 Node.js 工具，不把 require、process、Buffer 或 DOM 当成宿主能力。
2. 保存既有分类、筛选、搜索、列表 ID、每集播放值与登录流程，确认请求的正文、编码、Cookie、签名及错误处理。
3. 改为 default Spider 子类别与公开 catvod API，同步修改 helper 的 imports 及 vendor 路径；不导入 catvod:host 或自行重建 ctx.bind 桥接。
4. 沿调用链处理 Promise，核对完整 ids／vipFlags、搜索页码与分类 extend。不能只加 async 或只改方法名称；详情只处理 ids[0] 仍未完成迁移。
5. 依语意选 net.json／req、batch／map、Local／TTL；代理的 URL 产生端、接收参数与回传格式一起修改，不更动来源的签名、登录与字符编码规则。
6. 核对 Session、reader、WebSocket、弹幕停止回呼及 destroy 的资源释放，再依验证表逐项回报；Node.js 语法或单笔成功不代表全部入口与 Android 已通过。

共用字段与可选功能见 CatVodSpider 的 `docs/development.md` 与 QuickJS Demo。核对实作时，TV repo 的 `quickjs/src/main/assets/js/lib/catvod.js` 定义公开 SDK，`quickjs/src/main/assets/js/lib/net.js` 定义网络合约，`quickjs/src/main/java/com/fongmi/quickjs/crawler/Spider.java` 负责入口转接；爬虫仍只使用公开 API。

## 替换清单

| 旧写法 | 现行写法 | 移除原因与差异 |
| --- | --- | --- |
| 全域 `req/http/ws/connect` | `this.net.req/ws/connect` 或明确导入 `net` | 网络能力由 Spider 资源及模块提供 |
| 全域 `session/sleep/cached/clearCache` | `this.net.session/sleep/cached/clearCache` | 移除与 net 重复的入口 |
| 全域 `getCookie/setCookie/getUserAgent` | `this.net.getCookie/setCookie/getUserAgent` | 根网络仍使用 App 的登录信息与 UA；子 session 保持隔离 |
| 同步 `net.req()`、`net.http()` | `await this.net.req(url, options)` | HTTP 统一回传 Promise；子 session 与 batch 同样使用 req，不保留 http 别名 |
| `http/ws(url, {async, complete})` | `await this.net.req/ws(url, options)` | 一次请求只使用 Promise，不以 options 改变交付方式 |
| `evalJs(source, options)` | `await this.js.eval(source, options)` | 保留受管理的脚本运行 |
| `global/window/self` 别名 | `globalThis` | 使用标准全域对象名称；QuickJS 爬虫并未因此提供 DOM |
| `__jsEvalReturn()`、`__JS_SPIDER__`、default 对象／factory | `export default class extends Spider` | 统一为与 Java／Python 相近的类别入口 |
| `dialog({...options, onAction})`、`dialog.show(options, callback)` | `await this.dialog(options, callback)` | callback 是选填第二个参数，options 只放数据 |
| `js2Proxy()`、`proxy2()` 与 `from=catvod` 协定 | `this.proxyUrl(params)` 与 `proxy(params)` | URL、接收参数与回传数组一起修改，不再使用 Res |
| `getProxyUrl(params)`、`getProxy(local)` | `this.proxyUrl(params, local)` | 字符串键值参数；local 默认 true，false 使用局域网 IP，没有旧别名 |
| `getDanmakuUrl(id)` | `this.danmakuUrl(id)` | 由 App 加入站点识别，没有旧别名 |

`net.req/ws` 固定回传 Promise；旧 async／complete 字段不影响交付方式，也不会触发回呼，迁移时删除。可省略的 options 接受 null，视为空对象。`net.connect(url, options, listener)` 的 listener 用于持续 WebSocket 事件。

公开 API 从 `catvod` 模块导入。`hostReq`、`hostHttp` 等桥接不再注册为全域函数；底层函数绑定在原生对象，由内部 `catvod:host` 模块持有。爬虫派送与 Dialog callback 也由宿主持有引用，不公开 spider／dialogActions 全域对象。session 路由字段 netId 仍由宿主管理。

## 网络请求

JS 使用 `await net.req`，不再提供同步 HTTP。方法名称与 Java／Python 对齐；JS 的 Promise 交付方式仍保留语言本身的异步模型。时间单位为毫秒。

```javascript
import {net} from "catvod";

const response = await net.req(url, {
    params: {keyword: key, page: pg},
    headers,
    timeout: 5000
});
if (response.error) throw new Error(response.error);
if (response.code < 200 || response.code >= 300) throw new Error(`HTTP ${response.code}`);
const text = response.content;
```

调用端必须等待 Promise，不能直接读取 `net.req(...).content`。HTTP 非 2xx 仍保留回应，网络失败则带有 error；不要把登录页或错误正文当成成功列表。

只需要 JSON 时使用 `const data = await net.json(url, options)`；网络失败、HTTP 非 2xx 或无效 JSON 会抛出例外。一般请求默认回传文本，`buffer: 3` 回传 ArrayBuffer，可交给 Uint8Array、TextDecoder 或代理。

| buffer | content |
| --- | --- |
| `0` 或省略 | 文本，依回应编码解码 |
| `1` | 有号 byte 数组，值为 -128～127 |
| `2` | Base64 字符串 |
| `3` | ArrayBuffer；Python 对应 bytes |

buffer 只接受 0～3 的整数，net.json 使用文本模式。一般二进位用 3；若把 buffer: 1 的结果用于 body／proxy，先将每个值以 `value & 255` 转成 0～255。`buffer: 2` 仍可使用，与已移除的 proxy 第五格旗标不同。

需要区分 403 的会员限制、保留大整数 ID，或正文还需解密时，保留 req 与来源自己的解析流程。req 后使用 JSON.parse 本身不是过时写法，不把 HTTP 错误或解析失败改成成功空列表，也不把已登录流程改为访客请求。

| 请求情况 | 现行写法与注意事项 |
| --- | --- |
| URL query | options.params，保留来源既有编码及同名参数，不手动重复编码 |
| 表单 | `{method: "POST", postType: "form", data: fields}`；postType 默认 json，不能只搬 data |
| JSON | `{method: "POST", postType: "json", data: value}`；data 与 body 择一 |
| 原始正文 | `{method: "POST", body: text, headers: {"Content-Type": mime}}`；二进位 body 用 0～255 整数数组 |
| 二进位回应 | `await net.req(url, {buffer: 3})`，检查 error／code 后使用 content |
| 持续 WebSocket | `net.connect(url, options, listener)`；listener 处理事件，不是旧 complete callback |
| 每笔有解析或签名 | `await net.map(items, mapper, 4)`；mapper 接收 item、index，保留输入顺序 |
| 固定 HTTP 批量 | `await net.batch(requests, 4)`；每笔是 `{url, options}`，逐一检查结果 |

GET／HEAD 不传正文。buffer 只控制回应，不会把 JSON data 中的 Uint8Array 变成原始二进位正文。共用 net 没有逐次请求指定代理的 options；若来源另有此需求，保留其能力并列明缺口，不传入未支持的 proxy／proxies 字段当成已完成。

根网络的 Cookie API 仍对应 App 登录信息。需要独立 Cookie 与生命周期时创建子 session，使用完成后关闭。

```javascript
const client = this.net.session();
try {
    const data = await client.json(url, {headers});
    return {list: data.list};
} finally {
    client.close();
}
```

`await net.ws(url, options)` 是单次 WebSocket 交换，成功 code 为 101，收到首个完整消息后关闭。持续收讯使用 `net.connect(url, options, callback)`，立即回传 socket，callback 接收 `(socket, event)`；message 的 content 是文本或 Uint8Array。使用回呼传入的 socket，每次 open 重新订阅；reconnect:true 时在 retry 停止心跳并清除残包，停止或 destroy 时 close／cancel 连接。

## 爬虫入口

init 收到的 ext 一律是字符串。以下范例由来源解析包含 url、detailUrl 的 JSON；网址或其他字符串设置保留自己的解析流程。

```javascript
import {Spider} from "catvod";

export default class Source extends Spider {
    init(extend) {
        this.config = JSON.parse(extend);
    }

    async homeContent(filter) {
        const data = await this.net.json(this.config.url);
        return {class: data.categories};
    }

    async detailContent(ids) {
        const list = await this.net.map(ids, async id => {
            const data = await this.net.json(this.config.detailUrl + encodeURIComponent(id));
            return {vod_id: id, vod_name: data.name};
        });
        return {list};
    }
}
```

只接受继承 `Spider` 的 default class。constructor 保持同步；网络与其他异步初始化放在 `async init(extend)`，不需初始化时可省略 init。站点键由 `this.siteKey` 提供，资源使用 `this.net/local/js/dialog`；辅助模块可从 `catvod` 明确导入相同资源或便利函数。

| 旧方法 | 现行方法与参数 |
| --- | --- |
| `home(filter)` | `homeContent(filter)` |
| `homeVod()` | `homeVideoContent()` |
| `category(tid, pg, filter, extend)` | `categoryContent(tid, pg, filter, extend)` |
| `detail(id)` | `detailContent(ids)`，收到完整 ID 数组 |
| `search(key, quick, pg)` | `searchContent(key, quick, pg = "1")` |
| `play(flag, id, flags)` | `playerContent(flag, id, vipFlags)` |
| `live(url)` | `liveContent(url)` |
| `sniffer()` | `manualVideoCheck()` |
| `isVideo(url)` | `isVideoFormat(url)` |

其余公开入口为 `action(action)`、`proxy(params)`、`danmaku(id, emit)` 与 `destroy()`。action 回传操作结果，proxy 回传下节的代理数组，danmaku 回传停止函数；它们不是另一组需要改名的旧别名。

搜索的双参数入口会由宿主补上字符串页码 `"1"`，与 Python adapter 一致；显式传入的页码保持原值。detailContent 收到完整数组，playerContent 的 vipFlags 也保留完整数组，空数组不会省略。categoryContent 的 extend 是对象，参数顺序与 Java／Python 相同。

Python 入口目前使用同步 def，Java Spider 入口同步回传 String／boolean／Object[]；它们没有自动等待 coroutine 或 Future 的入口协定。JS 的派送层统一 await 同步值、Promise 与 thenable，再依入口交付数据字符串、代理数组或布尔值。异步模型依语言保留，对齐的是方法名称、参数及完成后的数据语意。

三语言的代理入口均为 `proxy(params)`。Dialog 使用 Java 的 `dialog(options, callback)`、Python 的 `self.dialog(options, callback)`、QuickJS 的 `await this.dialog(options, callback)`；callback 选填。Java init 额外置受 Android Context，JS／Python 的 init 只接收来源设置字符串。

init、上述入口、action、proxy、destroy 均可声明 async 或回传 Promise，宿主会等待结果。danmaku 可异步取得停止函数，停止函数本身也可回传 Promise。这不会让同步 CPU 计算自动平行运行；调用取消与期限仍由 Invocation 管理，constructor 和模块评估也受期限约束。

配置中的 ext 即使原本是 JSON 对象，到 init 仍是 JSON 字符串，不额外包成 `{stype, skey, ext}`。categoryContent 的 extend 才是筛选对象，不要对它重复 JSON.parse。类别方法绑定所属实例，可将 `this.method` 直接交给 map 或事件 callback，并保留相同函数引用；内部一般函数仍遵循 JS 的 this 规则。

数据入口直接回传对象或数组，由宿主串行化；旧 JSON 字符串改回对象。只有 liveContent 另接受 TXT／M3U 字符串。proxy 数组、manualVideoCheck／isVideoFormat 的布尔值不走数据串行化。批量详情的内部项目方法回传对象，合并各 list 后再交付。

串行化遵循 JSON.stringify：undefined／function 在对象中省略，在数组中转成 null；NaN／Infinity 也转成 null。BigInt、循环引用及尚未等待的 Promise 不能作为 JSON 结果。Python 不接受 NaN／Infinity，三语言共用数据应使用有效 JSON 值。

错误消息用字符串，例如 `{msg: String(error?.message || error)}`；不要直接放入 Error 对象，否则其消息通常会在 JSON 串行化时遗失。

Dialog 的 callback 收到 `{button, values}`，可回传更新对象或 Promise；最终回传 `{button, values}`，取消时为 null。callback 只从第二个参数注册，options 内的 callback／onAction 字段不会成为回呼。输入等待只暂停已让出运行权的调用期限，未 await 的 Dialog 不会停用正在运行 JS 的期限。

不再改写来源中的 `__JS_SPIDER__` 字符串，也不会依旧入口改变 req 的语意。

### 完整 ID 集合

把旧 detail(id) 的单笔请求与解析搬到 `async detailItem(id)`，保留签名、缓存、线路、集数及附加字段，回传含 list 的对象。入口处理完整集合：

```javascript
async detailContent(ids) {
    if (!ids.length) return {list: []};
    const responses = await this.net.map(ids, this.detailItem, 4);
    return {...responses[0], list: responses.flatMap(response => response.list)};
}
```

保留输入顺序，不自行去重；空集合不发请求。单笔失败让整次调用 reject，不捕捉后略过该 ID。范例沿用第一笔的顶层字段；若来源有批量消息语意，明确合并必要字段，不只取 list 而丢失信息。原 API 若直接支持批量 ID，优先使用其批量接口。

来源原本以空 list 与 msg 回报错误时，保留这个错误格式并让整个批量失败，不交付成功的部分当成完整结果。未上架或没有可播放内容等合法空结果可与其他详情合并，但须保留各笔提示，不能只沿用第一笔的 msg。

需要共用登录或依序变更状态时，以 for...of 逐笔 await，再合并结果。不要用未等待的 forEach(async ...) 或把 ids.map(async ...) 的 Promise 数组直接放进 list，也不以无上限 Promise.all 取代既有并行限制。

### 异步与生命周期

纯数据组装保留同步方法；调用 net.req／json、js.eval、dialog 或其他 Promise 时，await 或直接 return 该 Promise，并一路修改必要调用端。不得在同步 constructor 发出未受等待的初始化请求。

`await this.js.eval(source, {globals, timeout})` 在独立环境运行短段同步脚本，回传最后一个表达式的 JSON 值；globals 传 JSON 数据，timeout 默认 5000 毫秒，不沿用爬虫的变量、net 或模块。这里的 await 等待宿主运行完成，脚本结果本身不能是 Promise，也不能以 globals 覆盖内置 md5X。

只实作来源支持的功能，删除冗余覆写前核对 Spider 默认值。destroy hook 含 Promise 有 5 秒期限，调用前网络工作已取消；只释放自行创建的 Session、reader、连接、计时器或回呼引用，不在 destroy 安排新请求，不关闭共用站点 net。proxy 成功交付的 stream 由 App 接管，不能先关闭再回传。

## 代理

URL 产生端使用业务参数，不自行组合 do 或 siteKey。

```javascript
const url = this.proxyUrl({url: mediaUrl});
const lanUrl = this.proxyUrl({id: "a+b &=中文"}, false);
```

`proxyUrl(params, local = true)` 只接受字符串键值，省略 params 或传入 null 时视为空对象；local 只接受布尔。App 加入 do／siteKey 并编码 query，params 不可覆写它们。实例方法与从 catvod 导入的 proxyUrl 使用相同规则，不自行拼接路由。

`proxy(params)` 收到一个扁平的字符串键值对象，与 Java／Python 一致。HTTP 入口依序合并 query、headers、文件信息，后者覆写同名字段，do／siteKey 也会传入；query 已解码，不重复 decodeURIComponent。

回传 `[status, mime, body, headers]`，headers 可省略或为 null，额外字段忽略。status 是 100～599 的整数，MIME 是字符串；headers 的值用字符串或字符串数组，多值以 `, ` 合并并保留空白。HTTP server 管理 Content-Length／Transfer-Encoding，不转送来源的这两个 header。旧 proxy2 对象或 JSON 字符串改成此数组。

```javascript
async proxy(params) {
    const result = await this.net.stream(params.url);
    return [result.code, result.contentType, result.stream, result.headers];
}
```

此方法放在 default 导出的 Spider 子类别中。stream handle 只能交回目前代理调用；自行读取串流时改用 net.openStream，并在 finally 关闭 reader。

`await net.openStream(url, options)` 的 reader.read(size) 回传 `Promise<ArrayBuffer>`，EOF 为 null；size 默认 65536，范围 1～1048576，同一 reader 不并行读取。自订正文用 `await net.stream(asyncGenerator())`，每块输出 ArrayBuffer／TypedArray，不接受文本或普通数组，也不传 HTTP options。上游 reader 在产生器内打开并以 finally 关闭，不累积整段直播。

非串流 body 可为文本、0～255 整数数组、ArrayBuffer 或 TypedArray；TypedArray 按其实际 byte 范围传递，文本使用 UTF-8。

```javascript
return [200, "application/octet-stream", new Uint8Array([0, 127, 128, 255])];
```

第五格 Base64 旗标已移除；Base64 与 data URL 字符串都当成普通文本，不自动解码。需要二进位正文时先解码，再交付 bytes。

## 保留的便利方法与函数库

| 保留项目 | 保留原因 |
| --- | --- |
| `sha256X(text)` | 同步回传 SHA-256 hex；WebCrypto digest 是 Promise 与 ArrayBuffer，并非相同调用合约 |
| `randomX(size)` | 直接回传 URL-safe、无 padding 的随机字符串，size 是字节数；有效范围为 1..1024，超出范围仍回传空字符串 |
| `joinUrl(parent, child)` | 保留原 URI 解析语意，避免为改成 new URL 而改变既有来源行为 |
| `getPort()`、`proxyUrl(params, local)` | 取得代理连接端口或目前站点的本机／局域网代理 URL |
| `md5X/aesX/desX/rsaX` | 保留现有算法、模式与输入输出编码能力，标准 WebCrypto 不等价涵盖 |
| `s2t/t2s/openWeb/htmlQuery` | 仍有文本转换、App 与 HTML 截取用途 |
| `cheerio.min.js` | 保留 DOM 查找与操作能力，htmlQuery 不等价涵盖 |
| `crypto-js.js` | 提供共用哈希、加解密与二进位编码能力，来源不必携带副本 |
| `gbk.js` | 保留旧站搜索需要的 GBK URL 编码；TextDecoder 支持 GBK 解码，TextEncoder 仅提供 UTF-8 编码，不能直接取代 |

移除 `cat.js` 与 `similarity.js`：目前使用端没有引用，前者是多种工具的合并包，后者属于爬虫自己的字符串比对逻辑。仍使用这两个模块的外部爬虫，需改为自行提供相依函数库，或依实际功能迁移；不能只替换导入路径就视为兼容。

内置 JS 分为 `js/lib` 的宿主桥接与 `js/vendor` 的工具函数库。公开 SDK 仍以 `import {Spider} from 'catvod';` 使用；工具库改从 `assets://js/vendor/` 导入，原先的 `assets://js/lib/cheerio.min.js`、`crypto-js.js`、`gbk.js` 路径需同步调整。

红果的 CryptoJS 导入改为 `import 'assets://js/vendor/crypto-js.js';`，仍使用其 SHA-512、AES-CBC 与 Base64 能力。这是共用内置函数库，并非将签名与加密逻辑改成不同算法。

new URL、crypto.getRandomValues 与 crypto.subtle.digest 仍可直接使用，但不是要求所有爬虫迁移便利方法的理由。本次也保留 JAR 注入函数、类别方法的 this、冻结 prototype、弹幕释放与调用取消流程。

## Local 与 TTL 缓存

这部分使用既有共用 API，并非本次添加另一套缓存。local 保存站点隔离的 JSON 值，提供 get／set／delete，set／delete 不回传值；不要将 Local 设置当成会自动过期的缓存，也不要依赖旧成功／失败字符串或隐含 expiresAt 字段。

限时数据使用 `await net.cached(key, {ttl, stale, persist}, loader)`；ttl 必填，单位为毫秒，0 不缓存，stale 默认 true、persist 默认 false。播放 URL 或短期凭证应依来源期限明确设置，通常使用 `stale: false, persist: false`；失败应由 loader 抛出，不要伪装成空的成功结果。旧缓存数据不会自动转换成这个合约。

`peek(key)` 可能回传过期值；`refresh(key, options, loader)` 立即返回并背景更新；`clearCache(key)` 清单笔，省略 key 清整个站点。这些都位于 net，Session 不支持 persist。

## 验证

逐一盘点入口及 helper 中的旧入口名称、全域网络函数、net.http、__jsEvalReturn／__JS_SPIDER__、default 对象／factory、options.async／complete、js2Proxy／proxy2、getProxyUrl／getDanmakuUrl、dialog.show、第五格 Base64 旗标、旧 vendor 路径与 catvod:host。搜索命中只是定位，需阅读函数与必要调用端；保留便利函数及合法 JSON.parse 不列为失败。

| 检查 | 通过条件 |
| --- | --- |
| 语法与加载 | ES module 语法通过，配套 QuickJS 可加载 default Spider 子类别、helper 与 vendor，init 完成后才调用入口 |
| 详情集合 | 测 `[]`、`["a"]`、`["a", "b"]`、重复 ID 及单笔失败；核对顺序、完整字段与 reject 传递 |
| 参数 | 测 init 的 JSON／网址字符串、双参数搜索与显式页码、空／多笔 vipFlags、分类 extend 对象 |
| 异步 | 验证同步值、Promise 的完成顺序，纯数据与异步布尔接口、proxy、弹幕停止回呼均符合各自合约 |
| 列表与播放 | 分类／搜索两页、尾页、无结果与筛选正确；接通列表 ID → 详情 → 每集播放值 → 播放结果 |
| 网络与资源 | 正文格式、毫秒期限、非 2xx／登录错误、Cookie／Session、Local／TTL、Dialog、reader、取消与 destroy 符合来源规则 |
| 代理 | 测空参数、null、字符串键值、本机／区网 URL、headers、UTF-8 与含 0／128／255 的 bytes；第五格不解码，串流交接后可读取并由 App 关闭 |

完整清单才使用 page；外站已分页就保留当页，不再切片，有精确总笔数才填 total。记录配套 App、QuickJS 模块、wrapper／JNI 来源；旧模块缺少 API 时不以兼容分支掩盖配套错误。

完成回报列出已修改文件、保留分支及原因、实际运行的检查与未测项目。分开说明语法、QuickJS／Android 加载、采样详情／播放网址及实际设备播放；Node.js 或单笔成功不代表每个爬虫都已全面迁移。