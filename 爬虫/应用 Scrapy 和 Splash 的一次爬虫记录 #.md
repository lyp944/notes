# 应用 Scrapy 和 Splash 的一次爬虫记录 #

## 爬携程 http://vacations.ctrip.com/tours/r-ouzhou-120002#_flta ##

###难点###
* 爬去网页是根据窗口可见部分异步加载的，需要模拟用户滚动操作实现抓取
### 准备工作：安装 scrapy 和 splash ###
* 安装scrapy：https://docs.scrapy.org/en/latest/intro/install.html
* 安装splash：https://splash.readthedocs.io/en/stable/install.html（记得运行splash服务）
* scrapy上配置splash：https://github.com/scrapy-plugins/scrapy-splash

### 创建项目 ###

1. 打开终端`cd`到你要创建项目的文件夹
2. scrapy创建项目

```shell
scrapy startproject xiecheng
```

3. cd到spider文件夹中创建第一个spider

```shell
cd ./spider
scrapy genspider vocation vacations.ctrip.com
```

4. 成功后 vocation.py 会有出始的模版代码 修改为：


```python
# -*- coding: utf-8 -*-
import scrapy
from scrapy_splash import SplashRequest

lua = '''
  function main(splash)

    local num_scrolls = 10
    local scroll_delay = 0.5

    local scroll_to = splash:jsfunc("window.scrollTo")
    local get_body_height = splash:jsfunc(
      "function() {return document.body.scrollHeight;}"
    )
    assert(splash:go(splash.args.url))
    splash:wait(splash.args.wait)

    local height = get_body_height()
    local y = 0

	    --[[
	     因为页面是根据窗口可见部分异步加载的，所以
	     循环的作用是模拟人为的滚动来使页面完整加载，
	     用于爬去数据，每循环一次等待0.25s
	     ]]--

    repeat
       y = y+100
       height = get_body_height()
       scroll_to(0,y)
       splash:wait(0.25)

    until(y>=height)
    splash:wait(1)

    splash:set_viewport_full()
    splash:wait(0.2)

    return splash:html()
    --return splash:png()
  end
  '''


class VocationSpider(scrapy.Spider):
  name = 'vocation'
  allowed_domains = ['vacations.ctrip.com']
  start_urls = ['http://vacations.ctrip.com/tours/r-europe-120002#_flta', ]

  def go_splashRequest(self, url):
    return SplashRequest(url, endpoint='execute', args={'wait': 2, 'lua_source': lua, 'timeout': 30},
               callback=self.parse)

  def start_requests(self):
    for url in self.start_urls:
      yield self.go_splashRequest(url)
      # yield scrapy.Request(url,callback=self.parse)

  def parse(self, response):

    # debug
    # from scrapy.shell import inspect_response
    # inspect_response(response, self)

    # 通过css选择器来提取内容
    list = response.css('.main_mod.product_box.flag_product')

    for l in list:
      unit = l.css(".product_detuct .sr_price dfn::text").get()
      money = l.css(".product_detuct .sr_price strong::text").get()

      title = l.css(".product_title a::text").get()
      city = l.css(".start_info dt::text").get()

      yield {
        'title': str(title),
        'city': str(city),
        'price': str(unit) + str(money),
      }

    # 截图
    # with open("a.png", 'wb') as f:
    #     f.write(response.body)
	
	#递归抓取下一页
    next_page = response.css('#_pg > a.down::attr(href)').get()
    if next_page is not None:
      next_page = response.urljoin(next_page)
      yield self.go_splashRequest(next_page)

```
	
	5. 项目配置settings.py
	
```python
# -*- coding: utf-8 -*-

# Scrapy settings for xiecheng project
#
# For simplicity, this file contains only settings considered important or
# commonly used. You can find more settings consulting the documentation:
#
#     https://doc.scrapy.org/en/latest/topics/settings.html
#     https://doc.scrapy.org/en/latest/topics/downloader-middleware.html
#     https://doc.scrapy.org/en/latest/topics/spider-middleware.html

BOT_NAME = 'xiecheng'

SPIDER_MODULES = ['xiecheng.spiders']
NEWSPIDER_MODULE = 'xiecheng.spiders'


# Crawl responsibly by identifying yourself (and your website) on the user-agent
#USER_AGENT = 'xiecheng (+http://www.yourdomain.com)'

# Obey robots.txt rules
ROBOTSTXT_OBEY = False  #改为False 不遵守robots.txt rules

# Configure maximum concurrent requests performed by Scrapy (default: 16)
#CONCURRENT_REQUESTS = 32

# Configure a delay for requests for the same website (default: 0)
# See https://doc.scrapy.org/en/latest/topics/settings.html#download-delay
# See also autothrottle settings and docs
DOWNLOAD_DELAY = 3
# The download delay setting will honor only one of:
CONCURRENT_REQUESTS_PER_DOMAIN = 16
#CONCURRENT_REQUESTS_PER_IP = 16

# Disable cookies (enabled by default)
COOKIES_ENABLED = True

# Disable Telnet Console (enabled by default)
#TELNETCONSOLE_ENABLED = False

# Override the default request headers:
DEFAULT_REQUEST_HEADERS = {
  'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
  "Accept-Language":"zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3",
}

# Enable or disable spider middlewares
# See https://doc.scrapy.org/en/latest/topics/spider-middleware.html
SPIDER_MIDDLEWARES = {
   'scrapy_splash.SplashDeduplicateArgsMiddleware': 100,
   #'xiecheng.middlewares.XiechengSpiderMiddleware': 543,
}

# Enable or disable downloader middlewares
# See https://doc.scrapy.org/en/latest/topics/downloader-middleware.html
DOWNLOADER_MIDDLEWARES = {
    'scrapy_fake_useragent.middleware.RandomUserAgentMiddleware': 400, #开源randomUserAgent自动替换userAgent
    'scrapy_splash.SplashCookiesMiddleware': 723,
    'scrapy_splash.SplashMiddleware': 725,
    'scrapy.downloadermiddlewares.httpcompression.HttpCompressionMiddleware': 810,
    'xiecheng.middlewares.XiechengDownloaderMiddleware': 543,
}

# Enable or disable extensions
# See https://doc.scrapy.org/en/latest/topics/extensions.html
#EXTENSIONS = {
#    'scrapy.extensions.telnet.TelnetConsole': None,
#}

# Configure item pipelines
# See https://doc.scrapy.org/en/latest/topics/item-pipeline.html
#ITEM_PIPELINES = {
#    'xiecheng.pipelines.XiechengPipeline': 300,
#}

# Enable and configure the AutoThrottle extension (disabled by default)
# See https://doc.scrapy.org/en/latest/topics/autothrottle.html
#AUTOTHROTTLE_ENABLED = True
# The initial download delay
#AUTOTHROTTLE_START_DELAY = 5
# The maximum download delay to be set in case of high latencies
#AUTOTHROTTLE_MAX_DELAY = 60
# The average number of requests Scrapy should be sending in parallel to
# each remote server
#AUTOTHROTTLE_TARGET_CONCURRENCY = 1.0
# Enable showing throttling stats for every response received:
#AUTOTHROTTLE_DEBUG = False

# Enable and configure HTTP caching (disabled by default)
# See https://doc.scrapy.org/en/latest/topics/downloader-middleware.html#httpcache-middleware-settings
#HTTPCACHE_ENABLED = True
#HTTPCACHE_EXPIRATION_SECS = 0
#HTTPCACHE_DIR = 'httpcache'
#HTTPCACHE_IGNORE_HTTP_CODES = []
#HTTPCACHE_STORAGE = 'scrapy.extensions.httpcache.FilesystemCacheStorage'

```

6. 在终端输入scrapy crawl vocation -o a.jl 就可以爬到携程的数据了




