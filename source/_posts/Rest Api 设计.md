---
title: Rest Api 设计
categories:
    - 编程
        - 软件设计
tags:
    - Rest
---

RESTful是目前最流行的API设计规范，它是用于Web数据接口的设计。从字面可以看出，他是Rest式的接口，所以我们先了解下什么是Rest。

REST与技术无关，它代表的是一种软件架构风格，REST它是 Representational State  Transfer的简称，中文的含义是: "表征状态转移" 或  "表现层状态转化"。它是基于HTTP、URI、XML、JSON等标准和协议，支持轻量级、跨平台、跨语言的架构设计。



## 为什么使用RESTful API

在很久以前，工作时间长的同学肯定经历过使用velocity语法来编写html模板代码，也就是说我们的前端页面放在服务器端那边进行编译的，更准确的可以理解为  "前后端没有进行分离"，那么在那个时候，页面、数据及模板渲染操作都是放在服务器端进行的，但是这样做有一个很大的缺点是: 后期维护比较麻烦，前端开发人员还必须掌握velocity相关的语法。因此为了解决这个问题慢慢就出现了前后端分离的思想: 即后端负责数据接口, 前端负责数据渲染,  前端只需要请求下api接口拿到数据，然后再将数据显示出来。因此后端开发人员需要设计api接口，因此为了统一规范: 社区就出现了 RESTful  API 规范，其实该规范很早就有的，只是最近慢慢流行起来，RESTful API  可以通过一套统一的接口为所有web相关提供服务，实现前后端分离。

<!-- more -->

## 理解REST

简单搜索就能找到 REST 源于 Roy Thomas Fielding 在 2000 年发表的博士论文：《[Architectural Styles and the Design of Network-based Software Architectures](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm)》，此文的确是 REST 的源头，但我们不应该忽略 Fielding 的身份和此前的工作背景，这些信息对理解 REST 的设计思想至关重要。

首先，Fielding 是一名很优秀的软件工程师，他是 Apache 服务器的核心开发者，后来成为了著名的[Apache 软件基金会](https://www.apache.org/)的联合创始人；同时，Fielding 也是 HTTP 1.0 协议（1996 年发布）的专家组成员，后来还晋升为 HTTP 1.1 协议（1999 年发布）的负责人。HTTP 1.1 协议设计得极为成功，以至于发布之后长达十年的时间里，都没有收到多少修订的意见。用来指导 HTTP 1.1 协议设计的理论和思想，最初是以备忘录的形式用作专家组成员之间交流，除了 IETF、W3C 的专家外，并没有在外界广泛流传。

从时间上看，对 HTTP 1.1 协议的设计工作贯穿了 Fielding 的整个博士研究生涯，当起草 HTTP 1.1 协议的工作完成后，Fielding 回到了加州大学欧文分校继续攻读自己的博士学位。第二年，他更为系统、严谨地阐述了这套理论框架，并且以这套理论框架导出了一种新的编程思想，他为这种程序设计风格取了一个很多人难以理解，但是今天已经广为人知的名字 REST，即“表征状态转移”的缩写。

哪怕对编程和网络都很熟悉的同学，只从标题中也不太可能直接弄明白什么叫“表征”、啥东西的“状态”、从哪“转移”到哪。尽管在论文原文中确有论述这些概念，但写得确实相当晦涩（不想读英文的同学从此处[获得中文翻译版本](https://www.infoq.cn/article/2007/07/dlee-fielding-rest/)），笔者推荐比较容易理解 REST 思想的途径是先理解什么是 HTTP，再配合一些实际例子来进行类比，你会发现“REST”（**Re**presentational **S**tate **T**ransfer）实际上是“HTT”（**H**yper**t**ext **T**ransfer）的进一步抽象，两者就如同接口与实现类的关系一般。

HTTP 中使用的“超文本”（Hypertext）一词是美国社会学家 Theodor Holm Nelson 在 1967 年于《[Brief Words on the Hypertext](https://archive.org/details/SelectedPapers1977)》一文里提出的，下面引用的是他本人在 1992 年修正后的定义：

>Hypertext
>
>By now the word "hypertext" has become generally accepted for branching and responding text, but the corresponding word "hypermedia", meaning complexes of branching and responding graphics, movies and sound – as well as text – is much less used.
>
>现在，"超文本 "一词已被普遍接受，它指的是能够进行分支判断和差异响应的文本，相应地， "超媒体 "一词指的是能够进行分支判断和差异响应的图形、电影和声音（也包括文本）的复合体。
>
>—— Theodor Holm Nelson [Literary Machines](https://en.wikipedia.org/wiki/Literary_Machines), 1992

以上定义描述的“超文本（或超媒体，Hypermedia）”是一种“能够对操作进行判断和响应的文本（或声音、图像等）”，这个概念在上世纪 60 年代提出时应该还属于科幻的范畴，但是今天大众已经完全接受了它，互联网中一段文字可以点击、可以触发脚本执行、可以调用服务端，这一切已毫不稀奇。下面我们继续尝试从“超文本”或者“超媒体”的含义来理解什么是“表征”以及 REST 中其他关键概念，这里使用一个具体事例将其描述如下。

- **资源**（Resource）：譬如你现在正在阅读一篇名为《REST 设计风格》的文章，这篇文章的内容本身（你可以将其理解为其蕴含的信息、数据）我们称之为“资源”。无论你是购买的书籍、是在浏览器看的网页、是打印出来看的文稿、是在电脑屏幕上阅读抑或是手机上浏览，尽管呈现的样子各不相同，但其中的信息是不变的，你所阅读的仍是同一份“资源”。
- **表征**（Representation）：当你通过电脑浏览器阅读此文章时，浏览器向服务端发出请求“我需要这个资源的 HTML 格式”，服务端向浏览器返回的这个 HTML 就被称之为“表征”，你可能通过其他方式拿到本文的 PDF、Markdown、RSS 等其他形式的版本，它们也同样是一个资源的多种表征。可见“表征”这个概念是指信息与用户交互时的表示形式，这与我们软件分层架构中常说的“表示层”（Presentation Layer）的语义其实是一致的。
- **状态**（State）：当你读完了这篇文章，想看后面是什么内容时，你向服务器发出请求“给我下一篇文章”。但是“下一篇”是个相对概念，必须依赖“当前你正在阅读的文章是哪一篇”才能正确回应，这类在特定语境中才能产生的上下文信息即被称为“状态”。我们所说的有状态（Stateful）抑或是无状态（Stateless），都是只相对于服务端来说的，服务器要完成“取下一篇”的请求，要么自己记住用户的状态：这个用户现在阅读的是哪一篇文章，这称为有状态；要么客户端来记住状态，在请求的时候明确告诉服务器：我正在阅读某某文章，现在要读它的下一篇，这称为无状态。
- **转移**（Transfer）：无论状态是由服务端还是客户端来提供的，“取下一篇文章”这个行为逻辑必然只能由服务端来提供，因为只有服务端拥有该资源及其表征形式。服务器通过某种方式，把“用户当前阅读的文章”转变成“下一篇文章”，这就被称为“表征状态转移”。

通过“阅读文章”这个例子，笔者对资源等概念进行通俗的释义，你应该能够理解 REST 所说的“表征状态转移”的含义了。借着这个故事的上下文状态，笔者再继续介绍几个现在不涉及但稍后要用到的概念名词。

- **统一接口**（Uniform Interface）：上面说的服务器“通过某种方式”让表征状态发生转移，具体是什么方式？如果你真的是用浏览器阅读本文电子版的话，请把本文滚动到结尾处，右下角有下一篇文章的 URI 超链接地址，这是服务端渲染这篇文章时就预置好的，点击它让页面跳转到下一篇，就是所谓“某种方式”的其中一种方式。任何人都不会对点击超链接网页会出现跳转感到奇怪，但你细想一下，URI 的含义是统一资源标识符，是一个名词，如何能表达出“转移”动作的含义呢？答案是 HTTP 协议中已经提前约定好了一套“统一接口”，它包括：GET、HEAD、POST、PUT、DELETE、TRACE、OPTIONS 七种基本操作，任何一个支持 HTTP 协议的服务器都会遵守这套规定，对特定的 URI 采取这些操作，服务器就会触发相应的表征状态转移。

- **超文本驱动**（Hypertext Driven）：尽管表征状态转移是由浏览器主动向服务器发出请求所引发的，该请求导致了“在浏览器屏幕上显示出了下一篇文章的内容”这个结果的出现。但是，你我都清楚这不可能真的是浏览器的主动意图，浏览器是根据用户输入的 URI 地址来找到网站首页，服务器给予的首页超文本内容后，浏览器再通过超文本内部的链接来导航到了这篇文章，阅读结束时，也是通过超文本内部的链接来再导航到下一篇。浏览器作为所有网站的通用的客户端，任何网站的导航（状态转移）行为都不可能是预置于浏览器代码之中，而是由服务器发出的请求响应信息（超文本）来驱动的。这点与其他带有客户端的软件有十分本质的区别，在那些软件中，业务逻辑往往是预置于程序代码之中的，有专门的页面控制器（无论在服务端还是在客户端中）来驱动页面的状态转移。

- **自描述消息**（Self-Descriptive Messages）：由于资源的表征可能存在多种不同形态，在消息中应当有明确的信息来告知客户端该消息的类型以及应如何处理这条消息。一种被广泛采用的自描述方法是在名为“Content-Type”的 HTTP Header 中标识出[互联网媒体类型](https://en.wikipedia.org/wiki/Media_type)（MIME type），譬如“Content-Type : application/json; charset=utf-8”，则说明该资源会以 JSON 的格式来返回，请使用 UTF-8 字符集进行处理。

  

## Rest API 设计原则

那么怎么样可以设计成REST的架构规范呢? 需要符合如下的一些原则：

1、每一个URI代表一种资源;
2、同一种资源有多种表现形式(xml/json);
3、所有的操作都是无状态的。
4、规范统一接口。
5、返回一致的数据格式。 
6、可缓存(客户端可以缓存响应的内容)。

符合上述REST原则的架构方式被称作为 RESTful 规范。

### 为什么操作需要无状态

http请求本身是无状态的，它是基于 client-server  架构的，客户端向服务器端发的每一次请求都必须带有充分的信息能够让服务器端识别到，请求的一些信息通常会包含在URL的查询参数中或header中，服务器端能够根据请求的各种参数, 不需要保存客户端的状态,  直接将数据返回给客户端。无状态的优点是：可以大大提高服务器端的健状性和可扩展性。客户端可以通过token来标识会话状态。从而可以让该用户一直保持登录状态。

### 理解规范统一的接口

Rest接口约束定义为: 资源识别；请求动作；响应信息; 

它表示通过uri表示出要操作的资源，通过请求动作(http method)标识要执行的操作，通过返回的状态码来表示这次请求的执行结果。

可能看上面的解释还不够理解，下面通过自己的理解来解释下上面的具体含义; 比如说，我在未使用Rest规范之前，我们可能有 增删改查 等接口，因此我们会设计出类似这样的接口: /xxx/newAdd (新增接口), /xxx/delete(删除接口), /xxx/query (查询接口), /xxx/uddate(修改接口)等这样的。增删改查有四个不同的接口，维护起来可能也不好，因此如果我们现在使用Restful规范来做的话，对于开发设计来说可能就只需要一个接口就可以了，比如设计该接口为 /xxx/apis 这样的一个接口就可以了，然后请求方式(method)有 GET--查询(从服务器获取资源); POST---新增(从服务器中新建一个资源); PUT---更新(在服务器中更新资源)，DELETE---删除(从服务器删除资源)，PATCH---部分更新(从服务器端更新部分资源) 等这些方式来做，也就是说我们使用RESTful规范后，我们的接口就变成了一个了，要执行增删改查操作的话，我们只需要使用不同的请求动作(http method)方式来做就可以了，然后服务器端返回的数据也可以是相同的，只是我们前端会根据状态码来判断请求成功或失败的状态值来判断。具体有那些状态码我们下面会讲解到。

### 返回一致的数据格式

服务器端返回的数据格式可以是XML、或json. 或者直接返回状态码的方式。

比如返回错误的格式json数据如下:

```json
{
    "code": 401,
    "status": "error",
    "message": '用户没有权限',
    "data": null
}
```

返回正确的数据格式的json数据一般可以为如下:

```json
{
    "code": 200,
    "status": "success",
    "data": [{
        "userName": "tugenhua",
        "age": 31
    }]
}
```

在json中的属性使用小驼峰命名法。

不要使用下划线（ `year_of_birth`）或大驼峰命名法（ `YearOfBirth`）。通常，RESTful Web服务将被JavaScript编写的客户端使用。客户端会将JSON响应转换为JavaScript对象（通过调用 `varperson=JSON.parse(response)`），然后调用其属性。因此，最好遵循JavaScript代码通用规范。
对比：

- `person.year_of_birth`  // 不推荐，违反JavaScript代码通用规范
- `person.YearOfBirth`      // 不推荐，JavaScript构造方法命名
- `person.yearOfBirth`      // 推荐



## URL及参数设计规范

### uri设计规范

1、uri末尾不需要出现斜杠/
2、在uri中使用斜杠/是表达层级关系的。
3、在uri中可以使用连接符-, 来提升可读性。比如 http://xxx.com/xx-yy 比 http://xxx.com/xx_yy中的可读性更好。
4、在uri中不允许出现下划线字符_.
5、在uri中尽量使用小写字符。
6、在uri中不允许出现文件扩展名. 比如接口为 /xxx/api, 不要写成 /xxx/api.php 这样的是不合法的。
7、在uri中使用复数形式。
8、只使用名词。
9、非资源请求用动词。

具体可以看：（https://blog.restcase.com/7-rules-for-rest-api-uri-design/）

在RESTful架构中，每个uri代表一种资源，因此uri设计中不能使用动词，只能使用名词，并且名词中也应该尽量使用复数形式。使用者应该使用相应的http动词 GET、POST、PUT、PATCH、DELETE等操作这些资源即可。

那么在我们未使用RESTful规范之前，我们是如下方式来定义接口的，形式是不固定的，并且没有统一的规范。比如如下形式:

```bash
http://xxx.com/api/getallUsers;  // GET请求方式，获取所有的用户信息
http://xxx.com/api/getuser/1;    // GET请求方式，获取标识为1的用户信息
http://xxx.com/api/user/delete/1 // GET、POST 删除标识为1的用户信息
http://xxx.com/api/updateUser/1  // POST请求方式 更新标识为1的用户信息
http://xxx.com/api/User/add      // POST请求方式，添加新的用户
```

如上我们可以看到，在未使用Restful规范之前，接口形式是不固定的，没有统一的规范，下面我们来看下使用RESTful规范的接口如下，两者之间对比下就可以看到各自的优点了。

```bash
http://xxx.com/api/users;     // GET请求方式 获取所有用户信息
http://xxx.com/api/users/1;   // GET请求方式 获取标识为1的用户信息
http://xxx.com/api/users/1;   // DELETE请求方式 删除标识为1的用户信息
http://xxx.com/api/users/1;   // PATCH请求方式，更新标识为1的用户部分信息
http://xxx.com/api/users;     // POST请求方式 添加新的用户
```

如上我们可以看到，增删改成我们都是使用同一个api接口，只是请求的方式 GET(查询)、POST(新增)、DELETE(删除)、PACTH(部分更新)来代表的是增删改查操作的方式。然后开发获取到该请求的header头部信息，就可以知道是什么方式来请求数据的了。

### HTTP请求规范

| HTTP方法       | 动作 | 含义                                               |
| -------------- | ---- | -------------------------------------------------- |
| GET (SELECT)   | 查询 | 从服务器取出资源                                   |
| POST(CREATE)   | 新增 | 在服务器上新建一个资源                             |
| PUT(UPDATE)    | 更新 | 在服务器上更新资源(客户端提供改变后的**完整资源**) |
| PATCH(UPDATE)  | 更新 | 在服务器上更新**部分资源**(客户端提供改变的属性)   |
| DELETE(DELETE) | 删除 | 从服务器上删除资源                                 |

### 参数命名规范

参数推荐采用下划线命名的方式。比如如下demo:

```bash
http://xxx.com/api/today_login // 获取今天登录的用户。
http://xxx.com/api/today_login&sort=login_desc // 获取今天登录的用户、登录时间降序排序。
```



## http状态码相关的

### 状态码范围

客户端的每一次请求, 服务器端必须给出回应，回应一般包括HTTP状态码和数据两部分。

1xx: 信息，请求收到了，继续处理。
2xx: 代表成功，行为被成功地接收、理解及采纳。
3xx: 重定向。
4xx: 客户端错误，请求包含语法错误或请求无法实现。
5xx: 服务器端错误。

### 2xx 状态码

200 OK [GET]: 服务器端成功返回用户请求的数据。
201 CREATED [POST/PUT/PATCH]: 用户新建或修改数据成功。
202 Accepted 表示一个请求已经进入后台排队(一般是异步任务)。
204 NO CONTENT -[DELETE]: 用户删除数据成功。

### 4xx状态码

400：Bad Request - [POST/PUT/PATCH]: 用户发出的请求有错误，服务器不理解客户端的请求，未做任何处理。
401: Unauthorized; 表示用户没有权限(令牌、用户名、密码错误)。
403：Forbidden: 表示用户得到授权了，但是访问被禁止了, 也可以理解为不具有访问资源的权限。
404：Not Found: 所请求的资源不存在，或不可用。
405：Method Not Allowed: 用户已经通过了身份验证, 但是所用的HTTP方法不在它的权限之内。
406：Not Acceptable: 用户的请求的格式不可得(比如用户请求的是JSON格式，但是只有XML格式)。
410：Gone - [GET]: 用户请求的资源被转移或被删除。且不会再得到的。
415: Unsupported Media Type: 客户端要求的返回格式不支持，比如API只能返回JSON格式，但客户端要求返回XML格式。
422：Unprocessable Entity: 客户端上传的附件无法处理，导致请求失败。
429：Too Many Requests: 客户端的请求次数超过限额。

### 5xx 状态码

5xx 状态码表示服务器端错误。

500：INTERNAL SERVER ERROR; 服务器发生错误。
502：网关错误。
503:   Service Unavailable 服务器端当前无法处理请求。
504：网关超时。



## 统一返回数据格式

RESTful规范中的请求应该返回统一的数据格式。对于返回的数据，一般会包含如下字段:

1、**code**: http响应的状态码。
2、**status**: 包含文本, 比如：'success'(成功), 'fail'(失败), 'error'(异常) HTTP状态响应码在500-599之间为 'fail'; 在400-499之间为 'error', 其他一般都为 'success'。 对于响应状态码为 1xx, 2xx, 3xx 这样的可以根据实际情况可要可不要。当status的值为 'fail' 或 'error'时，需要添加 message 字段，用于显示错误信息。
3、**data**: 当请求成功的时候, 返回的数据信息。 但是当状态值为 'fail' 或 'error' 时，data仅仅包含错误原因或异常信息等。

返回成功的响应JSON格式一般为如下:

```json
{
    "code": 200,
    "status": "success",
    "data": [{
        "userName": "tugenhua",
        "age": 31
    }]
}
```

返回失败的响应json格式为如下:

```json
{
    "code": 401,
    "status": "error",
    "message": '用户没有权限',
    "data": null
}
```



## 面向资源设计接口还是面向服务（功能）设计接口

**统一接口**是 REST 的一条核心原则，REST 希望开发者面向资源编程，希望软件系统设计的重点放在抽象系统该有哪些资源上，而不是抽象系统该有哪些行为（服务）上。这条原则你可以类比计算机中对文件管理的操作来理解，管理文件可能会进行创建、修改、删除、移动等操作，这些操作数量是可数的，而且对所有文件都是固定的、统一的。如果面向资源来设计系统，同样会具有类似的操作特征，由于 REST 并没有设计新的协议，所以这些操作都借用了 HTTP 协议中固有的操作命令来完成。

统一接口也是 REST 最容易陷入争论的地方，基于网络的软件系统，到底是面向资源更好，还是面向服务更合适，这事情哪怕是很长时间里都不会有个定论，也许永远都没有。但是，已经有一个基本清晰的结论是：面向资源编程的抽象程度通常更高。抽象程度高意味着坏处是往往距离人类的思维方式更远，而好处是往往通用程度会更好。用这样的语言去诠释 REST，大概本身就挺抽象的，笔者还是举个例子来说明：譬如，几乎每个系统都有的登录和注销功能，如果你理解成登录对应于 login()服务，注销对应于 logout()服务这样两个独立服务，这是“符合人类思维”的；如果你理解成登录是 PUT Session，注销是 DELETE Session，这样你只需要设计一种“Session 资源”即可满足需求，甚至以后对 Session 的其他需求，如查询登陆用户的信息，就是 GET Session 而已，其他操作如修改用户信息等都可以被这同一套设计囊括在内，这便是“抽象程度更高”带来的好处。

想要在架构设计中合理恰当地利用统一接口，Fielding 建议系统应能做到每次请求中都包含资源的 ID，所有操作均通过资源 ID 来进行；建议每个资源都应该是自描述的消息；建议通过超文本来驱动应用状态的转移。

我认为应该依据场景来选择面向资源编程还是面向服务编程。比如像是登录、注销这种具有明确功能的场景，使用使用面向服务编程会更合适一些，具体点儿就是登录采用login、注销采用logout而不是PUT Session和DELETE Session。如果查询、修改用户信息就使用面向资源的编程方式，具体就是采用GET user_info、PUT user_info、PATCH user_info而不是采用getUserInfo、updateUserInfo、modifyUserInfo。

依据场景选择REST接口设计方式的好处是接口容易使用、理解，但是缺点就是接口设计风格不统一。

你的意见是什么呢？欢迎讨论。



## 参考资料

1、https://www.cnblogs.com/pyxiaomangshe/p/9339893.html

2、http://icyfenix.cn/architect-perspective/general-architecture/api-style/rest.html

3、https://zhuanlan.zhihu.com/p/34289466

4、https://www.cnblogs.com/tugenhua0707/p/12153857.html

5、[Zalando RESTful API and Event Guidelines](https://opensource.zalando.com/restful-api-guidelines/#_zalando_restful_api_and_event_guidelines)

6、https://blog.restcase.com/7-rules-for-rest-api-uri-design/

7、[RESTful Web API 设计](https://docs.microsoft.com/zh-cn/azure/architecture/best-practices/api-design)

8、[Api Design Guide](https://google-cloud.gitbook.io/api-design-guide)