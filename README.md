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
    解决方法：限制被启动的进程数；当进程数达到一定界限后，Chrome会将访问同一网站的tab合并到一个进程里面运行
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
#### 2、开始导航 network thread（进行DNS寻址、建立TSL连接等）
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
    在UI thread解析URL发给network thread的过程中，UI thread 同时为该网络请求启动一个renderer process
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
### 四、Service Worker 
    Service Worker 是浏览器在后台独立于网页运行的脚本，是独立于页面的一个运行环境，它在页面关闭后仍可以运行
    包括推送通知和后台同步等功能；不需要网页或用户交互的功能，
##### Service Worker 通过响应 postMessage 接口发送的消息来与其控制的页面通信，页面可在必要时对 DOM 执行操作
      service worker在注册的时候，它的作用范围（scope）会被记录下来
      在导航开始的时候，网络线程会根据请求的域名在已经注册的service worker作用范围里面寻找有没有对应的service worker。
        · 如果有命中该URL的service worker，UI线程就会为这个service worker启动一个renderer process 来执行它的代码。
        · Service worker既可能使用之前缓存的数据也可能发起新的网络请求
##### 导航预加载 navigation preload
    通过在service worker启动时并行加载对应的资源得方式来加快整个导航过程的效率
    预加载资源的请求头会有一些特殊的标志来让服务器决定是发送全新的内容给客户端还是仅仅发送更新了的数据给客户端
### 五、渲染进程
#### 1、构造dom
    渲染进程开始接收HTML数据，同时主线程也会开始解析接收到的文本数据（text string）并把它转化为一个DOM（Document Object Model）对象
    DOM对象：是浏览器对当前页面的内部表示，也是Web开发人员通过JavaScript与网页进行交互的数据结构以及API
#### 2、加载CSS和JavaScript等资源
    浏览器会同时运行“预加载扫描”（preload scanner）程序
    预加载扫描程序会在HTML解析器生成的token里面找到对应要获取的资源，并把这些要获取的资源告诉浏览器进程里面的网络线程
###### JavaScript会阻塞HTML的解析过程：
      因为script标签中的JavaScript可能会改变文档流（document）的形状，从而使整个DOM树的结构发生根本性的改变
      优化：在DOM和CSS渲染之后加载js文件，例如在尾部加载js文件，或者使用该元素属性defer和async，进行js问价异步加载
###### 由于CSS决定了DOM元素的样式、布局，所以浏览器遇到CSS文件时会等待CSS文件加载并解析完后才继续渲染页面
    尽可能早的提前引入css文件，例如在头部引入css文件；
    尽可能早的加载css文件中的引入的资源，例如自定义字体文件，可以使用预加载，在link标签中加入rel=“preload” as = “font”该元素属性，不会造成渲染阻塞
#### 3、计算样式
    主线程会解析页面的CSS从而确定每个DOM节点的计算样式（computed style）。
    计算样式是主线程根据CSS样式选择器（CSS selectors）计算出的每个DOM元素应该具备的具体样式
    note：每个浏览器都有自己的默认样式表
#### 4、布局
    主线程会遍历刚刚构建的DOM树，根据DOM节点的计算样式计算出一个布局树；即计算RenderObject树中的元素的尺寸，位置等
    布局树上每个节点会有它在页面上的x，y坐标以及盒子大小（bounding box sizes）的具体信息
    note：被设置为display:none的节点，不可见 不会出现在布局树上面； 被设置为visibility:hidden的节点，会出现在布局树上面
          如果一个伪元素（pseudo class）节点有诸如p::before{.....}，它会出现在布局上，而不存在于DOM树上
#### 5、绘制：绘制页面的像素信息
    主线程会遍历之前得到的布局树（layout tree）来生成一系列的绘画记录（paint records）。绘画记录是对绘画过程的注释   
    渲染流水线的每一步都要使用到前一步的结果来生成新的数据 ，因此渲染成本较高；
    可以将要被执行的JavaScript操作拆分为更小的块然后通过requestAnimationFrame这个API把他们放在每个动画帧中执行
### 六、绘制页面
#### 1、光栅化
###### 将文档结构，元素的样式，元素的几何信息以及它们的绘画顺序等信息转化为显示器的像素的过程
    光栅化视口内（viewport）的网页内容。如果用户进行了页面滚动，就移动光栅帧（rastered frame）并且光栅化更多的内容以补上页面缺失的部分
#### 2、合成 ---> 没有涉及到主线程 --> 构建流畅用户体验
        将页面分成若干层，然后分别对它们进行光栅化，最后在一个单独的线程 - 合成线程（compositor thread）里面合并成一个页面
        当页面滚动时，浏览器合成一个新的帧来展示滚动后的效果
##### 2.1 页面分层
        主线程遍历渲染树来创建一棵层次树（Layer Tree），来确定哪些元素需要放置在哪一层
        如果页面的某些部分应该被放置在一个单独的层上面（滑动菜单）可是却没有的话，通过使用will-change CSS属性来告诉浏览器对其分层
##### 2.2 在主线程之外光栅化和合成页面
        页面的层次树创建出来并且页面元素的绘制顺序确定后，主线程就会向合成线程（compositor thread）提交这些信息。
        合成线程就会光栅化页面的每一层，合成线程需要将它们切分为一块又一块的小图块然后将图块发送给一系列光栅线程（raster threads）。
        光栅线程会栅格化每个图块并且把它们存储在GPU的内存中（合成线程可以给不同的光栅线程赋予不同的优先级）
###### 绘画四边形：包含图块在内存的位置以及图层合成后图块在页面的位置之类的信息。
###### 合成帧：代表页面一个帧的内容的绘制四边形集合。（会被发送给GPU从而展示在屏幕上）
###### 完成以上的关栅化和合成等步骤之后，合成线程通过IPC向browser process commit一个渲染帧
        如果合成线程收到页面滚动的事件，合成线程会构建另外一个合成帧发送给GPU来更新页面
### 七、从浏览器角度看输入
##### 1、输入事件
        来自于用户的任何手势动作，如用户滚动页面，触碰屏幕以及移动鼠标等
        浏览器进程会将事件的类型(ckick mousedown...)以及坐标（x,y）发送给渲染进程; 渲染进程找到事件的目标对象（target）然后运行这个事件绑定的监听函数
##### 2、非快速滚动区域
        一个页面被合成的时候，合成线程会将页面那些注册了事件监听器的区域
            · 如果输入事件发生在在这些区域时，合成线程会将输入事件发送给主线程来处理
            · 如果输入事件不是发生在非快速滚动区域，合成线程就无须主线程的参与来合成一个新的帧
###### note：给body元素绑定了事件监听器后其实是将整个页面都标记为一个非快速滚动区域**
         解决方法：为事件监听器传递passive：true选项，告诉浏览器您仍要在主线程中侦听事件，合成线程也可以继续合成新的帧
###### 3、查找事件的目标对象（event target）
        当合成线程向主线程发送输入事件时，主线程要通过 ** 命中测试 ** 去找到事件的目标对象（target）。
        命中测试流程:遍历在渲染流水线中生成的绘画记录来找到输入事件出现的x, y坐标上面描绘的对象
##### 4、最小化
        输入事件的触发频率其实远远高于我们屏幕的刷新频率（60次/s）
        Chrome会合并连续事件（例如wheel，mousewheel，mousemove，pointermove，touchmove），将调度延迟到下一个requestAnimationFrame之前 --> 减少对主线程的过多调用
        相对不怎么频繁发生的事件都会被立即派送给主线程
##### 使用鼠标事件的getCoalescedEvents来获取被合成的事件的详细信息
        window.addEventListener('pointermove', event => {
            const events = event.getCoalescedEvents();
            for (let event of events) {
                const x = event.pageX;
                const y = event.pageY;
                // draw a line using x and y coordinates.
            }
        });
