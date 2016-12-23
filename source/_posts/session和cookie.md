---
title: session和cookie
date: 2016-12-22 18:35:35
categories: 
- web
tags:
- session
- cookie
---
### 基本认识  
首先有一点必须特别的清楚： 因为HTTP协议是无状态的，客户每次读取web页面时，服务器都打开新的会话，而且服务器也不会自动维护客户的上下文信息，对于一个浏览器发出的多次请求，WEB服务器无法区分 是不是来源于同一个浏览器，更别说是否是来自同一用户。为了保持用户的状态，有了两种机制，一般用户客户端的cookie机制，和用于服务器端的session机制，这两种机制都是为了保持状态，既有联系又有区别。
### cookie基本实现机制  
现在的cookie是HTTP协议的一部分，一般存在HTTP的响应头，内容是一系列的键值对的形式，简单说：cookie就是服务器在用户的浏览器中存储的一小段文本文件（大小不能超过3K）不包含任何可执行代码，里面一般包含的是用户的登录信息之类的比较少的，用来验证用户是否合法（不止局限与此，cookie是用来记录状态的，也可以是购物的等一系列状态，让服务器知道我们浏览了那些地方，购物对那些感兴趣，一般很多广告都是根据这个东西来推送的，你在某购物网站买了一个东西，然后浏览别的网站，发现广告都是和你浏览的相关）。  cookie的内容主要包括：key-value,Expires（过期时间），path和domain。path和domain一起构成cookie的作用范围。  
![](http://ofa8x9gy9.bkt.clouddn.com/cookie.png)  
&emsp;&emsp;&emsp;&emsp;现在的cookie，内容经过加密了
cookie的实现流程：  
1. 浏览器向某个URL发起HTTP请求 （可以是任何请求，比如GET，POST等）
2. 对应的服务器收到该HTTP请求，并做相应的响应（响应头和请求体两部分），在响应的头中加入`Set-Cookie`字段（设置相应的cookie，cookie是多个`key-value`组成的）
<!--more-->
3. 浏览器收到来自服务器的HTTP响应
4. 浏览器在响应头中发现存在`Set-Cookie`字段，就会将相应的cookie(key-value)保存在内存或者硬盘中。需要注意的是`Set-Value`字段可以包含多个cookie,每一项都可以指定过期时间，默认的过期的时间是用户浏览器关闭的时候
5. 浏览器下次给该服务器发送HTTP请求时，会将服务器设置的cookie附加在HTTP请求头`Cookie`中浏览器可以存储多个不同域名下的Cookie，但只发送当前请求的域名曾经指定的域名，这个可以在`Set-Cookie`中指定
6. 服务器收到这个HTTP请求，发现请求头中有`Cookie`字段，就知道这个用户的状态。获取相应的信息进行响应。  
这就是整个基本的cookie机制。保存了用户的操作状态，但是还需要注意的是,cookies是通过明文传递。在HTTP包中容易被劫持和伪造，是不安全的，不应该存一些比较重要的东西。还有就是cookie在整个会话都会在HTTP的请求中，增加了流量。     

### session的基本实现机制  
有一个session是不能改变的，是为了维持住HTTP的状态，所以在用户每次发起HTTP请求的时候，都需要让服务器知道是那个用户发起的这个请求，然后查找这个用户的状态，在进行相应的处理。session的实现基于这点，就需要一个唯一的ID标志某个用户（session）然后在这个ID中对应多个键值对来保证用户的状态，所以前后端只需要传递一个sessionId，服务器就可找到对应的状态（这个对应的键值对可以存在redies或者数据库中）。前后端传递值基本有三种：一种是直接写在URL中，一种通过表单中的隐藏域来提交，还有一种是现在流行的做法，通过在cookie中设置一个键值对`jsessionId=${sessionId}`在传递。现在第一二种都不是很建议这么做，当然在浏览器禁用的情况下也可以通过前面两种来传递，不过一般浏览器都支持使用cookie的方式。 

### 两种方式的区别和联系  

1. cookie数据存放在客户的浏览器上，session数据放在服务器上。
2. cookie不是很安全，别人可以分析存放在本地的cookie并进行cookie欺骗，考虑到安全应当使用session。
3. session会在一定时间内保存在服务器上。当访问增多，会比较占用你服务器的性能
   考虑到减轻服务器性能方面，应当使用cookie。
4. 单个cookie保存的数据不能超过4K，很多浏览器都限制一个站点最多保存20个cookie。
联系是：两个都是用来保持HTTP协议状态的方式，一种是客户端的实现，一种是服务器端的实现。但是session可以依赖于cookie来传递sessionId

### 一个小项目例子--在微信中开发的小程序（公众号自动回复）里面的session问题  
首先简单描述下小项目:用户在公众号中输入某一个触发关键词（项目里面的例子是：绑定），然后就进入绑定所涉及的流程。用户在公众中输入触发词，是由微信的服务器进行响应，然后转发到程序的服务器。程序的服务器在把处理的结果（用规定的格式）传递给微信的服务器，由微信的服务器进行响应给用户。整个项目用golang开发，简单的使用了beego框架。
#### 基于session来实现（初始版）
```go 
func (c *MainController) Dispatch() {
	//进行请求的分发，和request数据的解析,POST
	w := new(models.WeixinUser)
	xml.Unmarshal(c.Ctx.Input.RequestBody,&w)
	//第一步是判断是否为开启逻辑的语句，同一个逻辑只能同时开启一次
	str:=w.Content
	sc:=c.GetSession("status-count").(int)
	switch str {
	case "绑定": //如果匹配进入绑定，判断是否同时开启两次
		str0:=""
		binds:=assertionInt(c.GetSession("bindstep"))
		if binds==-1{
			c.SetSession("bindstep",int(1))
			sc++;
			c.SetSession("status-count",int(sc))
			c.SetSession("status-"+strconv.Itoa(sc),string("bind"))
			_,str0=bind(c,w)
		}
		if str0==""{
			str0="已经进入绑定流程"
		}
		prints(c,w,str0)
		return
	}
	//第二步判断有多少个session保持者状态，从0开始计数,输入的数据从最上面的应用进行匹配处理
	if sc >=0 {//判断是否有触发的逻辑
		for i:=sc;i>=0;i--{//进行逻辑匹配
			str2:=c.GetSession("status-"+strconv.Itoa(i)).(string)
			if str2!=""{
				str1:=" "//进行默认的逻辑处理，不做回应
				switch str2 {
				case "bind":
					code,src:=bind(c,w)//code用来处理完成逻辑后的删除session的作用
					if code==1{
						c.DelSession("status-"+strconv.Itoa(i))
						c.SetSession("status-count",int(sc-1))
						sc--;
					}
					str1=src
				}
				if str1 != " "{//如果进行处理就退出逻辑
					prints(c,w,str1)
					return
				}
				if str1 ==" "&&i==0{
					str1="输入格式错误"
					prints(c,w,str1)
					return
				}
			}
		}
	}
	str="输入：绑定，可以进入绑定流程"
	prints(c,w,str)
}
func bind(c *MainController,w *models.WeixinUser) (int,string) {
//进行绑定请求的处理，如果匹配不上，退出同时开始另一个逻辑的匹配
	step:=c.GetSession("bindstep").(string)//内部逻辑计数器，记住用户的处理的位置，从1开始计数
	if step==1||step==2||step==3||step==4{
		switch step {
		case 1://用户输入进入词，进入逻辑
			c.SetSession("bindstep",int(2))
			return 0,"请输入手机号，进行绑定"
		case 2://用户输入了手机号，发送短信，获取验证码
			//todo 验证手机号格式并且发短信验证验证码
			ok:=validatePhone(w.Content)
			if !ok{
				return 0,"手机号格式不正确，请重新输入"
			}
			alidayu.AppKey="*******"
			alidayu.AppSecret="**************"
			alidayu.UseHTTP=true
			str:=randNum()//生成随机4位数字
			success,_:=alidayu.SendSMS(w.Content,"德玛西亚","********", `{ "code":"`+str+`"}`)
			if !success{
				return 0,"验证码发送失败，请重新输入手机号"
			}
			c.SetSession("rand",str)
			f:=models.Fluge{}
			f.Openid=w.FromUserName
			f.Phone=w.Content
			c.SetSession("fluge",f)
			c.SetSession("bindstep",int(3))
			return 0,"验证码已发送，请输入验证码完成绑定"
		case 3://进行验证码的验证，成功就开始绑定
			if w.Content==c.GetSession("rand").(string) {
				f:=c.GetSession("fluge").(*models.Fluge)
				c.DelSession("fluge")
				c.DelSession("rand")
				flag,_:=models.CheckUser(f)
				if flag {
					str3, err := models.AddUser(f)
					if err != nil {
						return 0, str3
					}
					c.SetSession("bindstep", int(4))
					return 0, str3
				}
				c.SetSession("bindstep",int(2))
				return 0,"手机号无法重复绑定，请重新输入手机号"
			}
			return 0,"验证码错误，请重新输入"
		case 4://用户已经完成绑定，等待退出
			if w.Content=="8" {
				c.DelSession("bindstep")
				return 1,"谢谢使用"
			}
			return 0,"绑定完成，请输入8退出"
		}
	}
	return 0," "
}
```
  
现在这个程序是执行不了的。在beego中默认传递sessionId是cookie。但如果我在程序中必须使用session保持用户的状态（不然流程没办法继续下去）。两个服务器之间的交互式没有办法传递cookie。所以就会出现在一直在第一个流程，无法进入第二个流程。  
#### 解决
首先明白一点，sessionId的作用是一个唯一标识符，用来标记同一个用户。但是在微信的整个架构中有一个跟sessionId很类似的东西:openid:用户对于某个公众号唯一的标识。所以在微信服务器向程序的服务器提交POST消息市本身也会自带这个openid。这样就为解决session提供了遍历。  
不需要额外去产生和传递sessionId。直接使用openid来作为用户的唯一标识符。  
![](http://ofa8x9gy9.bkt.clouddn.com/%E5%BE%AE%E4%BF%A1%E5%85%AC%E4%BC%97%E5%8F%B7%E6%8E%A5%E5%8F%97%E6%B6%88%E6%81%AF.png)  
然后对于sessionId对应的具体内容我选择了方便的redis来存储  
实现代码--直接复写了beego的Getseeion和SetSession方法    

```golang
func (c *MainController)initSession(sid string) {
	config := &redissession.SessionConfig{
		Prefix:"lyfluge-",
		RedisHost:"**********",
		RedisPassword:"f***********",
		LifeTime: 60 * time.Second,
	}
	c.sesion = redissession.NewSession("redis",config)
	c.sesion.SetSessionID(sid)
	c.sesion.Start()
}
func (c *MainController)storeSession() {
	c.sesion.Store()
}
func (c *MainController) SetSession(str string,value interface{})interface{}{
	return c.sesion.Set(str,value)
}
func (c *MainController) GetSession(str string) interface{}{
	return c.sesion.Get(str)
}
func (c *MainController) DelSession(str string) interface{}{
	return c.sesion.Delete(str)
}
//在Dispatch函数中取到对应的sessionId，然后进行状态判断
func (c *MainController) Dispatch() {
	//进行请求的分发，和request数据的解析.POST
	w := new(models.WeixinUser)
	xml.Unmarshal(c.Ctx.Input.RequestBody,&w)
	c.initSession(w.FromUserName)
   /..../
}
```
完整代码(包括微信golang的接入,大鱼短信,beego,session-redis的具体实现):
#### 总结  
session中的sessionId是用来标志唯一用户的。通过找到这个用户来判断用户的状态  

----  
参考:[Cookie/Session的机制与安全](http://harttle.com/2015/08/10/cookie-session.html)