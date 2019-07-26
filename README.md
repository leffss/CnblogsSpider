### 目标任务
爬取 `https://www.cnblogs.com` 首页文章，爬取的内容包括：标题 、推荐数 、链接、内容预览、作者、作者blogs链接、评论数、查看数。


### 安装爬虫
```
pip install scrapy
```
- `python` 版本 `3.7`， `scrapy` 版本 `1.6.0`


### 创建爬虫
```
#  创建工程
scrapy startproject CnblogsSpider

# 创建爬虫
cd CnblogsSpider
scrapy genspider -t crawl cnblogs cnblogs.com
```
- 爬虫名称 `cnblogs` , 作用域 `cnblogs.com`，爬虫类型 `crawl`


### 编写 `items.py`
``` python
# -*- coding: utf-8 -*-

# Define here the models for your scraped items
#
# See documentation in:
# https://doc.scrapy.org/en/latest/topics/items.html

import scrapy


class CnblogsspiderItem(scrapy.Item):
    # define the fields for your item here like:
    # name = scrapy.Field()
    
    # 文章标题
    title = scrapy.Field()
    
    # 推荐数
    diggnum = scrapy.Field()
    
    # 文章链接
    link = scrapy.Field()

    # 文章内容预览
    post_item_summary = scrapy.Field()
    
    # 作者
    author = scrapy.Field()
    
    # 作者blogs链接
    author_link = scrapy.Field()
    
    # 文章评论数
    article_comment = scrapy.Field()
    
    # 文章查看数
    article_view = scrapy.Field()
    
```
- 定义需求爬取的 `item` 项


### 编写 `spiders/cnblogs.py`
``` python
# -*- coding: utf-8 -*-
import scrapy
# 导入CrawlSpider类和Rule
from scrapy.spiders import CrawlSpider, Rule
# 导入链接规则匹配类，用来提取符合规则的连接
from scrapy.linkextractors import LinkExtractor
from CnblogsSpider.items import CnblogsspiderItem


class CnblogsSpider(CrawlSpider):
    # 爬虫名称
    name = 'cnblogs'
    allowed_domains = ['cnblogs.com']
    start_urls = ['https://www.cnblogs.com/sitehome/p/1']
    
    # Response里链接的提取规则，返回的符合匹配规则的链接匹配对象的列表
    pagelink = LinkExtractor(allow=("/sitehome/p/\d+"))

    rules = [
        # 获取这个列表里的链接，依次发送请求，并且继续跟进，调用指定回调函数处理
        Rule(pagelink, callback="parse_item", follow=True)
    ]
    
    # CrawlSpider的rules属性是直接从response对象的文本中提取url，然后自动创建新的请求。
    # 与Spider不同的是，CrawlSpider已经重写了parse函数
    # scrapy crawl spidername开始运行，程序自动使用start_urls构造Request并发送请求，
    # 然后调用parse函数对其进行解析，在这个解析过程中使用rules中的规则从html（或xml）文本中提取匹配的链接，
    # 通过这个链接再次生成Request，如此不断循环，直到返回的文本中再也没有匹配的链接，或调度器中的Request对象用尽，程序才停止。
    # 如果起始的url解析方式有所不同，那么可以重写CrawlSpider中的另一个函数parse_start_url(self, response)用来解析第一个url返回的Response，但这不是必须的。
    
    def parse_item(self, response):
        for each in response.xpath("//div[@class='post_item']"):
            item = CnblogsspiderItem()
            item['diggnum'] = each.xpath("./div[1]/div[1]/span[1]/text()").extract()[0].strip()
            item['title'] = each.xpath("./div[2]/h3/a/text()").extract()[0].strip()
            item['link'] = each.xpath("./div[2]/h3/a/@href").extract()[0].strip()
            
            item['post_item_summary'] = ''.join(each.xpath("./div[2]/p[@class='post_item_summary']/text()").extract()).strip()
            item['author'] = each.xpath("./div[2]/div/a/text()").extract()[0].strip()
            item['author_link'] = each.xpath("./div[2]/div/a/@href").extract()[0].strip()
            
            item['article_comment'] = each.xpath("./div[2]/div/span[1]/a/text()").extract()[0].strip()
            item['article_view'] = each.xpath("./div[2]/div/span[2]/a/text()").extract()[0].strip()

            yield item

```
- 爬虫的主逻辑


### 编写 `pipelines.py`
``` python
# -*- coding: utf-8 -*-

# Define your item pipelines here
#
# Don't forget to add your pipeline to the ITEM_PIPELINES setting
# See: https://doc.scrapy.org/en/latest/topics/item-pipeline.html
import json


class CnblogsspiderPipeline(object):
    def __init__(self):
        self.filename = open("cnblogs.json", "w", encoding='utf-8')
    
    def process_item(self, item, spider):
        try:
            text = json.dumps(dict(item), ensure_ascii=False) + "\n"
            self.filename.write(text)
        except BaseException as e:
            print(e)
        return item
    
    def close_spider(self, spider):
        self.filename.close()
```
- 处理每个页面爬取得到的 `item` 项


### 编写 `settings.py`
```
# -*- coding: utf-8 -*-

# Scrapy settings for CnblogsSpider project
#
# For simplicity, this file contains only settings considered important or
# commonly used. You can find more settings consulting the documentation:
#
#     https://doc.scrapy.org/en/latest/topics/settings.html
#     https://doc.scrapy.org/en/latest/topics/downloader-middleware.html
#     https://doc.scrapy.org/en/latest/topics/spider-middleware.html

BOT_NAME = 'CnblogsSpider'

SPIDER_MODULES = ['CnblogsSpider.spiders']
NEWSPIDER_MODULE = 'CnblogsSpider.spiders'


# Crawl responsibly by identifying yourself (and your website) on the user-agent
#USER_AGENT = 'CnblogsSpider (+http://www.yourdomain.com)'
# 自定义user_agent
USER_AGENT = 'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) ' \
             'Chrome/73.0.3683.86 Safari/537.36'

# Obey robots.txt rules
# 如果启用,Scrapy将会采用 robots.txt策略
ROBOTSTXT_OBEY = False

# Configure maximum concurrent requests performed by Scrapy (default: 16)
# 设置最大请求数
CONCURRENT_REQUESTS = 32

# Configure a delay for requests for the same website (default: 0)
# See https://doc.scrapy.org/en/latest/topics/settings.html#download-delay
# See also autothrottle settings and docs
#DOWNLOAD_DELAY = 3
# The download delay setting will honor only one of:
#CONCURRENT_REQUESTS_PER_DOMAIN = 16
#CONCURRENT_REQUESTS_PER_IP = 16

# Disable cookies (enabled by default)
# 禁用Cookie（默认情况下启用）
#COOKIES_ENABLED = False

# Disable Telnet Console (enabled by default)
# 禁用Telnet控制台（默认启用）
TELNETCONSOLE_ENABLED = False

# Override the default request headers:
#DEFAULT_REQUEST_HEADERS = {
#   'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
#   'Accept-Language': 'en',
#}
# 设置请求头
DEFAULT_REQUEST_HEADERS = {
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8',
    'Accept-Encoding': 'gzip,deflate,br',
    'accept-language': 'zh-CN,zh;q=0.9',
    'cache-control': 'no-cache',
    'pragma': 'no-cache',
    'upgrade-insecure-requests': '1',
    'host': 'www.cnblogs.com'
}

# 启用或禁用蜘蛛中间件
# Enable or disable spider middlewares
# See https://doc.scrapy.org/en/latest/topics/spider-middleware.html
#SPIDER_MIDDLEWARES = {
#    'CnblogsSpider.middlewares.CnblogsspiderSpiderMiddleware': 543,
#}

# 启用或禁用下载器中间件
# Enable or disable downloader middlewares
# See https://doc.scrapy.org/en/latest/topics/downloader-middleware.html
#DOWNLOADER_MIDDLEWARES = {
#    'CnblogsSpider.middlewares.CnblogsspiderDownloaderMiddleware': 543,
#}

# 启用或禁用扩展程序
# Enable or disable extensions
# See https://doc.scrapy.org/en/latest/topics/extensions.html
#EXTENSIONS = {
#    'scrapy.extensions.telnet.TelnetConsole': None,
#}

# 配置项目管道
# Configure item pipelines
# See https://doc.scrapy.org/en/latest/topics/item-pipeline.html
ITEM_PIPELINES = {
    'CnblogsSpider.pipelines.CnblogsspiderPipeline': 300,
}

# 启用和配置AutoThrottle扩展（默认情况下禁用）
# Enable and configure the AutoThrottle extension (disabled by default)
# See https://doc.scrapy.org/en/latest/topics/autothrottle.html
#AUTOTHROTTLE_ENABLED = True

# 开始下载时限速并延迟时间
# The initial download delay
#AUTOTHROTTLE_START_DELAY = 5

# 高并发请求时最大延迟时间
# The maximum download delay to be set in case of high latencies
#AUTOTHROTTLE_MAX_DELAY = 60

# The average number of requests Scrapy should be sending in parallel to
# each remote server
#AUTOTHROTTLE_TARGET_CONCURRENCY = 1.0

# Enable showing throttling stats for every response received:
#AUTOTHROTTLE_DEBUG = False

# 启用和配置HTTP缓存（默认情况下禁用）, 如果开启会优先读取本地缓存，从而加快爬取速度，视情况而定
# Enable and configure HTTP caching (disabled by default)
# See https://doc.scrapy.org/en/latest/topics/downloader-middleware.html#httpcache-middleware-settings
#HTTPCACHE_ENABLED = True
#HTTPCACHE_EXPIRATION_SECS = 0
#HTTPCACHE_DIR = 'httpcache'
#HTTPCACHE_IGNORE_HTTP_CODES = []
#HTTPCACHE_STORAGE = 'scrapy.extensions.httpcache.FilesystemCacheStorage'

# 设置日志等级
# LOG_LEVEL = 'DEGUG'

```


### 运行爬虫
```
cd CnblogsSpider
scrapy crawl cnblogs
```
- `CnblogsSpider` 为项目文件夹， `cnblogs` 为爬虫名

结果文件 `cnblogs.json` 在项目文件夹根目录，内容如下：
``` json
{"diggnum": "2", "title": "在嵌入式设备中使用 JavaScript 的前景", "link": "https://www.cnblogs.com/conmajia/p/javascript-in-embedded-devices.html", "post_item_summary": "这几年嵌入式/可穿戴设备处理器从 51、AVR 单片机瞬间起飞到 ARM 4 核、8 核，主频 3GHz 起步。硬件能力的提升让硬件集成 JavaScript 引擎成为现实。我大概了解了一下这方面的进展，觉得前景非常广阔。 ...", "author": "Conmajia", "author_link": "https://www.cnblogs.com/conmajia/", "article_comment": "评论(0)", "article_view": "阅读(283)"}
{"diggnum": "1", "title": "Python学习笔记", "link": "https://www.cnblogs.com/xjtu-blacksmith/p/10347247.html", "post_item_summary": "[TOC] 学了N年 的我，现在得为着学业需要而学一下 ，这简明轻快的语言风格真让我有些不习惯。最主要的是，各种各样的命令我总是忘记。以此笔记摘录一下主要内容，以免遗忘。 入门的读本是：《 \"Python语言及其应用\" 》，Bill Lubanovic著，丁嘉瑞、梁杰、禹常隆译。 笔记中若有提到“其 ...", "author": "黑山雁", "author_link": "https://www.cnblogs.com/xjtu-blacksmith/", "article_comment": "评论(0)", "article_view": "阅读(176)"}
{"diggnum": "0", "title": "补习系列(17)-springboot mongodb 内嵌数据库", "link": "https://www.cnblogs.com/littleatp/p/10462395.html", "post_item_summary": "[TOC] 简介 前面的文章中，我们介绍了如何在SpringBoot 中使用MongoDB的一些常用技巧。 那么，与使用其他数据库如 MySQL 一样，我们应该怎么来做MongoDB的单元测试呢？ 使用内嵌数据库的好处是不需要依赖于一个外部环境，如果每一次跑单元测试都需要依赖一个稳定的外部环境，那么 ...", "author": "美码师", "author_link": "https://www.cnblogs.com/littleatp/", "article_comment": "评论(0)", "article_view": "阅读(84)"}
......
......
......
```
- `cnblogs` 首页一般显示200页，每页20条数据，如果爬取不出错结果应该有4000条

### 特别说明
此爬虫具有时效性，如果源站改变html代码结构，爬虫就可能会失效。经过测试，截止2019年7月26日，爬虫依然有效。