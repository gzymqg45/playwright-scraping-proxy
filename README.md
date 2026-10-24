# playwright scraping：真实浏览器能帮你拿到什么、什么时候会被拦，以及如何通过代理端口接入 ScraperAPI 扩展规模

Playwright 擅长解决一件事：让依赖 JavaScript 的网站能够被抓取。它会启动一个真正的 Chromium（或 Firefox、WebKit）引擎，等待元素渲染完成，然后为你提供完整的 DOM。本指南将带你完成实际的 playwright scraping 工作流，包括安装、等待、会话、代理，以及那些实际上会被封锁的教程所跳过的部分。此外，当你的 Playwright 爬虫数量增加时，如何通过代理端口将其接入 ScraperAPI（使用其现有的定价方案）。

> **快速建议：** 大多数使用 Playwright 并连接到 ScraperAPI 的开发人员，从每月 49 美元的 Hobby 计划开始（包含 100,000 个 API 额度、20 个并发线程）。你可以在不绑定信用卡的情况下免费尝试 5,000 个额度，为期 7 天。 👉 [Check ScraperAPI's current plans and trial](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Playwright 爬虫能提供什么原生 HTTP 请求无法提供的服务

Playwright 会驱动一个真实的浏览器。当页面的内容只有通过 JavaScript 才能显示时，普通的 HTTP 请求返回的 HTML 是空的，而 Playwright 会先运行页面的脚本，然后交给你的代码一个已完全渲染的 DOM。

使用两个命令即可开始使用：

bash
pip install playwright
playwright install chromium


（Node.js 用户运行 `npm init playwright@latest`。每个浏览器引擎都会下载约 100–300 MB；如果你只传递 `playwright install chromium` 参数，则只会下载 Chromium。）

一旦开始使用，有三点使它比旧的自动化栈更省事：

- **Locators 自动等待。** 每次定位器操作都会重新检查元素，并等待其附加、可见、稳定且启用，超时时间为 30 秒。你会编写的针对 Selenium 的显式 sleep/wait 代码大部分都消失了，这直接减少了不稳定的爬虫。
- **每个上下文隔离。** `browser.new_context()` 可以在单个浏览器进程内为你提供独立的 Cookie、存储、缓存、区域设置、时区和地理位置。并行爬取的标准模式是一个浏览器和许多廉价的上下文。
- **路由拦截。** 你可以阻止图像和字体以加快爬取速度，或者——通常更好——使用 `page.expect_response()` 捕获网站自身的 JSON API 响应，并完全跳过 HTML 解析。

对于无限滚动页面，循环滚动鼠标滚轮，等待片刻，然后对比产品卡片的数量；当数量停止增长时，你就完成了。对于调试，请使用 `headless=False, slow_mo=300` 运行，以便观察你的爬虫在工作。

当 Playwright 在新的项目中胜过 Selenium 和 Puppeteer 时：Selenium 在你确实需要真正的 Safari 或奇特的浏览器网格时仍然胜出，而 Puppeteer 适用于专注于 Node + Chrome 的情况。对于其他情况，Playwright 的等待模型和语言支持使其成为默认选择。

## 教程忽略的部分：无头模式仍然会被检测到

Playwright 会自动化一个真实的浏览器，但它不会隐藏它。网站可以从默认 User-Agent 中的 `HeadlessChrome` 字符串以及其他更隐蔽的环境信号中检测到自动化。这对你意味着：

- 简单的目标：默认的无头 Chromium 构建（`chromium-headless-shell`）工作正常。传入 `channel="chromium"` 可切换到行为更像有头浏览器的真实 Chrome 新无头模式。
- 中等目标：在每个上下文级别设置自定义 User-Agent、区域设置和地理位置。
- 强硬目标：在 Cloudflare bot management、Datadome 或 PerimeterX 之后的网站，无论你的选择器有多好，都会阻止普通的 Playwright。这时你需要轮换代理或抓取 API，这也是 ScraperAPI 发挥作用的地方。

## 通过代理端口将 Playwright 连接到 ScraperAPI

ScraperAPI 没有专门的 Playwright SDK，你也不需要。其代理端口是整个 API 的轻量级前端：相同的轮换、CAPTCHA 处理和重试逻辑，只是通过普通的 HTTP 代理连接公开。任何接受代理的库（包括 Playwright）都可以使用它。

代理端点是 `http://proxy-server.scraperapi.com:8001`。用户名始终为 `scraperapi`，密码是你的 API key。额外参数通过在用户名中以句点分隔来传递，因此启用 JavaScript 渲染意味着用户名变为 `scraperapi.render=true`。

在 Playwright 中，看起来像这样：

python
context = browser.new_context(
    proxy={
        "server": "http://proxy-server.scraperapi.com:8001",
        "username": "scraperapi.render=true",
        "password": "YOUR_API_KEY",
    },
    ignore_https_errors=True,
)


以下是官方文档中容易被忽略的陷阱，也是大多数第三方教程从未提及的：**必须禁用 SSL 验证，否则请求将无法通过 API 正确路由。** 在 Playwright 中，这就是上下文选项中的 `ignore_https_errors=True`。如果你确实想保留证书验证，ScraperAPI 提供了可以手动信任的 Proxy CA 证书——在 Windows 上通过 Microsoft Management Console 导入“受信任的根证书颁发机构”，在 macOS 上导入 Keychain Access 并设置为 Always Trust，或者在 Linux 上安装 `proxyca.pem` 并更新你的 CA 存储。

同样的用户名技巧适用于多个参数：`scraperapi.render=true.premium=true` 在同一个字符串中启用了渲染和高级住宅池。无需更改其他任何内容，你的 Playwright 代码结构保持不变。

准备好在真实目标上试用吗？ 👉 [Grab a free API key and test it against your target site](https://www.scraperapi.com/?fp_ref=coupons)

## 当你不需要本地浏览器时：渲染指令集

这里是第二个人们忽略的选项。如果你的唯一原因是需要填写表单、点击按钮或滚动，你可以在不下载浏览器、不管理上下文的情况下，完全在 ScraperAPI 内部完成，使用其 Render Instruction Set。你将指令的 JSON 数组作为标头与 `render=true` 一起发送，ScraperAPI 的无头浏览器会在渲染期间执行它们。

七种可用指令：`click`、`input`、`loop`、`scroll`、`wait`、`wait_for_event` 和 `wait_for_selector`。一个搜索流程看起来像输入到 `#searchInput`，点击提交按钮，然后等待 `networkidle`。无限滚动页面使用带有 `for` 计数的 `loop`，包含滚动到底部和等待的步骤，注意有一个限制：循环不能嵌套。

事件等待包括 `domcontentloaded`、`load`、`navigation`、`networkidle` 和 `stabilize`，其中 stabilize 的默认值为 5 秒，最大值为 30 秒。

诚实地说，两者的权衡：Playwright 为你提供了完整的编程逻辑、任意交互以及调试功能。Render Instruction Set 的表达能力较弱，但它完全在你的服务器之外运行，没有浏览器下载，没有上下文管理，也少了一层检测。对于“搜索、滚动、提取”类的工作，它的代码量要少得多。

## ScraperAPI 定价：完整计划表

所有付费计划都包含相同的功能集，包括 JS 渲染、高级住宅和移动 IP、CAPTCHA 绕过、结构化数据 API 和 DataPipeline 访问。不同之处在于额度、并发和地理覆盖范围。

| 计划 | 价格/月 | 年付价格/月 | API 额度 | 并发线程 | 订阅 |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | — | 1,000 | 5 | [开始免费](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | $49 | $44.10 | 100,000 | 20 | [获取 Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | $149 | $134.10 | 1,000,000 | 50 | [获取 Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | $299 | $269.10 | 3,000,000 | 100 | [获取 Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | $475 | $427.50 | 5,000,000 | 200 | [获取 Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | $975 | $877.50 | 10,500,000 | 300 | [获取 Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | $1,975 | $1,777.50 | 21,500,000 | 500 | [获取 Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | 定制 | 定制 | 22,000,000+ | 500+ | [联系销售](https://www.scraperapi.com/contact-sales/) |

年度计费比月度价格节省 10%。请注意，Hobby 和 Startup 地理定位仅覆盖美国和欧盟，其分析面板仅保留 30 天的数据；国家级地理定位和无限保留期从 Business 计划开始。

## 每页的真实成本，而不是表头数字

这是任何 Playwright 教程中都不会告诉你的数字，也是 ScraperAPI 文档中最坦诚的部分：一个额度代表一个标准页面，但大多数真正的目标并不是标准页面。

根据官方成本表：

- 标准页面：1 个额度
- Amazon：5 个额度
- Google 和 Bing（包括子域名）：25 个额度
- LinkedIn：30 个额度
- 受保护站点（Cloudflare、Datadome、PerimeterX）：当绕过激活时，每请求增加 10 个额度
- `render=true` JavaScript 渲染：大约每个请求 10 个额度

将它们叠加起来，情况变化很快。一个使用 JS 渲染加上高级池针对受保护目标的任务，其每次成功抓取的成本可能比表头价格高出 20–30 倍；一个包含启用渲染的 Amazon 搜索结果的百万额度计划可能只能覆盖 40,000–100,000 个实际页面。在决定使用哪个计划之前，先确定你实际的单页成本。

两个工具使这变得易于管理，且都包含在每个计划中：

1. 仪表板中的 **Domain Cost Estimator** 会在你抓取任何 URL 之前告诉你其真实的额度成本。
2. **`max_cost`** 参数设置了你愿意让任何单个请求消耗的上限，因此一个配置错误的请求不会耗尽你整个月的额度。

## 选择哪个计划：场景下的规则

- **副业项目，每月最多约 10 万个标准页面：** Hobby 计划，每月 49 美元。每月 20 个线程足以处理基于文本的价格监控或小型 SERP 跟踪器。但请注意其仅限美国/欧盟的地理定位。
- **严肃的产品或 SERP 工具，几十万到约 100 万页：** Startup，每月 149 美元。50 个线程，一百万个额度，这是大约 20,000–200,000 页/月的常规使用区间。
- **需要非美国/欧盟目标或无限数据保留的国家级地理定位：** 从 Business（每月 299 美元）开始。
- **生产规模的管道：** Scaling（每月 475 美元，使用最广泛）及以上级别。Scaling、Professional、Advanced 和 Enterprise 增加了按使用量计费超额，因此你的爬虫不会在额度用尽时停止，而是以固定的每额度费率持续运行，最高可达到你设置的月度上限。Hobby 到 Professional 级别则提供了可选的自动升级功能，当你的额度耗尽时会自动将你移至下一个计划。

超过并发限制的请求将收到 429 错误，不会被排队或计费，只需减少你的并发限制并重试即可。

## 谁应该完全跳过这条路线

诚实胜过推介。两个使用场景更适合其他工具：

- **大批量抓取像 LinkedIn 或 Instagram 这样的强硬目标。** 超高级额度倍数（高级池每请求 10–30 个额度）会使这比按带宽计费的专用住宅代理计划更昂贵。
- **需要超过约一小时会话持久性的任何内容**，例如广告验证或长期运行的账户工作流。`session_number` 的粘性会话旨在用于单次抓取运行；独立的住宅或移动代理提供商更适合长期固定 IP。

同时，如果你的目标位于重度机器人保护之后，ScraperAPI 的默认池可以清除大部分请求，而无需使用昂贵的高级层级，因此普通 Playwright 目标的定价非常接近表头价格。

## 常见问题

**Playwright 可以直接与 ScraperAPI 一起使用吗？**
可以，通过 `proxy-server.scraperapi.com:8001` 上的代理端口，用户名为 `scraperapi`（加上像 `.render=true` 这样的点分隔参数），密码为你的 API key。你必须在 Playwright 上下文中禁用 SSL 验证，除非你安装了 ScraperAPI 的 CA 证书。

**ScraperAPI 的代理端口是否有免费计划？**
有，注册即获得 1,000 个免费额度，最多 5 个并发连接，无需信用卡。如果你需要更多测试额度，7 天试用提供 5,000 个，同样无需绑定信用卡。

**当我的额度用尽时会发生什么？**
默认情况下，抓取会暂停，直到额度更新。根据计划，你可以启用自动升级（Hobby 到 Professional）或切换到按使用量计费超额（Scaling 及以上，带有自我设置的月度上限）。年度计划可以提前续费以获取新的额度批次。

**我将 Playwright 连接到代理端口后，还需要 `render=true` 吗？**
视情况而定。你的本地 Playwright 浏览器已经执行了 JavaScript，所以对于页面本身，你不需要。当 ScraperAPI 处理像 Google 或 Amazon 这样的目标时，你希望它在服务器端进行渲染和绕过，在这种情况下，用户名中的 `.render=true` 参数会启用它。

## 最后检查清单

- 如果你的目标在没有激进机器人保护的情况下渲染 JavaScript，使用带有 ScraperAPI 代理端口的 Playwright，Hobby 计划即为合适的起点，别忘了 `ignore_https_errors=True`。
- 如果目标受到 Cloudflare 或类似服务的保护，请先针对该特定 URL 运行仪表板中的 Domain Cost Estimator，然后根据你真实的每页成本而非额度数量来估算预算，并将 `max_cost` 设置为安全网。
- 如果你的工作流只是表单、点击和滚动，请在编写任何浏览器代码之前尝试 Render Instruction Set 方法，这是最少的维护。

 👉 [See all ScraperAPI plans and start the 7-day free trial](https://www.scraperapi.com/pricing?fp_ref=coupons)
