---
title: 博客SEO优化小记
date: 2017-10-20 15:27:12
tags: 
- 搜索引擎
categories: 
- 技术杂谈
---
# 前言

在我将博客建立完成，发布了几篇文章后，发现在百度和google上搜不到我的博客，真是悲伤，上网查阅资料后发现需要进行SEO优化，然后自己也花了一天时间，摸索了一下SEO优化过程，终于是把博客显示在百度和google首页了。下图是优化后的效果。

![](/images/seo/blog_google_search.png)

# 优化过程

## 首页标题优化

首先打开index.swig文件`(site\themes\next\layout)`

```
把
{% block title %} {{ config.title }} {% endblock %}
改为
{% block title %} {{ config.title }} - {{ theme.description }} {% endblock %}
```

这时候首页会符合一般 *网站名称 - 网站描述* 的格式，若是想加入关键字进行进一步优化，则可以把代码进一步改为

``` 
{% block title %} {{ theme.keywords }} - {{ config.title }}{{ theme.description }} {% endblock %}
```

**注意**：首页标题字数有限制，一般不超过80个字符，请勿堆砌关键字，否则出现首页显示...省略号等情况。

## 添加sitemap站点地图

>站点地图是一种文件，您可以通过该文件列出您网站上的网页，从而将您网站内容的组织架构告知 Google 和其他搜索引擎。Googlebot 等搜索引擎网页抓取工具会读取此文件，以便更加智能地抓取您的网站。
>此外，站点地图能够提供与其中所列网页相关的宝贵元数据：元数据是网页的相关信息，例如此网页的上次更新时间、更改频率及其重要性（与相应网站中的其他网址相较而言）。

* 安装sitemap站点地图自动生成插件

``` bash
npm install hexo-generator-sitemap --save
npm install hexo-generator-baidu-sitemap --save
```

* 在hexo的站点配置文件中添加配置

``` 
sitemap: 
  path: sitemap.xml
baidusitemap:
  path: baidusitemap.xml
```

* 在站点配置文件中修改为你的域名，例如

``` 
url: https://heshangbuxitou.github.io/
```

然后执行`hexo g -g`命令生成静态文件,这时你可以在`site\public`发现`sitemap.xml`和 `baidusitemap.xml`了，显然，`sitemap.xml`是要提交给google的，而`baidusitemap.xml`则会提交给百度。

* 在your-hexo-site\source中新建文件robots.txt，内容如下：

``` http
User-agent: *
Allow: /
Allow: /archives/
Allow: /categories/
Allow: /tags/ 
Disallow: /vendors/
Disallow: /js/
Disallow: /css/
Disallow: /fonts/
Disallow: /vendors/
Disallow: /fancybox/


Sitemap: https://heshangbuxitou.github.io/sitemap.xml
Sitemap: https://heshangbuxitou.github.io/baidusitemap.xml
```

其中Allow表示的是你的menu;

**注意：要将`heshangbuxitou.github.io`改为你自己的域名**
然后`hexo d -g`提交

## 给非友情链接的出站链接添加 “nofollow” 标签
1. 找到footer.swig,路径在`site\themes\next\layout\_partials`，将下面代码

``` h
{{ __('footer.powered', '<a class="theme-link" href="http://hexo.io">Hexo</a>') }}
改成
{{ __('footer.powered', '<a class="theme-link" href="http://hexo.io" rel="external nofollow">Hexo</a>') }}
```

将下面代码

``` h
<a class="theme-link" href="https://github.com/iissnan/hexo-theme-next">
改成
<a class="theme-link" href="https://github.com/iissnan/hexo-theme-next" rel="external nofollow">

```

2. 修改sidebar.swig文件，路径在site\themes\next\layout\_macro，将下面代码

``` h
<a href="{{ link }}" target="_blank">{{ name }}</a>
改成
<a href="{{ link }}" target="_blank" rel="external nofollow">{{ name }}</a>
```

将下面代码
``` h
<a href="http://creativecommons.org/licenses/{{ theme.creative_commons }}/4.0" class="cc-opacity" target="_blank">
改成
<a href="http://creativecommons.org/licenses/{{ theme.creative_commons }}/4.0" class="cc-opacity" target="_blank" rel="external nofollow">
```

## 注册Google Search Console

链接：[https://www.google.com/webmasters/](https://www.google.com/webmasters/)
根据提示注册好之后，把你的博客域名加进去。

![](/images/seo/google_Console.png)

如图，添加完了之后可以点击域名进入：

![](/images/seo/FireShot Capture 8 - Search Console_dashboard.png)

## 测试robots.txt

点击左侧的robots.txt测试工具，根据提示提交你的robots.txt，其实刚才我们已经提交了。

![](/images/seo/Console_robots-testing-tool.png)

注意要0错误才可以，如果有错误的话，会有提示，改正确就可以了。

## 验证blog网站


### google验证

google验证比较简单。

进入 Google 搜索引擎入口，`添加属性 > 备用方法 > HTML 标记`，将 Google 的 html 标签，添加到 `site/themes/next/layout/_partials/head.swig` 文件中，重新发布 blog，点击验证就可以了。

``` html
<meta name="google-site-verification" content="-rILxxxxx7gbfxxxxx-E1VWxxxxxTcq6pxgs_xxxxx" />
```

### baidu验证

baidu验证的话比较麻烦，因为github是https，而百度不支持https验证（会出现`301`错误），一般标签验证和文件比较麻烦，如果可以的话，请选CNAME的验证方式。

我因为懒得搞域名和DNS解析而用不了第三种方法，所以在baidu验证这里卡住了，花了较多的时间去网上寻找解决方案，以下是我综合了几种方案后最终采用的一种方案。

* 首先在网站配置文件`_config.yml`中修改验证文件的渲染属性，比如我设置忽略渲染

```
skip_render: [google2b659428d9daf7b6.html, baidu_verify_65gIf28eKK.html]
```

* 把百度验证的文件下载下来放到`source`目录下，执行`hexo g -g`生成静态文件，
之后在点击百度静态文件验证。


## 提交站点地图
还记得我们刚才创建创建`sitemap.xml`文件吧,现在它要派上用场了。点击左侧工具栏的`站点地图`。

![](/images/seo/站点地图.png)

这里我已经添加了，所以你看到的和我看到界面应该不一样，然后点右上角的添加/测试站点地图。输入`sitemap.xml`先点测试，如果没问题的话，再提交。

## Google 抓取方式

提交站点地图之后，点击左侧的Google 抓取工具

![](/images/seo/FireShot Capture 13.png)

在这里我们填上我们需要抓取的url,不填这表示抓取首页，抓取方式可以选择桌面，智能手机等等，自行根据需要选择。填好url之后，点击抓取。
然后可能会出现几种情况，如:完成、部分完成、重定向等，自由这三种情况是可以提交的。
提交完成后，提交至索引，根据提示操作就可以了。我的提交：

![](/images/seo/FireShot Capture 14.png)

至此，你的博客在google搜索上排名想不靠前都难了，马上到google搜索一下你的关键词和博客title测试一下吧；

## baidu提交站点地图

百度提交站点地图的方式与google类似，这里不再阐述，但是这里有一个问题。

你即使提交了baidu站点地图也无法让百度收录你的网站，因为`Github`禁止了百度的爬虫，所以我们需要主动提交网页到百度站点工具。

可以通过推送脚本把网页主动向百度提交，推送脚本如下。其中的url = XXXXXXXXXXXXXXXXXXXXXXXXX，需要打开http://zhanzhang.baidu.com/linksubmit/index 找到自己的推送接口填上去

``` python
import os
import sys
import json
from bs4 import BeautifulSoup as BS
import requests
import msvcrt

"""
hexo 博客专用，向百度站长平台提交所有网址

本脚本必须放在hexo博客的根目录下执行！需要已安装生成百度站点地图的插件。
百度站长平台提交链接：http://zhanzhang.baidu.com/linksubmit/index 从中找到自己的接口调用地址
主动推送：最为快速的提交方式，推荐您将站点当天新产出链接立即通过此方式推送给百度，以保证新链接可以及时被百度收录。

"""


url = XXXXXXXXXXXXXXXXXXXXXXXXX
baidu_sitemap = os.path.join(sys.path[0], 'public', 'baidusitemap.xml')
google_sitemap = os.path.join(sys.path[0], 'public', 'sitemap.xml')
sitemap = [baidu_sitemap, google_sitemap]

assert (os.path.exists(baidu_sitemap) or os.path.exists(
    google_sitemap)), "没找到任何网站地图，请检查！"


# 从站点地图中读取网址列表
def getUrls():
    urls = []
    for _ in sitemap:
        if os.path.exists(_):
            with open(_, "r", encoding="utf-8") as f:
                xml = f.read()
            soup = BS(xml, "html.parser")
            tags = soup.find_all("loc")
            urls += [x.string for x in tags]
            if _ == baidu_sitemap:
                tags = soup.find_all("breadCrumb", url=True)
                urls += [x["url"] for x in tags]
    return urls


# POST提交网址列表
def postUrls(urls):
    urls = set(urls)  # 先去重
    print("一共提取出 %s 个网址" % len(urls))
    data = "\n".join(urls)
    return requests.post(url, data=data).text


if __name__ == '__main__':

    urls = getUrls()
    result = postUrls(urls)
    print("提交结果：")
    print(result)
    msvcrt.getch()
```

**注意：**百度收到推送后，会用爬虫爬一次新增的页面。当场就收录了。
当天的收录结果要到第二天才能在站长平台的后台看到，「网页抓取」、「索引量」，能看到详细的图表。

## 参考

[Hexo Seo优化让你的博客在google搜索排名第一](http://www.jianshu.com/p/86557c34b671)

[个人博客搜索引擎优化](https://chenyuzuoo.github.io/posts/40228/)

[如何解决百度爬虫无法爬取搭建在Github上的个人博客的问题](https://www.zhihu.com/question/30898326)





