---
title: "自动化记账--利用 n8n 将银行账单自动记录到 MoneyWiz"
date: 2023-01-30T13:14:00+00:00
# weight: 1
# aliases: ["/first"]
tags: ["记账", "自动化", "n8n", "moneywiz"]
author: "Rory"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: ""
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
hideSummary: true
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "/cmb-billing-n8n-workflow.png" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/gongzili456/btree.fun/blob/master/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

由于中国大陆的银行不对个人提供 API 能力，所以无法实现自动化的记账流程，本文介绍一个变通的实现方式。本文主要以招商银行为例。其他银行可以自行尝试。


招商银行的储蓄卡提供了动账的邮件实时提醒能力（也有短信提醒），信用卡也有每日账单提醒。我将这个账单配置成发送到我的 Gmail 邮箱。

通过对 Gmail 的监听，我们可以获取到账单的邮件，然后通过邮件内容解析出账单的信息，会拿到一个结构化的数据。


[MoneyWiz](https://www.wiz.money/) 是一款优秀的财务管理软件。他支持 URL Scheme，可以通过 URL Scheme 来添加账单。具体可参考文档 [Automate transaction management with URL Schemas](https://help.wiz.money/en/articles/4525440-automate-transaction-management-with-url-schemas)。

## 必要条件
- 安装 n8n，我这里使用是 self-host 方式安装在了 DigitalOcean。你可以选择合适方式安装。参考 [n8n 官方文档](https://docs.n8n.io/choose-n8n/)
- 一个域名，n8n 默认使用 HTTPS，一些需要对接的服务如 Gmail 的鉴权也需要 HTTPS。
- (可选) 如何需要使用 Slack 或者 Google 服务，那么需要注册对应的账号。

## 分析账单内容
招商银行储蓄卡的账单有很多类型，每一种都是固定文本的内容，所以可以通过正则表达式来解析出账单的信息。

### 消费类
您账户1234于01月19日11:26在【支付宝-星巴克】发生快捷支付扣款，人民币123.50，余额10000.00

### 退款类
您账户1234于01月12日10:43在【程支付-携程旅行网】发生快捷支付退款，人民币5.00，余额10000.00

### 转账（转出）
您账户1234于01月19日10:27实时转至他行人民币1.00，余额10000.00，收款人张三

### 转账（转入）
您账户1234于01月02日他行实时转入人民币1234.56，余额18063.77，付方张三

### 信用卡还款
您账户1234于12月25日18:20信用卡还款交易人民币11233.45，余额10000.00

其他还有诸如贷款收款、贷款还款、银证转账等就不一一列举了，可以自行分析。

## 创建 n8n 工作流
![](/cmb-billing-n8n-workflow.png)

### 创建 Gmail Trigger
Gmail Trigger 用于监听 Gmail 邮件，当有新邮件时触发。这里可以配置监听过滤器，只关心来自 95555@message.cmbchina.com 的邮件。其输出结果如下：
```json
[
    {
        "id":"186052ab09dfc60e",
        "threadId":"186052ab09dfc60e",
        "snippet":"您账户1234于01月31日08:12在【支付宝-星巴克】发生快捷支付扣款，人民币30.58，余额1234.56",
        "payload":{
            "mimeType":"multipart/mixed"
        },
        "sizeEstimate":3143,
        "historyId":"8327386",
        "internalDate":"1675123927000",
        "labels":[
            {
                "id":"INBOX",
                "name":"INBOX"
            },
            {
                "id":"IMPORTANT",
                "name":"IMPORTANT"
            },
            {
                "id":"CATEGORY_UPDATES",
                "name":"CATEGORY_UPDATES"
            }
        ],
        "From":"95555@message.cmbchina.com",
        "To":"example@gmail.com",
        "Subject":"一卡通账户变动通知"
    }
]
```
### 分析邮件正文
n8n 提供了 Code 节点来让我们用 JavaScript 代码实现我们想要的处理，你可能注意到了上面 Trigger 的输出是数组，而我们只需要关心数组中的元素即可，因为 Code 节点提供了 `Run once for each item` 的模式来执行代码。

### 组成 MoneyWiz 数据结构
由于账单的类型有多种，所以我们的分析邮件的代码只需要将正则匹配到的数据直接返回即可，然后使用 n8n 的 Set 节点来统一组成 MoneyWiz 的数据结构。
每一个字段的映射都可以用 expression 表达式来编写。如图所示，用鼠标点击下就能轻松实现。
![](/n8n-set-node.png)

### 拼接 MoneyWiz URL Schema
还是利用 n8n 的 Code 节点来实现，具体的参照官方文档 [Automate transaction management with URL Schemas](https://help.wiz.money/en/articles/4525440-automate-transaction-management-with-url-schemas)。
> 注意 URL 需要进行 URL 编码。

### Slack 通知
最后需要将 URL Schema 通过 Slack 通知到手机上，这样就可以直接点击打开 MoneyWiz App 了。
![](/moneywiz-slack.JPG)

### 导入到 MoneyWiz
现在，只需要点击 Slack 消息中按钮就可以打开 MoneyWiz App 的创建交易页面了，并填充好了数据。至此，我们的自动化流程就完成了。
> 需要注意的是，由于账单内容无法识别到账单分类，所以还需要手动选择分类。好在 MoneyWiz 有分类记忆功能，对于常用的交易，MoneyWiz 会自动填充分类。

![](/moneywiz-trasiction-create.JPG)

## 总结
通过这个小项目，我们演示了如何使用 n8n 来实现自动化记账的流程，以及如何使用 MoneyWiz 的 URL Schema 来实现自动化交易录入。
其他银行的账单也可以通过类似的方式来实现自动化记账，只需要分析邮件正文，然后组成 MoneyWiz 的数据结构即可。
如果你使用的银行没有邮件提醒，那么也可以通过 iOS 的捷径监听短信来实现，大致的思路是一致的。

如果你有更好的想法，欢迎在 Twitter 给我留言。

<blockquote class="twitter-tweet"><p lang="zh" dir="ltr">利用自建的n8n实现通过监控招商银行的动帐邮件，将消费记录结构化数据后存google sheet中，符合MoneyWiz的csv文件格式，方便后期导入。 <a href="https://t.co/bO3vN61vfx">pic.twitter.com/bO3vN61vfx</a></p>&mdash; Rory (@hola_rory) <a href="https://twitter.com/hola_rory/status/1616422426694553607?ref_src=twsrc%5Etfw">January 20, 2023</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

> 我把这个工作流的 JSON 开源在了 GitHub 上，可以参考 [gist](https://gist.github.com/gongzili456/5c934c988c3c8ba09706b8a7329b637c)。