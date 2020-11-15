---
title: muuto_scrapy爬虫
tags: [爬虫]
index_img: https://gitee.com//L0yy/BlogImg/raw/master/typora/image-20200705164322714.png
banner_img: https://gitee.com//L0yy/BlogImg/raw/master/typora/image-20200705164322714.png
date: 2020-7-4
---


本项目是爬取一个家具网站的部分数据

目标站：https://muuto.com/furniture

### 前言

本爬虫使用scrapy框架

框架大概流程如下

![image-20200705164322714](https://gitee.com//L0yy/BlogImg/raw/master/typora/image-20200705164322714.png)

配图来源：https://doc.scrapy.org/en/master/topics/architecture.html

#### 下载样式

保存格式如下

![image-20200705164404797](https://gitee.com//L0yy/BlogImg/raw/master/typora/image-20200705164404797.png)



### 生成新对象

```shell
scrapy startproject sSofo		
```

创建一个scrapy空对象，方便后期使用

### 使用通用模板创建一个spider

```shell
  #scrapy genspidere [options] <name> <domain>
  scrapy genspidere xxxx xxxx.com
```

### 详细配置文件

上面生成默认配置文件后，下面贴出我修改的文件，没贴出的代码都默认处理

#### item.py

```python
#item.py
import scrapy

class SsofaItem(scrapy.Item):
    # define the fields for your item here like:
    # name = scrapy.Field()
    srcurl = scrapy.Field()
    videourl = scrapy.Field()
    title = scrapy.Field() 
```

#### spiders/example.py

```python
#spiders/example.py
import scrapy
from sSofa.items import SsofaItem

class ExampleSpider(scrapy.Spider):
    name = 'example'
    myheaders = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/53.0.2785.143 Safari/537.36',
    }

    def start_requests(self):
        #爬虫开始运行执行的第一个函数
        purl = "https://muuto.com/furniture"
        yield scrapy.Request(purl,headers = self.myheaders)

    def parse(self, response):
        #默认第一次返回的response处理函数
        div_xapth_list = response.xpath("//a[@class='product photo product-item-photo']/@href").extract()
        for each in div_xapth_list:
            # 注意下面的callback是sec_page，所以返回的数据就到下面去处理
            yield scrapy.Request(each,headers = self.myheaders,callback = self.sec_page)

    def sec_page(self,response):
        #处理第二次request返回的数据
        
        #item在这个框架中可以认为是一个全局变量一样的东西，给各个模块之间使用数据（个人理解）
        item = SsofaItem()

        title = response.xpath("//span[@data-ui-id='page-title-wrapper']/text()").extract()
        item['title'] = title 
        
        source_video = response.xpath("//section[@id='video-unbranded']//iframe[@id='player']/@src").extract()
        item['videourl'] = source_video

        xpathfilter ="//section[@id='fifty-fifty-image-slider-text']//img/@src | //section[@id='brand-images']//img/@src | //section[@id='designer']//img/@src"
        srcurl = response.xpath(xpathfilter).extract()
        item['srcurl'] = srcurl

        # srcurl = response.xpath("//section[@id='fifty-fifty-image-slider-text']//img/@src").extract()
        # item['srcurl'] = srcurl

        # brand_images = response.xpath("//section[@id='brand-images']//img/@src").extract()
        # item['srcurl'].append(brand_images)

        # designer = response.xpath("//section[@id='designer']//img/@src").extract()
        # item['srcurl'].append(designer)

        #这里返回item后，如果没有开启pipelimes，就到这里结束
        #如果打开后，就去执行pipelimes中的代码
        yield item
```
#### pipelines.py

```python
#pipelines.py
import scrapy
from scrapy.pipelines.images import ImagesPipeline

#注意这个类是继承自ImagesPipeline 而不是 object 
class SsofaPipeline(ImagesPipeline):

    myheaders = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/53.0.2785.143 Safari/537.36',
    }
    
    def get_media_requests(self,item,info):
        #这里是重写父类ImagesPipeline的对应方法，建议对照父类代码理解
        #我这里item中的srcurl是用xpath获取的数组，所以要遍历取每个值
        for each in item['srcurl']:
            #注意最后这个meta，因为file_path中没传入item，所以想用的话可以通过这种方式传过去给file-path 使用
            yield scrapy.Request(each,headers = self.myheaders,meta={'item':item})

    def file_path(self, request, response=None, info=None):
        image_guid = request.url.split("/")[-1].split('.')[0]
        #前一个表示目录，后一个表示保存的文件名
        path = "{}/{}.jpg".format(request.meta['item']['title'][0],image_guid)
        return path
```
### settings.py

```python
#settings.py
import os
#使用ImagesPipeline 下载的文件保存在这，名字不能乱改
IMAGES_STORE = os.path.join(os.path.dirname(os.path.dirname(__file__)),'Sofo_images')
#关闭机器人协议，否则某些网站会获取不到数据
ROBOTSTXT_OBEY = False

#这个很重要，默认是未开启的，也就是默认SsofaPipeline中的代码不执行，需要在这接触注释
ITEM_PIPELINES = {
   'sSofa.pipelines.SsofaPipeline': 1,
}


# 可以在这设置默认request headers
# DEFAULT_REQUEST_HEADERS = {
#             'User-Agent': 'Mozilla/5.0 (Windows NT 5.1) AppleWebKit/536.3 (KHTML, like Gecko) Chrome/19.0.1063.0 Safari/536.3',
#             'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
#             "Accept-Language": "zh-CN,zh;q=0.9,en-US;q=0.5,en;q=0.3",
#             "Accept-Encoding": "gzip, deflate",
#             'Content-Length': '0',
#             "Connection": "keep-alive"
#             }
```

### 执行

爬虫名是在`spiders/example.py`中`name`设置的

命令行运行

`scrapy crawl example`

脚本运行

```python
from scrapy import cmdline

cmdline.execute('scrapy crawl example'.split())
```

网上还有很多其他运行方式