---
layout:     post
title:      "武汉大学图书馆预约系统"
subtitle:   "仅供学习使用"
date:       2018-03-08
author:     "RAY"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 爬虫
---


本篇文章主要分享我最近练习并得到了实践的python3.x爬虫技术，并在学校图书馆做了简单的尝试，能够成功的预约到自己心仪的座位（当然，仅供学习使用，不建议用此脚本投机取巧，破坏预约秩序）。

首先，简单介绍一下我从零开始一点点完成脚本的学习历程。
> * 使用fiddler从8888端口进行抓包并解析每一个session
> * http中request的get和post方法，请求headers的内容以及cookie的作用
> * urllib包中request和response的使用
> * 使用BeautifulSoup进行HTML网页的解析
> * python3中datetime使用方法

这些技术初学还是需要一些时间来掌握，等熟练之后就会发现爬虫就是一种熟能生巧的工作，多多练习会让对各种意外情况的处理更加迅速。下面介绍一下我一步步分析的流程。
# 模拟登录
## 准备工作
首先打开fiddler软件，在chrome浏览器中打开[武汉大学图书馆预约官网](http://seat.lib.whu.edu.cn/)。

![Markdown](http://i4.bvimg.com/634149/22d111e4b1ece5b6s.png)

发现有一次的跳转，根据多次练习的经验，直接访问不带后缀的主机时，会获得一个cookie以便进行后来的登录验证和验证码的固定生成，查看16号session

![Markdown](http://i4.bvimg.com/634149/24bfecce5fe34380s.png)

HTTP请求没有含有任何的cookie信息，得到了一个cookie的返回，这个返回的cookie将是进行登录以及以后操作的基础，查看17号session以验证我们的猜想

![Markdown](http://i4.bvimg.com/634149/2a44128f7ecfea79s.png)

可以看到，请求跳转的链接时携带了刚刚获得的cookie，并且返回了登录的界面，由此cookie的获得已经可以确定。

下面在登录界面输入自己的学号和密码，发现一个重要的细节，图书馆的网页登录设置了验证码，这将是爬虫的一个大麻烦，一开始想通过一些API来解决验证码识别的难题来解决，使用了百度ocr识别和谷歌tesseract之后发现效果不太满意，错误率太高，若要更加精确的识别还需要自己进行深度学习神经网络的训练，太过繁琐，为了快些完成爬虫项目，这些进阶工作暂且放下，为此我采取了直接显示验证码图片手动输入的方式。查看验证码的链接为*/simpleCaptcha/captcha*,在fiddler中发现每一次请求验证码地址都携带了初始的cookie，这样验证码不会刷新，这对爬虫来说是一个好消息，验证码的有效时间长可以保证用户输入后绝对有效。

在浏览器中登录成功，查看登录的请求为post方法，携带的数据有username,password,captcha,地址为*/auth/signIn*

![Markdown](http://i4.bvimg.com/634149/d7696639fad52314s.png)

## 开始模拟
首先就是基本的Request初始化工作，注意post方法带data即可，不带data的Request就为get方法，post方法主要用于表单的提交和提交数据等，大部分对地址的请求等都为get方法。另外要提的是，request.urlopen方法是一种特殊的opener，如果要获取cookie，那么应该自己构建带cookie的opener，如果不需要cookie，直接使用此方法即可。
```python
index_url = 'http://seat.lib.whu.edu.cn/'
cookie = cookiejar.CookieJar()
handler = request.HTTPCookieProcessor(cookie)
opener = request.build_opener(handler)
index_req = request.Request(index_url)
index_req.add_header('User-Agent','Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.146 Safari/537.36')
index_res = opener.open(index_req)
index_cookie = ''
for item in cookie:
    index_cookie += item.name + '=' + item.value + ';'
```
上述代码展示了如何从访问主机并且获取index_cookie
然后是验证码图片的抓取，注意在请求中一定要在headers中加上cookie，这样才能保证此验证码能长时间有效，爬取到的数据只是一堆二进制字符需要进行文件的写入(open方法中'wb'代表写入，'rb'代表读取)，此外还需要用到pillow包（PIL），使用PIL的插件展示图片，以便我们能直接看清验证码。
```python
captcha_url='http://seat.lib.whu.edu.cn/simpleCaptcha/captcha'
req=request.Request(captcha_url,headers=headers)
res=request.urlopen(req)
with open('captcha.png','wb') as fp:
    fp.write(res.read())
image=Image.open('captcha.png')
if image:
        print('验证码已经获取')
image.show()
```
获取到验证码后便可以让用户输入所看到的字符了，登录需要的数据也收集完毕，下面便可使用post方法将数据提交给服务器，让本地的cookie在主机服务器中得到注册长时间有效，根据fiddler查看返回的cookie信息可知，此cookie的有效时间为两个小时。
```python
headers={
        'User-Agent':'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.146 Safari/537.36',
        'Cookie':index_cookie
    }
login_url='http://seat.lib.whu.edu.cn/auth/signIn'
login_data={
        'username':username,
        'password':password,
        'captcha':captcha
    }
login_req=request.Request(login_url,parse.urlencode(login_data).encode(),headers)
login_res=request.urlopen(login_req)
```
上述代码中的headers由于已经注册，便可以长期进行使用了，下面所有的请求的headers都是这个。

至此已经完成了模拟登录。
# 查询座位
## 寻找接口
介绍一下武汉大学信息部图书馆的座位分布，图书馆一共四层，第一层是讨论区和电脑区，讨论区的座位是圆桌型的，不太适合自习，而由于自己一般都会使用自带的电脑电脑区也基本上没有什么用处，因此第一层pass，从第二层开始每层都有两个自习室，第三层还有一个自主学习区，因此这七个房间的座位就是本次爬虫预约的目标。

刚刚开始进行爬虫编写的时候我的想法是直接寻找预约座位的地址进行预约，从第二层的西房间开始，如果预约失败了那么预约当前房间的下一号座位，直到有满足要求的座位为止，但是实际实践下来由于会进行频繁的预约请求，除去各种形形色色的返回异常不说，图书馆的服务器还会有相应的反爬虫机制封锁掉当前的IP，严重者设置会被封禁学号，因此我调整了我的预约策略，搜索为主，最后的预约请求应该尽量的少。

观察图书馆的主页，发现图书馆直接提供了座位查询的功能，因此点击此功能进行查询，并且用fiddler进行抓包,网页结果以及抓包结果如下。

![Markdown](http://i1.bvimg.com/634149/4b3edde1ca4fa806s.png)

![Markdown](http://i1.bvimg.com/634149/f9db81c2df924da7s.png)

不难发现，采用post方法进行请求，返回的是一个json文件，其中seatStr字段内为可用座位列表的html，每一个li标签的id属性为当前座位的id，而dt标签则是此id对应的房间内的座位号，为了简单，我们提取第一个li标签的座位进行预约即可。

## 提取出座位id
下面重要的任务便是如何对json文件和html进行解析以获得最终的座位id以及座位号（方便展示），座位id将是最后进行预约的主要数据。查看此session可以知道需要post的数据有onDate,building（这里只考虑信息学部图书馆，building号为1）,room,hour,startMin,endMin,power(是否需要有电源座位),window（是否靠窗），下面随意设置一些数据进行试验。

解析json的工具为python自带的json包，使用json.loads即可将json格式的文件转化为python的字典结构。得到字典后首先用seatNum的key得到当前房间查询到的可用座位数，若为0即可开始下一个房间的查询，不为0则查询seatStr得到座位的html。

得到html后需要使用BeautifulSoup将其转化为一个soup对象，beautifulsoup有几种解析模式，lxml、html.parser、xml等，这里使用html.parser进行解析，得到soup对象后，观察html内容，发现每一个座位都在li标签内，每一个座位的座位号都在dt标签内，因此使用soup.p（p为标签名，返回得到的第一个标签）可以直接得到查询到的第一个座位，再使用.attrs('属性名')即可获得id属性的内容，再用字符串的从后往前截取的方法即可。代码如下：
```python
search_url = 'http://seat.lib.whu.edu.cn/freeBook/ajaxSearch'
search_data = {
            'onDate': date,
            'building': '1',
            'room':room,
            'hour':'null',
            'startMin':start_time,
            'endMin':end_time,
            'power':'null',
            'window':'1',
        }
search_req=request.Request(search_url,parse.urlencode(search_data).encode(),headers)
search_res=request.urlopen(search_req)
search_json=json.loads(search_res.read().decode('utf-8'))
if search_json['seatNum']!=0:
    find=True
    seat_html=search_json['seatStr']
    seat_soup=BeautifulSoup(seat_html,'html.parser')
    seat_id=seat_soup.li.attrs['id'][-4:]
    seat_num=seat_soup.dt.text
```
## 搜索优化
注意到座位还有靠窗和不靠窗之分，出于自己的一点点私心当然要优先预约靠窗的座位了，因此安排了前后两次查询，先查询靠窗的座位，若没有找到座位则再查询不靠窗的座位，各种循环的结构这里不再贴出了。
# 座位预定
## 抓包准备
自己先尝试进行座位预约，摸清楚各种预约结果所返回的信息。

![Markdown](http://i1.bvimg.com/634149/af2521250a66f498s.png)

发现所有的返回信息都在dd标签内，根据此线索即可进行预约和结果的打印。
## 预约工作
同样是对预约地址进行post，直接用BeautifulSoup进行解析，使用findAll方法，获得的list进行分别打印即可。当然，预约之前要用此cookie再获取一次验证码。
```python
getCaptcha(headers)
captcha=input('请输入预定验证码')
book_data={
     'date':date,
     'seat':seat_id,
     'start':start_time,
     'end':end_time,
     'captcha':captcha
    }
book_req=request.Request(book_url,parse.urlencode(book_data).encode(),headers)
book_res=request.urlopen(book_req)
book_soup=BeautifulSoup(book_res.read().decode('utf-8'),'html.parser')
results=book_soup.find_all('dd')
for result in results:
    if '成功' in result.text:
        book=True
    print(result.text)
```
这样，基本上预约工作就完成了。
# 程序改进

基本功能的完成并不就意味着能够完美的符合我们的需求，学校的座位抢座时间一般是在十点半，因此最好能让程序自动在每天十点半时自动运行，这需要用到datetime包里面的方法，使用while无限循环，每一次都获取当前的时间，由于还带有微秒数，因此需要对此时间在十点半和十点半过一秒内进行判断，如果符合则执行上述的程序，注意，这里还需要暂停一秒钟以使得下一次循环到之后必定不会还在当前的范围内，避免多次执行程序，这样就达到了定时执行程序的效果且资源占用很小。
```python
if __name__=='__main__':
    a=input('1、现在运行，2、定时运行（22：30）')
    if a=='1':
        function()
    if a=='2':
        schedual_time = datetime.datetime(2018, 3, 10, 22, 30, 0)
        loop = 0
        while True:
            if schedual_time <= datetime.datetime.now() < schedual_time + datetime.timedelta(seconds=1):
                loop = 1
                time.sleep(1)
                if loop == 1:
                    function()
                    loop == 0
```
# 总结
本次爬虫练习基本上涵盖了爬虫所有的主流技术，还有一些python数据结构的运用，掌握之后对今后进行更复杂的结构化爬虫工作奠定了很好的基础。当然还有一些后话，如何运用深度学习的tensorflow框架对爬取的大量类似的验证码图片集进行训练以提高验证码识别精度从而改善程序体验，还有使用代理IP和高斯随机数的暂停来应对当前网站的反爬虫机制，以及分布式爬虫提高爬虫的效率，都是今后可以深入研究的地方。

本项目的[地址](https://github.com/RayVec/WHUlib_Book),欢迎大家前来阅读。