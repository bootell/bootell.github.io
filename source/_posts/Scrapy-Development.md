---
title: 'Scrapy 爬虫开发'
date: 2019-08-06 10:00:00
tags:
- 爬虫
---


### Installation
Scrapy 支持 Python2.7 及 3.4+，安装步骤按照[官方文档](<https://docs.scrapy.org/en/latest/intro/install.html>)进行
```bash
# 安装
pip3 install Scrapy
# 创建项目
scrapy startproject crawler
```


安装完毕后，目录结构如下所示
```
crawler/
    scrapy.cfg            # scrapyd-client 项目部署时使用
    crawler/
        __init__.py
        items.py          # 爬取结果结构化
        middlewares.py    # 项目中间件
        pipelines.py      # 项目管道
        settings.py       # 项目设置
        spiders/          # 爬虫所在目录
            __init__.py
```


Scrapy 执行时的流程大致是

![Scrapy 流程](/images/scrapy_flow.png)

1. Engine 从 Spider 获取 `start_urls`
2. Engine 将 `start_urls` 发送到 Scheduler，并请求下一个爬取的 Request
3. Scheduler 将下一个要爬取 Request 返回给 Engine
4. Engine 将收到的 Request 执行所有 Middleware 的 `process_request()` 后，发送到 Downloader
5. Downloader 下载内容后，执行所有 Middleware 的 `process_response()` 后，将结果返回给 Engine
6. Engine 将内容发送给 Spider 做数据处理，之前执行 Middleware 的 `process_spider_input()`
7. Spider 处理后，将结果通过 Middleware 的 `process_spider_output()` 后，返回给 Engine
8. Engine 将处理后的数据发送给 Pipline 进行操作，并将处理过的 Request 发送给 Scheduler，请求下一个 Request
<!--more-->


`settings.py` 配置文件中，一般需要修改的配置如下
```python
# 是否遵循 robots 协议
ROBOTSTXT_OBEY = False
# 并发请求数
CONCURRENT_REQUESTS = 16
# 下载间隔，实际范围在 0.5*DOWNLOAD_DELAY 到 1.5*DOWNLOAD_DELAY 之间
DOWNLOAD_DELAY = 1
# 域名/IP 并发请求限制
CONCURRENT_REQUESTS_PER_DOMAIN = 16
CONCURRENT_REQUESTS_PER_IP = 16
# 启用 Cookies
COOKIES_ENABLED = False
# 下载中间件，后面数字表示优先级
DOWNLOADER_MIDDLEWARES = {}
# 管道
ITEM_PIPELINES = {}
```




### Spider
框架事先定义了几个通用的爬虫
- `scrapy.spiders.Spider` 最基本的爬虫
- `scrapy.spiders.CrawlSpider` 最常用爬虫，能根据规则对全站网站进行爬取
- `scrapy.spiders.XMLFeedSpider`
- `scrapy.spiders.CSVFeedSpider`
- `scrapy.spiders.SitemapSpider`


`CrawlSpider` 继承了 `Spider` 外，提供了额外的属性和方法：

- `rules`：是 `Rule` 对象的列表，定义了爬取网站的规则。它对不同的连接所需要执行的动作进行了定义。
- `parse_start_url()`：`start_url` 的请求返回时，该方法会被调用


`Rule` 对象主要作用为过滤有效链接，指定链接处理方法，并确定是否继续跟进

- `link_extractor` 从爬取的页面中提取指定格式的链接，生成新的 Request 请求，规则有 `allow`，`deny` 等，详细见[官方文档](https://docs.scrapy.org/en/latest/topics/link-extractors.html#topics-link-extractors)
- `callback` 对格式匹配的页面执行对应的处理方法
- `cb_kwargs` 回掉函数的参数
- `follow` 是否对页面中的连接继续跟进，当 `callback` 为 `None` 时默认为 `True`，其他默认为 `False`
- `process_links` 过滤提取的链接
- `process_request` 对指定链接 Request 请求进行处理


完整示例 `crawlwer/spiders/ithome.py` 如下所示
```python
import scrapy
from scrapy.spiders import CrawlSpider, Rule
from scrapy.linkextractors import LinkExtractor
from ..items import CrawlerItem


class Ithome(CrawlSpider):
    # 爬虫名臣
    name = "ithome"
    # 允许爬取的域名
    allowed_domains = [
        'ithome.com',
    ]
    # 初始链接
    start_urls = ['https://www.ithome.com']

    # 爬取的规则
    rules = (
        Rule(LinkExtractor(allow=('[0-9]+\.htm')), callback='parse_article', follow=True),
        Rule(LinkExtractor(allow=('.*\.htm'))),
    )

    def parse_article(self, response):
        self.logger.info('Parsing url: %s', response.url)
        item = CrawlerItem()
        item['url'] = response.url
        item['title'] = response.css('.post_title h1::text').get()
        content = response.css('.post_content p ::text').getall()
        item['content'] = ''.join(content)
        return item
```




### Items
用来定义结构化的结果
完整示例 `crawler/items.py` 如下所示
```python
import scrapy


class CrawlerItem(scrapy.Item):
    url = scrapy.Field()
    title = scrapy.Field()
    content = scrapy.Field()
    comments = scrapy.Field()
```




### Middlewares
middleware 主要用来在下载，处理爬虫等前后，进行相关操作。
本次主要用来处理反爬虫。最基本反爬虫一般是通过浏览器的 UA，客户端的 IP，以及动态加载的 JS 来实现。于是针对以上措施，分别进行处理。


- 请求 UA 随机中间件
  需要安装 `pip3 install fake-useragent`，它从 `useragentstring.com` 和 `w3schools.com` 获取真实的浏览器 useragent，并在本地进行缓存
  最早时从网上找了一些 UA，在本地做了一个随机获取，结果网上的 UA 已被翻爬虫过滤了，不能绕过反爬机制
```python
from fake_useragent import UserAgent


class UserAgentMiddleware(object):
    def __init__(self):
        self.ua = UserAgent()

    def process_request(self, request, spider):
        request.headers.setdefault('User-Agent', self.ua.random)
```


- 随机代理中间件
```python
import redis


class ProxyMiddleware(object):

    def __init__(self, from_url, proxy_pool_key):
        self.redis = redis.from_url(from_url)
        self.proxy_pool_key = proxy_pool_key

    @classmethod
    def from_crawler(cls, crawler):
        return cls(
            from_url=crawler.settings.get('REDIS_URL'),
            proxy_pool_key=crawler.settings.get('PROXY_POOL_KEY'),
        )

    def process_request(self, request, spider):
        proxy = self.redis.srandmember(self.proxy_pool_key)
        if proxy:
            proxy = proxy.decode()
            spider.logger.info('Using proxy: %s', proxy)
            request.meta['proxy'] = proxy
```

> #### 代理配置
>
> 服务器有多个IP，可以使用 [squid](<https://wiki.squid-cache.org/>) 创建 http 代理服务器，通过设置代理不同端口使用不同的 IP 地址。
>
> 安装直接通过 `apt-get install squid` ，安装完成后修改配置文件 `/etc/squid/squid.conf`
>
> ```
# 定义权限规则列表
acl port3128 localport 3128
acl port3129 localport 3129
acl port3130 localport 3130
acl port3131 localport 3131
acl port3132 localport 3132
acl port3133 localport 3133

# 定义访问控制，允许接入的地址
http_access allow all

# 定义监听的端口
http_port 3128
http_port 3129
http_port 3130
http_port 3131
http_port 3132
http_port 3133

# 定义转发地址
tcp_outgoing_address 172.16.0.106 port3128
tcp_outgoing_address 172.16.0.107 port3129
tcp_outgoing_address 172.16.0.108 port3130
tcp_outgoing_address 172.16.0.109 port3131
tcp_outgoing_address 172.16.0.110 port3132
tcp_outgoing_address 172.16.0.248 port3133
```


- CloudFlare 反爬虫，起主要反爬方法是通过 JS 生成本地 Cookie。
  可以通过 [scrapy_cloudflare_middleware](https://github.com/clemfromspace/scrapy-cloudflare-middleware) 进行处理，直接安装 `pip3 install scrapy_cloudflare_middleware`


启动的 Middlewares 需要写入 `settings.py` 配置文件
``` python
# CloudFlare 反爬需要开启 cookie
COOKIES_ENABLED = True

DOWNLOADER_MIDDLEWARES = {
    # 禁用 Scrapy 自带的 UA Middleware
    'scrapy.contrib.downloadermiddleware.useragent.UserAgentMiddleware': None, 
    'crawler.middlewares.UserAgentMiddleware': 400,
    'crawler.middlewares.ProxyMiddleware': 543,
    'scrapy_cloudflare_middleware.middlewares.CloudFlareMiddleware': 560
}
```




### Piplines

```python
import pymongo


class MongoPipeline(object):
    collection_name = 'scrapy_items'

    def __init__(self, mongo_uri, mongo_db):
        self.mongo_uri = mongo_uri
        self.mongo_db = mongo_db

    @classmethod
    def from_crawler(cls, crawler):
        return cls(
            mongo_uri=crawler.settings.get('MONGO_URI'),
            mongo_db=crawler.settings.get('MONGO_DATABASE', 'items')
        )

    def open_spider(self, spider):
        self.client = pymongo.MongoClient(self.mongo_uri)
        self.db = self.client[self.mongo_db]

    def close_spider(self, spider):
        self.client.close()

    def process_item(self, item, spider):
        self.db[self.collection_name].insert_one(dict(item))
        return item
```

在 `settings.py` 配置中增加

```python
ITEM_PIPELINES = {
    'crawler.pipelines.MongoPipeline': 300,
}

MONGO_URI='mongodb://127.0.0.1:27017'
MONGO_DATABASE = 'items'
```




### Distributed crawling
Scrapy Scheduler 和 Duplication Filter 本身使用了本地文件来来存储，不能进行水平的扩展。可以使用 [Scrapy-Redis](<https://github.com/rmax/scrapy-redis>) 来存放这些数据，使爬虫能够方便扩展，可以分布式部署。

安装通过 `pip3 install Scrapy-Redis`，安装后修改 settings 配置
```python
SCHEDULER = "scrapy_redis.scheduler.Scheduler"
DUPEFILTER_CLASS = "scrapy_redis.dupefilter.RFPDupeFilter"
SCHEDULER_PERSIST = True

REDIS_URL='redis://:password@127.0.0.1:6379/0'
```

修改 Spider 继承 `scrapy_redis.spiders.RedisCrawlSpider`
``` python
from scrapy_redis.spiders import RedisCrawlSpider


class Ithome(RedisCrawlSpider):
    # start_urls = ['https://www.ithome.com']
```

由于任务是从 redis 中读取，所以 `start_urls` 需要直接存入 redis `redis-cli lpush ithome:start_urls https://www.ithome.com`



### Deploy
只运行单个爬虫时，直接使用 `scrapy crawl spider-name` 命令来运行，按 `Ctrl+C` 来停止。

部署到服务器执行时，需要执行多个爬虫，可以通过 `scrapyd` 服务运行、监控。
首先安装 `pip3 install scrapyd `，然后在项目目录创建配置文件 `scrapyd.conf` 如下
```ini
[scrapyd]
eggs_dir    = eggs
logs_dir    = logs
items_dir   =
# 日志最大保存数量
jobs_to_keep = 5
dbs_dir     = dbs
# 同时执行的最大爬虫数量，设为0时，最大执行数量为 max_proc_per_cpu * cpu 核心数
max_proc    = 0
# 每个 CPU 最大同时执行爬虫数量
max_proc_per_cpu = 4
# 爬虫历史最大保存数量
finished_to_keep = 100
poll_interval = 5.0
bind_address = 127.0.0.1
http_port   = 6800
debug       = off
runner      = scrapyd.runner
application = scrapyd.app.application
launcher    = scrapyd.launcher.Launcher
webroot     = scrapyd.website.Root
[services]
schedule.json     = scrapyd.webservice.Schedule
cancel.json       = scrapyd.webservice.Cancel
addversion.json   = scrapyd.webservice.AddVersion
listprojects.json = scrapyd.webservice.ListProjects
listversions.json = scrapyd.webservice.ListVersions
listspiders.json  = scrapyd.webservice.ListSpiders
delproject.json   = scrapyd.webservice.DeleteProject
delversion.json   = scrapyd.webservice.DeleteVersion
listjobs.json     = scrapyd.webservice.ListJobs
daemonstatus.json = scrapyd.webservice.DaemonStatus
```

scrapyd 提供了 API 接口用来控制和监控爬虫
```bash
# 列出可运行的所有爬虫
curl http://localhost:6800/listspiders.json\?project\=default
# 执行爬虫
curl http://localhost:6800/schedule.json -d project=default -d spider=spider-name 
# 列出所有任务
curl http://localhost:6800/listjobs.json?project=default
# 取消爬虫
curl http://localhost:6800/cancel.json -d project=default -d job={job-id}
```

另外 scrapyd 还提供了一个 web 界面方便查看，由于 scrapyd 本身没有提供授权机制，可以使用 nginx 反向代理并设置 Basic Auth。创建 nginx 配置文件 `/etc/nginx/sites-enabled/scrapyd` 
```nginx
server {
    listen 6801;
    location / {
        proxy_pass http://127.0.0.1:6800/;
        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/conf.d/.htpasswd;
    }
}
```

生成密码文件
```bash
htpasswd -c /etc/nginx/conf.d/.htpasswd username password
```




### Speed optimization
按照默认配置部署到服务器之后，发现服务器负载非常低，爬取速度也很慢。可以简单的修改配置加快爬虫速度
```python
CONCURRENT_REQUESTS = 100

DOWNLOAD_DELAY = 0

CONCURRENT_REQUESTS_PER_DOMAIN = 100
CONCURRENT_REQUESTS_PER_IP = 100

REACTOR_THREADPOOL_MAXSIZE = 20
```

