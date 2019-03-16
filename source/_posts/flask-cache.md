---
title: Flask-Cache缓存使用小记
date: 2017-11-21 22:19:00
tags:
- flask
categories:
- python
---
# 原因

这两天一直在玩我的[SiYuYong Player](http://119.29.169.58:8888)播放器的时候，发现有时候会出现`Connection reset by peer`的错误，然后服务器一直返回不了响应，原因是：

```
"Connection reset by peer" is the TCP/IP equivalent of slamming the phone back on the hook. It's more polite than merely not replying, leaving one hanging. But it's not the FIN-ACK expected of the truly polite TCP/IP converseur.
```

就是说因为客户端频繁请求，服务器粗暴的关闭了请求连接，所以出现了这个错误。

## 解决方案

因为首页的播放歌单不要实时获取更新，所以使用缓存的方法来解决这个问题。对于大规模互联网应用，缓存是必不可少的，一个好的缓存设计可以使得应用的性能几何级数地上升。我们使用缓存后，服务器获取歌单数据就更快了。

# 缓存原理

首先我们来看一个缓存服务器，Werkzeug框架中的提供了一个简单的缓存对象[SimpleCache](http://werkzeug.pocoo.org/docs/0.11/contrib/cache/#werkzeug.contrib.cache.SimpleCache)，它是将缓存项存放在Python解释器的内存中，我们可以用下面的代码获取SimpleCache的缓存对象：

``` python
from werkzeug.contrib.cache import SimpleCache
cache = SimpleCache()
```

它也可以使用`memcached`、`redis`、`filesystem`等作为缓存服务器，具体用哪个看你的需求，需要注意的是`SimoleCache`缓存服务器不适合在生产服务器中使用，因此你需要在生产环境中使用其他的缓存服务器。

你可以使用cache对象的“set(key, value, timeout)”和”get(key)”方法来操作缓存项。注意”set()”方法的第三个参数”timeout”是缓存过期时间，默认为0，也就是永不过期。

接下来我们来看一个缓存装饰器：

``` python
def cached(timeout=5 * 60, key='view_%s'):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            cache_key = key % request.path
            value = cache.get(cache_key)
            if value is None:
                value = f(*args, **kwargs)
                cache.set(cache_key, value, timeout=timeout)
            return value
        return decorated_function
    return decorator
```

很显然这个缓存装饰器默认的缓存时间是5min，使用的缓存key是`view_`+`request.path`。然后我们写个视图来使用它：

``` python
@app.route('/hello')
@cached()
def hello():
    print 'run hello!'
    return render_template('hello.html)
```

尝试访问这个视图你会发现只有第一次控制台会输出`run hello!`，第二次则不会了，过5min后再访问又会输出了。

## 使用Flask-Cache进行缓存处理

上面我们看了一个自定义缓存装饰器的例子。下面我们来看`Flask`的一个扩展`Flask-Cache`的使用。

像其他的`Flask`扩展使用一样，你可以先实例化`cache = Cache()`，然后再初始化：
``` python
cache.init_app(app, config={
        'CACHE_TYPE': 'filesystem',
        'CACHE_DIR': './flask_cache',
        'CACHE_DEFAULT_TIMEOUT': 922337203685477580,
        'CACHE_THRESHOLD': 922337203685477580
    }
)
```

我这里选择使用文件系统的方式来缓存数据。为什么呢？因为我用的服务器内存小了一点，为了不影响服务器性能，所以使用文件系统来缓存。你可以选择更加高效的`memcached`或者`redis`来缓存数据。

`Flask-Cache`实例可以直接使用`@cache.cached()`来装饰函数：

``` python
@app.route('/hello')
@cache.cached()
def hello():
    print 'run hello!'
    return render_template('hello.html)
```

你还可以传入`timeout`、`key_prefix`等参数控制缓存配置，但是，问题来了，我的视图函数是使用`GET`参数来返回结果的，这个参数是使用`request.path`作为键来缓存函数返回值的，就是说即使`GET`参数不同，他返回的结果是不变的，这个就很烦了，达不到我要的效果。该怎么办呢？中间我曾有一度放弃使用`Flask-Cache`的想法，但是我又觉得`Flask-Cache`不应该没有这个实现，于是我去它的官方文档里面找解决办法。

## 问题解决

我重新读了一遍`Flask-Cache`文档，发现文档有一处地方是这么写的：

```
key_prefix – Default ‘view/%(request.path)s’. Beginning key to . use for the cache key.
New in version 0.3.4: Can optionally be a callable which takes no arguments but returns a string that will be used as the cache_key.
```

这个意思是说在最新的0.3.4版本中。`key_perfix`参数的值可以是一个可调用对象，也就是函数，于是，问题解决了，我们先自定义一个函数返回key值，然后把函数作为`key_perfix`的值传给`cache`：

``` python
def cache_key():
    args = request.args
    key = request.path + '?' + urllib.urlencode([
        (k, v) for k in sorted(args) for v in sorted(args.getlist(k))
    ])
    return key

@app.route('/hello')
@cache.cached(key_prefix=cache_key)
def hello():
    print 'run hello!'
    return render_template('hello.html)
```

把这个应用到服务器上去，不同的`GET`参数对应不同的键值，因此也可以返回不同的值了。

# 参考

[Flask-Cache — Flask-Cache 0.13 documentation](https://pythonhosted.org/Flask-Cache/)

[Cache — Werkzeug Documentation (0.11)](http://werkzeug.pocoo.org/docs/0.11/contrib/cache/#werkzeug.contrib.cache.SimpleCache)


