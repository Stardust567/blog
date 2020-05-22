---
title: Scrapy爬虫框架
comments: true
abbrlink: b2a
date: 2019-07-07 11:44:36
tags:
  - 大二
  - 爬虫
  - scrapy
categories: Python
---

Scrapy是一个为了爬取网站数据，提取结构性数据而编写的应用框架。

以下整理于自己业余写的简单爬虫 [theGuardianNews](https://github.com/Stardust567/theGuardianNews)路过朋友有兴趣可以看看。 <!-- More -->

## 创建项目

创建新Scrapy项目，在存储代码的目录下git bash
`scrapy startproject news`

这将创建一个名为*news*的目录，文件tree如下：

news
│  scrapy.cfg&emsp; &emsp; &emsp; # deploy configuration file
└─news&emsp; &emsp; &emsp; &emsp; &emsp; # project Python module, import your code from here
&emsp; │  items.py&emsp; &emsp; &emsp; # project items definition file
&emsp; │  middlewares.py&emsp; # project middlewares file
&emsp; │  pipelines.py&emsp; &emsp; # project pipelines file
&emsp; │  settings.py&emsp; &emsp; # project settings file
&emsp; │  \_\_init\_\_.py
&emsp; └─spiders&emsp; &emsp; &emsp; # a directory where you will later put your spiders
&emsp; &emsp; │  \_\_init\_\_.py



## 编写Spiders

定义Spider类用来从网站中提取信息。

1. 必须子类化 `scrapy.Spider`并定义要生成的初始请求
2. 可选地如何跟踪页面中的链接
3. 解析下载的页面内容以提取数据

在news目录下cmd输入`scrapy genspider news theguardian.com`

```python
import scrapy

class news(scrapy.Spider):
    name = "news"
    start_urls = [
        'https://www.theguardian.com/',
    ]
    '''start_urls将默认执行yield scrapy.Request故可省略以下：
    for url in urls:
        yield scrapy.Request(url=url, callback=self.parse)
    '''
    def parse(self, response):
        pass
```

`parse()`方法通常解析响应，将抽取的数据提取为dicts，并查找要遵循的新URL并`Request`从中创建新的request()

## 提取数据

强烈安利用`scrapy shell url`的方式先在shell里试试，这样不会运行几次就被反爬。

`::text`表示抽取标签内字符串，`::attr(title)`表示抽取title属性内容

`extract()`为一个包含数据串的list，`extract_first()`为list的第一个值

```python
urls = response.css('a[class="fc-item__link"]').css('a[data-link-name="article"]').xpath('@href').extract()
title = response.css('h1[class ="content__headline "]').css('h1[itemprop="headline"]::text').extract()
time = response.css('time[itemprop = "datePublished"]::text').extract_first()
category = response.css('a[class ="subnav-link subnav-link--current-section"]::text').extract_first()
tags = response.css('a[class = "submeta__link"]::text').extract()
content = response.css('div[itemprop = "articleBody"]').css('p::text').extract()
```

## 创建Item

打开news文件夹下的items.py创建类NewsItem

```python
class NewsItem(scrapy.Item):
    title = scrapy.Field()
    time = scrapy.Field()
    category = scrapy.Field()
    tags = scrapy.Field()
    content = scrapy.Field()
```

## 设置Pipeline

Item pipeline组件有两个典型作用：1. 查重丢弃 ；2. 保存数据到文件或数据库中。

假设我们采用本地json存取：

```python
from scrapy.exceptions import DropItem
import json
import codecs

class NewsPipeline(object):
    
    def process_item(self, item, spider):
        title = item['title']
        if title:
            title_str = ''.join(title)
            item['title'] = title_str.replace('\n', '')
        else: // 如果title为空则跳过这个item
            raise DropItem('Information was missing')
        time = item['time']
        if time:
            item['time'] = time.replace('\n', '')
        else:
            raise DropItem('Information was missing')
        category = item['category']
        if category:
            item['category'] = category.replace('\n', '')
        else:
            raise DropItem('Information was missing')
        tags = item['tags']
        if tags:
            tags_str = ','.join(tags)
            item['tags'] = tags_str.replace('\n', '')
        else:
            raise DropItem('Information was missing')
        content = item['content']
        if content:
            content_str = ''.join(content)
            item['content'] = content_str.replace('\n', '')
        else:
            raise DropItem('Information was missing')
            
class JsonWriterPipeline(object):
    def __init__(self):
        self.file = codecs.open('data_utf8.json', 'w', encoding='utf-8')
		// 如果是中文需要utf8的话，import codecs
    def process_item(self, item, spider):
        line = json.dumps(dict(item), ensure_ascii=False) + "\n"
        self.file.write(line)
        return item
    def spider_closed(self, spider):
        self.file.close()
```

之后别忘了在setting中改一下`ITEM_PIPELINES`的设置：

```python
ITEM_PIPELINES = {
    'news.pipelines.NewsPipeline': 300,
    'news.pipelines.JsonWriterPipeline': 800,
}
```

假设我们采用MongoDB，打开news文件夹下的pipelines.py创建类NewsPipline

```python
from scrapy.exceptions import DropItem
import pymongo
import codecs
import logging

class NewsPipeline(object):
    
    def process_item(self, item, spider):
        title = item['title']
        if title:
            title_str = ''.join(title)
            item['title'] = title_str.replace('\n', '')
        else: // 如果title为空则跳过这个item
            raise DropItem('Information was missing')
        time = item['time']
        if time:
            item['time'] = time.replace('\n', '')
        else:
            raise DropItem('Information was missing')
        category = item['category']
        if category:
            item['category'] = category.replace('\n', '')
        else:
            raise DropItem('Information was missing')
        tags = item['tags']
        if tags:
            tags_str = ','.join(tags)
            item['tags'] = tags_str.replace('\n', '')
        else:
            raise DropItem('Information was missing')
        content = item['content']
        if content:
            content_str = ''.join(content)
            item['content'] = content_str.replace('\n', '')
        else:
            raise DropItem('Information was missing')
            
class MongoPipeline(object):
	def __init__(self):
        # 链接数据库
        client = pymongo.MongoClient(host=settings['MONGO_HOST'], port=settings['MONGO_PORT'])
        self.db = client[settings['MONGO_DB']]  # 获得数据库的句柄
        self.coll = self.db[settings['MONGO_COLL']]  # 获得collection的句柄
        # 数据库登录需要帐号密码的话
        # self.db.authenticate(settings['MONGO_USER'], settings['MONGO_PSW'])
 
    def process_item(self, item, spider):
        postItem = dict(item)  # 把item转化成字典形式
        self.coll.insert(postItem)  # 向数据库插入一条记录
        return item  # 会在控制台输出原item数据，可以选择不写
```

同样我们需要在setting中改一下`ITEM_PIPELINES`的设置：

```python
ITEM_PIPELINES = {
    'news.pipelines.NewsPipeline': 300,
    'news.pipelines.MongoPipeline': 800,
}
MONGO_HOST = "127.0.0.1"  # 主机IP
MONGO_PORT = 27017  # 端口号
MONGO_DB = news_tutorial"  # 库名
MONGO_COLL = "news_items"  # collection名
# MONGO_USER = "simple" #用户名
# MONGO_PSW = "test" #用户密码
```

## 修改Spider

```python
from news.items import NewsItem
def get_news(self, response):
        item = NewsItem()
        ......# 提取数据的过程
        item['title'] = title
        item['time'] = time
        item['category'] = category
        item['tags'] = tags
        item['content'] = content
        yield item
```

修改完的spider如下：

``` python
import scrapy
from scrapy.http import Request
from news.items import NewsItem

class HeadSpider(scrapy.Spider):
    name = 'news'
    allowed_domains = ['theguardian.com']
    base_url = 'https://www.theguardian.com/'
    start_urls = []
    topics = ['world', 'science', 'cities', 'global-development',
              'uk/sport', 'uk/technology', 'uk/business', 'uk/environment', 'uk/culture']
    years = [2019]
    months = ['jan', 'feb', 'mar', 'apr', 'may', 'jun',
              'jul', 'aug', 'sep', 'oct', 'nov', 'dec']
    dates = range(1, 31)

    for topic in topics:
        for y in years:
            for m in months:
                for d in dates:
                    url = base_url + topic + '/' + str(y) + '/' + m + '/' + '%02d' % d + '/' + 'all'
                    start_urls.append(url) 
                    # 类似于 https://www.theguardian.com/uk/sport/2019/apr/01/all
                    
    def parse(self, response):
        urls = response.css('a[class="fc-item__link"]').css('a[data-link-name="article"]').xpath('@href').extract()
        for url in urls: 
            # 每个https://www.theguardian.com/uk/sport/2019/apr/01/all页面上的news连接
            yield Request(url, self.get_news)
    
    def get_news(self, response):
        item = NewsItem()
        title = response.css('h1[class ="content__headline "]').css('h1[itemprop="headline"]::text').extract()
        if(len(title)==0): 
            # 卫报的news页面有两种形式，如果第一种没抓到，就用模式二
            title = response.css('meta[itemprop="description"]').xpath('@content').extract()
        time = response.css('time[itemprop = "datePublished"]::text').extract_first()
        category = response.css('a[class ="subnav-link subnav-link--current-section"]::text').extract_first()
        tags = response.css('a[class = "submeta__link"]::text').extract()
        content = response.css('div[itemprop = "articleBody"]').css('p::text').extract()
        if (len(content) == 0):
        content = title # 有些news内容可能时视频或者其它非文本模式，这时我们用title作为content
    
        item['title'] = title
        item['time'] = time
        item['category'] = category
        item['tags'] = tags
        item['content'] = content
    
        yield item
```

## 修改设置

打开news文件夹下的settings.py可以修改一些小细节便于我们爬取数据。

TIPS：加上`FEED_EXPORT_ENCODING = 'utf-8'`适用于爬取中文内容哦。

```python
FEED_EXPORT_ENCODING = 'utf-8' # 修改编码为utf-8
DOWNLOAD_TIMEOUT = 10 # 下载超时设定，超过10秒没响应则放弃当前URL
ITEM_PIPELINES = {
   'news.pipelines.NewsPipeline': 300, // 这个没改，参照pipe看个人选择
}
CONCURRENT_REQUESTS = 32 # 最大并发请求数（默认16
DOWNLOAD_DELAY = 0.01 # 增加爬取延迟，降低被爬网站服务器压力
DEFAULT_REQUEST_HEADERS = {
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8',
    'Accept-Encoding': 'gzip, deflate',
    'Accept-Language': 'zh-CN,zh;q=0.8',
    'User-Agent': 'Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/59.0.3071.109 Safari/537.36'
}
```

## 针对反爬

去奥斯汀访学（旅游）时候写的yelp爬虫 [yelpReview](<https://github.com/Stardust567/yelp>)路过朋友有兴趣可以看看。

- 使用user agent池，轮流选择之一来作为user agent；池中包含常见的浏览器的user agent。
- 禁止cookies(参考 [`COOKIES_ENABLED`](https://scrapy-chs.readthedocs.io/zh_CN/0.24/topics/downloader-middleware.html#std:setting-COOKIES_ENABLED))，有些站点会使用cookies来发现爬虫的轨迹。
- 设置下载延迟(2或更高)。参考 [`DOWNLOAD_DELAY`](https://scrapy-chs.readthedocs.io/zh_CN/0.24/topics/settings.html#std:setting-DOWNLOAD_DELAY) 设置。
- 如果可行，使用 [Google cache](http://www.googleguide.com/cached_pages.html) 来爬取数据，而不是直接访问站点。
- 使用IP池。例如免费的 [Tor项目](https://www.torproject.org/) 或付费服务([ProxyMesh](http://proxymesh.com/))。
- 使用分布式下载器(downloader)来绕过禁止(ban)，就只需要专注分析处理页面。eg: [Crawlera](http://crawlera.com/)

## Mongodb

创建userAdminAnyDatabase角色，用来管理用户，可以通过这个角色来创建、删除用户。

```shell
> use admin
switched to db admin
> db.createUser(
...   {
...     user: "userAdmin",//用户名
...     pwd: "123",//密码
...     roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
...   }
... )
Successfully added user: {
    "user" : "dba",
    "roles" : [
        {
            "role" : "userAdminAnyDatabase",
            "db" : "admin"
        }
    ]
}
```

以上是我怕忘了自己设置过的user和pwd各位游客可以不用在意（小声）

打开安装mongoDB环境的命令行，比如我之前是在Windows下安装的，故打开cmd输入`mongo`会有如下输出：

```shell
MongoDB shell version v3.4.10-58-g9847179
connecting to: mongodb://127.0.0.1:27017
MongoDB server version: 3.4.10-58-g9847179
```

然后我个人安利下可视化工具adminMongo下载轻松简单（主要是界面好看清爽）

打开adminMongo目录，`npm start`会有如下输入，根据提示打开http://localhost:1234即可

```shell
> admin-mongo@0.0.23 start D:\adminMongo
> node app.js

adminMongo listening on host: http://localhost:1234
```

打开页面后，connection名字仅供参考（随便设）MongoDB连接字符串的格式可以是：`mongodb://<user>:<password>@127.0.0.1:<port>/<db>`指定<db>级别是可选的，一般`mongodb://:@127.0.0.1:27017`即可。
有关MongoDB连接字符串的更多信息，请参阅[正式的MongoDB文档](http://docs.mongodb.org/manual/reference/connection-string/)。

## 写入依赖

打开项目所在文件目录，然后打开命令行输入即可得到requirements.txt

```shell
pipreqs ./ --encoding=utf-8
```

这样之后有人需要用这个项目的时候直接打开命令行输入以下即可：

```shell
pip install -r requirements.txt
```

