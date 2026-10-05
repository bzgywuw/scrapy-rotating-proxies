# scrapy proxy: 如何为你的爬虫接入轮换住宅代理，使其在第 1 万页之后依然存活

你的蜘蛛在 200 个页面上运行良好。然后你把它指向 10,000，大概在第三个几千页左右，响应就会开始变得奇怪：空的正文、`403` 响应，以及每十二个请求中就有一个抛出的 `TCP connection timed out`。本地什么都修不好，因为问题不在解析器。

代理能修好其中一部分。剩下的问题——中间件顺序、封禁检测、代理认证——才是大多数 Scrapy 代理配置悄悄死掉的地方，日志里除了`closespider_itempassed`之外没有任何明显线索。这篇文章将按你会遇到它们的顺序，逐一介绍实际配置。

## 代理解决什么问题，以及它解决不了什么

值得先说清楚，因为“使用代理”经常被当成万灵药来推销。

爬虫被屏蔽通常出于三个相互独立的原因：

- **IP 声誉。** 来自云服务 ASN 的请求，在它被读取之前就已经被评定了风险。
- **请求指纹。** TLS 握手、header 一致性、cookie 行为。现代反机器人系统会综合数十个信号来评估请求，而 IP 只是其中一个。
- **速率和模式。** 来自一个 IP 的同一端点请求太多，或并发太高。

代理只解决第一个问题。如果你的 header 在声明 Chrome/120 时却说 `Accept-Encoding: identity`，或者没有 cookie jar，一个干净的住宅 IP 也无济于事。所以这个顺序是：先修好请求，再添加代理，然后测量。

## 在 Scrapy 中附加代理的三种方式

### 1. 逐请求通过 `meta`

最快，而且适合先证明凭证确实有效。

python
import scrapy

class ProxyTestSpider(scrapy.Spider):
    name = "proxy_test"
    start_urls = ["https://httpbin.org/ip"]

    def start_requests(self):
        proxy = "http://YOUR_LOGIN__cr.us:YOUR_PASSWORD@gw.dataimpulse.com:823"
        for url in self.start_urls:
            yield scrapy.Request(url, meta={"proxy": proxy}, dont_filter=True)

    def parse(self, response):
        self.logger.info(response.json()["origin"])


如果 `httpbin.org/ip` 回显的是代理的 IP 而不是你的 IP，那么凭证和路由都是正常的。如果它卡住然后因超时结束，那么端点或者登录信息就是错的——直接说，认证出问题比你希望的要常见得多。

### 2. 通过 settings 和内置中间件全局设置

Scrapy 自带的 `HttpProxyMiddleware` 会读取 `proxy` 元数据键，并且还支持来自 `http_proxy` / `https_proxy` 环境变量的代理。对于容器化的爬虫，注入环境变量是让凭证不进入代码的最干净方式。

bash
export https_proxy="http://YOUR_LOGIN__cr.us:YOUR_PASSWORD@gw.dataimpulse.com:823"


**注意：** 环境变量方式会让你拿到一个端点，所以它适合单个轮换网关，而不适合轮换代理列表。

### 3. 通过自定义中间件，统一选择代理

一旦你有多个爬虫，硬编码的代理字符串就会在每个文件里重复出现。要在 `middlewares.py` 中集中管理：

python
from urllib.parse import quote

class ProxyMiddleware:
    def __init__(self, proxy_url):
        self.proxy_url = proxy_url

    @classmethod
    def from_crawler(cls, crawler):
        url = crawler.settings.get("PROXY_URL")
        if not url:
            raise NotConfigured("PROXY_URL is required")
        return cls(proxy_url=url)

    def process_request(self, request, spider):
        if "proxy" not in request.meta:
            request.meta["proxy"] = self.proxy_url


然后在 `settings.py` 中注册它，并从环境变量读取凭证——不要从 Python 文件读取：

python
import os

PROXY_URL = os.environ.get("PROXY_URL")

DOWNLOADER_MIDDLEWARES = {
    "myproject.middlewares.ProxyMiddleware": 350,
}


如果你的密码包含 `@`、`:` 或 `/`，在把它放进 URL 之前要用 `quote(password, safe="")` 转义。未转义的特殊字符是 `407` 的一个常见原因，它会让你误以为问题出在代理服务商身上。

## 中间件顺序是大多数 Scrapy 代理配置的致命伤

下载器中间件按固定顺序运行。请求钩子按优先级从低到高执行；响应和异常钩子按从高到低执行。

Scrapy 内置的优先级值（Scrapy 2.17）：

| 中间件 | 优先级 |
| --- | --- |
| RobotsTxtMiddleware | 100 |
| HttpAuthMiddleware | 300 |
| DownloadTimeoutMiddleware | 350 |
| DefaultHeadersMiddleware | 400 |
| UserAgentMiddleware | 500 |
| RetryMiddleware | 550 |
| RedirectMiddleware | 600 |
| CookiesMiddleware | 700 |
| HttpProxyMiddleware | 750 |
| DownloaderStats | 850 |
| HttpCacheMiddleware | 900 |

除了看这张表，你还需要记住三件事：

- **不要禁用 `HttpProxyMiddleware`。** 有些教程会告诉你在设置成 `None`，理由是“我的自定义中间件已经处理了代理”。你的自定义中间件就是个选择器，它并不负责传输。禁用它以后，`Proxy-Authorization` 头就不会被附加，请求就会静默地以未认证状态发出，或者直接失败。
- **不要把它设为 750。** 和 `HttpProxyMiddleware` 使用同一个优先级，会产生一个由字典顺序决定的竞态条件，而不是由你的逻辑决定。你的代理行为就会变得时断时续。
- **重试会复制 `meta`，包括 `proxy`。** `RetryMiddleware` 会克隆失败的请求。如果你的选择器使用 `setdefault()` 或 `if "proxy" not in request.meta`，那个过期的代理就会继续留在重试请求上。你可以显式检查 `request.meta.get("retry_times", 0) > 0`，然后主动覆盖它。

## 轮换网关 vs 代理列表

有两种模式，它们适用于不同的任务。

**列表模式** 使用 `scrapy-rotating-proxies` 包。安装它，声明你的端点，启用两个中间件：

python
# settings.py
ROTATING_PROXY_LIST_PATH = "proxies.txt"

DOWNLOADER_MIDDLEWARES = {
    "rotating_proxies.middlewares.RotatingProxyMiddleware": 610,
    "rotating_proxies.middlewares.BanDetectionMiddleware": 620,
}


该包会把失效的代理标记为失效，按随机指数退避重新检查它们，并通过 `ROTATING_PROXY_BACKOFF_BASE` 和 `_CAP` 进行调优。它还有一个副作用值得了解：当你启用 `RotatingProxyMiddleware` 后，`DOWNLOAD_DELAY`、`CONCURRENT_REQUESTS_PER_DOMAIN` 和自动节流设置会变成 *按代理* 生效，而不是按域名生效。

**网关模式** 提供了单个端点，在服务商侧轮换出口 IP。在 `ROTATING_PROXY_LIST` 中只放一行即可，两层仍然可以叠加：网关在每个请求上给出新的住宅 IP，中间件在标记到封禁时用同一个网关重试。

对于 DataImpulse，网关端点是：

- `gw.dataimpulse.com:823` — HTTP/HTTPS，每个请求轮换
- `gw.dataimpulse.com:824` — SOCKS5，每个请求轮换
- 粘性会话在 `10000–20000` 范围内获取端口，持续 1 到 120 分钟，默认 30 分钟

认证以 `login:password` 形式放在 URL 中，国家定向后接在用户名上——`YOUR_LOGIN__cr.us` 用于美国出口，这个后缀是 URL 参数，而不是仪表盘里的开关。也就是说，你的爬虫可以通过直接修改字符串，从同一个网关切换国家。

对于大多数爬虫而言，网关模式是更简单的可靠模式。只有在你真正需要并行管理多个服务商或池时，才使用列表模式。

👉 [Get started with DataImpulse's rotating gateway at $1/GB](https://bit.ly/dataimPulse)

## 验证轮换是否真的在发生

不要假设。测试它。用一个访问 IP 回显端点的爬虫：

python
class IPCheckSpider(scrapy.Spider):
    name = "ipcheck"
    start_urls = ["https://httpbin.org/ip"] * 10

    def parse(self, response):
        self.logger.info("Exit IP: %s", response.json()["origin"])


运行 `scrapy crawl ipcheck`，然后读取日志。你应当看到多个不同的地址。如果每一行都显示同一个 IP，那么要么中间件没有启用，要么代理列表为空，要么请求是从 HTTP 缓存中返回的，根本没经过网络。最后这一种比它应有的情况更常见，因为 `HttpCacheMiddleware` 的优先级是 900，恰好位于代理层之后。

## 封禁检测和 407 错误

轮换问题几乎都可以归结为两个问题：中间件是否真的能识别封禁，以及代理认证是否正确。

`scrapy-rotating-proxies` 默认启发式规则很粗糙——任何不是 200 的状态码、响应体为空或异常，都会被当作已失效代理。这在目标站点到处返回 404 时会过度封禁，在站点用 200 状态码返回验证码页面时又封禁不足。要子类化 `BanDetectionPolicy`，并对外层状态码做出明确判断，而不是靠猜。

值得按信号区分：

- **`407` 是代理认证失败。** 通用请求重试无法修复它。你的凭证格式错了，或者密码里有特殊字符没有转义。
- **`403` 是目标站点拒绝。** 它可能意味着 IP 问题，也可能意味着 header 本身可疑。把两者混为一谈，是你把健康代理烧掉的原因。

保留 Scrapy 的默认重试状态码（`500, 502, 503, 504, 522, 524, 408, 429`），不要为了“以防万一”就把 `403` 和 `407` 加进去。如果你确实需要针对特定目标重试某个特定状态码，请把它们分开测试，并记录原因。

## Scrapy 爬取的按流量计费实际要花多少钱

住宅代理按流量计费，而 Scrapy 爬取是按字节计费的。这不总是一个舒服的组合，所以值得做一下计算。

一个 gzip 压缩后的 HTML 页面大约是 50–150 KB。也就是说，大约有 7,000–20,000 个页面包含在 1 GB 内。如果你的爬虫在验证器或选择器上大量浪费流量——完整的浏览器头部、多次重试、渲染出来的 JSON——账单会涨得很快，但同样的原则仍然成立：测量你每抓取一个页面的开销成本，而不是凭感觉。

DataImpulse 的计费方式是预充值余额的按量付费，流量永不过期。当你的爬取量随需求或可用性波动时，这比每个月订阅更合适，不会让半计划额度白白浪费。

## 全部方案，以及各方案适用的场景

DataImpulse 上架了四种代理类型，每种都有入门、基础和大流量层级。所有都是按量付费；基础价格包含国家级定向。

| 代理类型 | 层级 | 流量 | 价格 | 单价 | 购买 |
| --- | --- | --- | --- | --- | --- |
| 住宅 | Intro | 5 GB | $5 | $1.00/GB | [以 $5 试用住宅代理](https://bit.ly/dataimPulse) |
| 住宅 | 批量 | 1 TB | $800 | $0.80/GB | [购买 1 TB 住宅流量](https://bit.ly/dataimPulse) |
| 住宅 | 大批量 | 5 TB+ | 定制（起价 $4,000） | 定制 | [获取住宅批量报价](https://bit.ly/dataimPulse) |
| 移动 | Intro | 2.5 GB | $5 | $2.00/GB | [试用移动代理](https://bit.ly/dataimPulse) |
| 移动 | 批量 | 1 TB | $1,600 | $1.60/GB | [购买 1 TB 移动流量](https://bit.ly/dataimPulse) |
| 移动 | 大批量 | 5 TB+ | 定制（起价 $8,000） | 定制 | [获取移动批量报价](https://bit.ly/dataimPulse) |
| 数据中心 | Intro | 10 GB | $5 | $0.50/GB | [以 $5 试用数据中心代理](https://bit.ly/dataimPulse) |
| 数据中心 | 批量 | 1 TB | $450 | $0.45/GB | [购买 1 TB 数据中心流量](https://bit.ly/dataimPulse) |
| 数据中心 | 大批量 | 5 TB+ | 定制（起价 $2,250） | 定制 | [获取数据中心批量报价](https://bit.ly/dataimPulse) |
| 高级住宅 | Intro | 1 GB | $5 | $5.00/GB | [试用高级住宅代理](https://bit.ly/dataimPulse) |
| 高级住宅 | 大批量 | 5 TB+ | 定制（起价 $20,000） | 定制 | [获取高级住宅批量报价](https://bit.ly/dataimPulse) |

相同的额外项适用于所有方案：流量永不过期，没有订阅，无需改代码即可使用 HTTP/HTTPS 和 SOCKS5，基础价格包含国家级定向。更精细的定向——城市、州、ZIP、ASN——在标准住宅流量上是付费附加项，每 GB 按 2 倍计费。提前做预算，因为 Zscaler 级网格可能会让你的有效费率翻倍。这是选错地方就真正会咬人的细节。

**将代理类型映射到你的爬虫：**

- **数据中心（$0.50/GB）**——速度快、成本低，而且一旦目标站点排查 IP 来源就会被屏蔽。用于你自己的基础设施、开放的 API 以及无防护的站点。
- **住宅（$1/GB）**——受防护目标网站的默认选择：电商 SKU、SERP、市场平台。如果你只买一种代理，请买这种。
- **移动（$2/GB）**——用于反机器人最激烈的场景，以及任何仍然在意运营商 ASN 的移动 Web 界面。
- **高级住宅（$5/GB）**——经过筛选的池子、专属账户经理、不额外收取定向费用。适合替代品开始失败时的按请求成本计算。

DataImpulse 表示池子是在内部构建的，而不是转售的，这通常意味着共享滥用历史较少。该公司公布的 7 天退款政策覆盖新用户，所以入门层级是一个低成本的方式来测试你的实际每请求成本。

👉 [Try DataImpulse risk-free with a $5 intro pack](https://bit.ly/dataimPulse)

## 值得尽早设定三个设置

1. **设置显式的超时。** 在 Crawler 上配置 `DOWNLOAD_TIMEOUT`。没有这个设置，死掉的端点会拖住整个请求槽位，看起来就像你的爬虫卡住了。
2. **限制并发。** 即使通过网关，几千个并行请求看起来也像滥用行为。住宅流量情况更糟，因为每个 IP 都是真实用户。
3. **保持登录信息不进入代码。** 用环境变量或 `.env` 文件配合 `python-dotenv`，然后在项目初始化后立即把它加入 `.gitignore`。如果你曾经用 `print()` 打印过请求 URL 来调试，那么凭证可能已经出现在某个日志里了。

## 常见问题

**我需要在 Scrapy 中使用 SOCKS5 吗？**

只有在你的目标或基础设施需要它的时候才用。HTTP/HTTPS 可以处理大多数爬虫工作，DataImpulse 分别在 823 和 824 端口提供两者的轮换网关。
