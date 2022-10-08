---
title: Python 爬虫学习笔记
date: 2022-06-29 21:18:51
tags:
---

# Python 爬虫

## 1. urllib

### 1.1. 参考文档

https://docs.python.org/3/howto/urllib2.html

### 1.2. 抓取 url

```python
import urllib.request
with urllib.request.urlopen('http://www.baidu.com/') as response:
    html = response.read()
```

如果你想抓取 url 并存储到临时文件夹下：
```python
import shutil
import tempfile
import urllib.request

with urllib.request.urlopen('http://python.org/') as response:
    with tempfile.NamedTemporaryFile(delete=False) as tmp_file:
        shutil.copyfileobj(response, tmp_file)

with open(tmp_file.name) as html:
    pass
```

定制化 Request

```python
import urllib.request

req = urllib.request.Request('http://www.voidspace.org.uk')
with urllib.request.urlopen(req) as response:
   the_page = response.read()
```

#### Data

```python
import urllib.parse
import urllib.request

url = 'http://www.someserver.com/cgi-bin/register.cgi'
values = {'name' : 'Michael Foord',
          'location' : 'Northampton',
          'language' : 'Python' }

data = urllib.parse.urlencode(values)
data = data.encode('ascii') # data should be bytes
req = urllib.request.Request(url, data)
with urllib.request.urlopen(req) as response:
   the_page = response.read()
```
## 2. requests

https://requests.readthedocs.io/en/latest/


### 快速开始

```python
import requests
r = requests.get('https://api.github.com/events')
r = requests.post('https://httpbin.org/post', data={'key': 'value'})
r = requests.put('https://httpbin.org/put', data={'key': 'value'})
r = requests.delete('https://httpbin.org/delete')
r = requests.head('https://httpbin.org/get')
r = requests.options('https://httpbin.org/get')
# 设置编码格式
r.encoding = 'UTF-8'
# 返回响应字符串
r.text
# 返回 url 地址
r.url
# 返回响应字节
r.content
# 响应状态码
r.status_code
# 响应头
r.headers
```

### GET 请求

```python
url = 'https://www.baidu.com/s'
headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/102.0.0.0 Safari/537.36'
}
params = {
    'wd': '北京'
}
response = requests.get(url=url, params=params, headers=headers)
```

# BS4
```python
# 返回数组
soup.select('#haha')
```

# selenium
## headless


```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

def share_browser():
    chrome_options = Options()
    chrome_options.add_argument('--headless')
    chrome_options.add_argument('--disable-gpu')

    path = 'C:\Program Files\Google\Chrome\Application\chrome.exe'
    chrome_options.binary_location = path

    return webdriver.Chrome(chrome_options=chrome_options)
```


## requests

```bash
pip install requests
```

### Get 请求

```python
import requests

url = 'https://www.baidu.com/s'
headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/102.0.0.0 Safari/537.36'
}
params = {
    'wd': '北京'
}
response = requests.get(url=url, params=params, headers=headers)
content = response.text
```


## scrapy

以下这段话来自 [https://scrapy.org/](https://scrapy.org/)

> An open source and collaborative framework for extracting the data you need from websites.
> In a fast, simple, yet extensible way.

scrapy 是一个开源和协作框架，用于从网站上提取所需的数据。
以快速，简单但可扩展的方式。


### 安装错误总结

> 来自于尚硅谷 Python 视频

1. 缺少 twisted.test.raiser 扩展


http://www.lfd.uci.edu/~gohlke/pythonlibs/#twisted


2. 升级 pip

```bash
python -m pip install --upgrade pip
```



### 快速开始

1. 创建爬虫项目

```bash
scrapy startproject myproject
```
> myproject 为自定义项目名称。项目名称必须以字母开头，并且只包含字母、数字、下划线。

命令执行完毕会得到如下目录：

```bash
├─myproject
│  │  scrapy.cfg
│  │
│  └─myproject
│      │  items.py
│      │  middlewares.py
│      │  pipelines.py
│      │  settings.py
│      │  __init__.py
│      │
│      └─spiders
│              __init__.py
```


1. 创建爬虫文件

```bash
scrapy genspider 爬虫名 爬取网页
```


1. 运行爬虫代码

```bash
scrapy crawl 爬虫名
```


### response

获取响应字符串

```python
response.text
```


获取响应二进制数据

```python
response.body
```


获取