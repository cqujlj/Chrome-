# Chrome-
### 一、一些概念
#### 1、CUP:串行的处理任务(单核/多核)
#### 2、GPU:并行计算能力强（单片GPU上有很多核心）
#### 3、进程：
  进程是一个具有一定独立功能的程序在一个数据集上的一次动态执行的过程，
  进程是拥有资源和独立运行的最小单位，是操作系统分配资源、程序执行的最小单位
  每个进程有各自独立的一块内存，使得各个进程之间内存地址相互隔离
  是应用程序运行的载体
#### 4、线程：
  线程是程序执行中一个单一的顺序控制流程，程序执行的最小单位，是处理器调度和分派的基本单位
  一个进程可以有一个或多个线程，各个线程之间共享程序的内存空间(也就是所在进程的内存空间)
### 二、浏览器架构
#### 1、单进程架构：启动一个进程（进程里可能有多个线程）
#### 2、多进程架构：进程之间通过IPC(Inter Process Communication)通信
  进程和进程之间互相独立、互不影响，如果其中一个工作进程（worker process）挂掉了其他进程不会受到影响，而且挂掉的进程还可以重启
#### 3、Brower Process（浏览器进程）
  负责浏览器的‘Chrome’部分，包括导航栏、书签、前进后退按钮以及数据请求文件读取等操作
#### 4、Render Process（渲染进程）
  负责tab内和网页展示相关的工作
#### 5、plugin Process（插件进程）、扩展进程（Extension Process）和工具进程（utility process
  控制网页使用的所有插件、扩展工具
#### 5、GPU Process（GPU进程）
  负责独立于其他进程的GPU任务；
  之所以被独立为一个进程，因为它要处理来自不同tab的渲染请求并把它在同一界面绘制出来
#### 6、多进程的好处：
###### （1）容错性
  Chrome为每个tab单独分配一个进程；当其中一个tab挂掉，其他tab不会收到影响
###### （2）安全性
###### （3）沙盒性
  沙盒：按照安全策略限制程序行为的执行环境
  操作系统提供方法限制每个进程拥有的能力和不具备的能力；比如限制对系统文件的读取
#### 7、多进程的缺点：内存消耗
  解决方法：限制被启动的进程数；当进程树达到一定界限后，Chrome会将访问同一网站的tab合并到一个进程里面运行
#### 8、Chrome服务化
  将以上解决方案用于Brower Process
  将和浏览器本身相关的部分拆分为一个个不同的服务，服务化之后，这些功能既可以放在不同的进程中运行，也可以放在同一个进程中
    · 硬件性能好：在不同进程里面运行    · 硬件性能不好： 在同一个进程中运行
#### 9、网站隔离：加强网络安全，清楚网页之间共享进程
  每一个跨站点的iframe分配一个独立的renderer process
### 三、导航过程
#### 从Brower process开始
  浏览器进程有很多负责不同工作的线程：
    绘制浏览器顶部按钮、导航栏输入框等组件的UI线程 UI thread
    管理网络请求的网络线程 network thread
    控制文件读写的存储线程 storage thread等.........
#### 1、处理输入 UI thread
  当导航栏输入，输入可能是请求的域名/搜索内容的关键字
  UI thread进行一系列的解析来判定是“将用户输入发送给搜索引擎”还是“直接请求你输入的站点资源”
#### 2、开始导航 network thread（进行DNS寻址、简历TSL连接等）
  当用户按下Enter，UI thread会叫network thread初始化一个网络请求来获取站点的内容
  若网络线程收到服务器的 HTTP 301 重定向响应，它会告知UI线程进行重定向然后再次发起网络请求
#### 3、读取响应
  network thread 收到HTTP响应的主题流（payload stream）之后，先检查stream的前几个字节以确定响应主题的媒体类型
  一般通过HTTP头部的content-type来确定 或者MIME类型嗅探
    · 若响应主体是HTML文件---> 浏览器获取响应数据给renderer process
    · 若响应主体是压缩文件---> 交给下载管理器处理
#### 4、寻找一个renderer process 来渲染界面
  network thread对数据进行safeBrowsing检查 CORB跨域敏感数据检查后，
  能够确定浏览器应该导航到该请求的站点，它就会告诉UI线程所有的数据都已经被准备好
  UI thread 在收到网络线程的确认后会为这个网站寻找一个渲染进程（renderer process）来渲染界面
##### 优化策略：为了缩短navagitor时间，进行优化
  在UI thread解析URL发给network thread的过程中，UI thread同事为该网络请求启动一个renderer process
  · 若数据请求顺利，则能立即渲染
  · 若发生重定向，则摒弃该renderer process，重新启动一个新的renderer process
#### 5、提交导航 commit navigation
  当数据和渲染进程都OK：
    · 则Browser process通过IPC(Inter Process Communication)告诉renderer process去commit
    · 将刚收到的响应数据流传递给对应的renderer process让它继续接受HTML数据
    · Browser process收到renderer process的回复说导航已经commit，则navigation过程结束；文档加载阶段正式开始
  当导航栏被更新，安全指示符security indicator、站点设置UI会展示新页面相关的站点信息
  当前站点的历史纪录也会被更新；当关闭当前tab，历史记录会被存储到磁盘上
### 四、导航到不同的站点
#### 1、在导航栏输入不同的URL
  会重新执行一次上述过程
  在这之前，需要当前渲染页面做一个收尾工作：浏览器进程要在重新导航的时候和当前渲染进程确认是否需要beforeunload事件
###### beforeunload事件：可以在用户重新导航或者关闭当前tab时给用户一个二次确认框“确定要离开当前页面吗？”
  note：不要随便使用，容易增加重导航的时延
###### 当前页面发生的一切（包括页面的JavaScript执行）是不受browser process 控制而是受renderer process 控制，所以它也不知道里面的具体情况，需要确认
#### 2、在页面中点击了一个URL
  renderer process先检查是否注册beforeunload
   若有：执行beforeunload，执行完之后进行一次navigator(此次的导航请求是由renderer process给browser process发起的)
   若导航到不同站点：浏览器进程告诉新的 renderer process 去渲染新的页面并且告诉当前的 renderer process 进行收尾工作
