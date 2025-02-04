# 开发前必读

使用WorkerMan开发应用，你需要了解以下内容：


## 一、WorkerMan开发与普通PHP开发的不同之处

除了与HTTP协议相关的变量函数无法直接使用外，WorkerMan开发与普通PHP开发并没有很大不同。

### 1、应用层协议不同
* 普通PHP开发一般是基于HTTP应用层协议，WebServer已经帮开发者完成了协议的解析
* WorkerMan支持各种协议，目前内置了HTTP、WebSocket等协议。WorkerMan推荐开发者使用更简单的自定义协议通讯

 *  HTTP协议开发请参考[Http服务部分](../http/request.md)

### 2、请求周期差异
* PHP在Web应用中一次请求过后会释放所有的变量与资源
* WorkerMan开发的应用程序在第一次载入解析后便常驻内存，使得类的定义、全局对象、类的静态成员不会释放，便于后续重复利用

### 3、注意避免类和常量的重复定义
* 由于WorkerMan会缓存编译后的PHP文件，所以要避免多次require/include相同的类或者常量的定义文件。建议使用require_once/include_once加载文件。

### 4、注意单例模式的连接资源的释放
* 由于WorkerMan不会在每次请求后释放全局对象及类的静态成员，在数据库等单例模式中，往往会将数据库实例（内部包含了一个数据库socket连接）保存在数据库静态成员中，使得WorkerMan在进程生命周期内都复用这个数据库socket连接。需要注意的是当数据库服务器发现某个连接在一定时间内没有活动后可能会主动关闭socket连接，这时再次使用这个数据库实例时会报错，（错误信息类似mysql gone away）。WorkerMan提供了[数据库类](../components/workerman-mysql.md)，有断开重连的功能，开发者可以直接使用。

### 5、注意不要使用exit、die出语句
* WorkerMan运行在PHP命令行模式下，当调用exit、die退出语句时，会导致当前进程退出。虽然子进程退出后会立刻重新创建一个的相同的子进程继续服务，但是还是可能对业务产生影响。

### 6、改完代码需要重启服务才能生效
由于WorkerMan是常驻内存的，php类即函数的定义加载一次后便常驻内存，不会再次读取磁盘加载，所以每次修改完业务代码需要重启才能生效。


## 二、需要了解的基本概念

### 1、TCP传输层协议
TCP是一种面向连接的、可靠的、基于IP的传输层协议。TCP传输层协议一个重要特点是TCP是基于数据流的，客户端的请求会源源不断的发送给服务端，服务端收到的数据可能不是一个完整的请求，也有可能是多个请求连在一起。这就需要我们在这源源不断的数据流中区分每个请求的边界。而应用层协议主要是为请求边界定义一套规则，避免请求数据混乱。

### 2、应用层协议

应用层协议(application layer protocol)定义了运行在不同端系统上（客户端、服务端）的应用程序进程如何相互传递报文，例如HTTP、WebSocket都属于应用层协议。例如一个简单的应用层次协议可以如下```{"module":"user","action":"getInfo","uid":456}\n"```。此协议是以```"\n"```（注意这里```"\n"```代表的是回车）标记请求结束，消息体是字符串。

### 3、短连接

短连接是指通讯双方有数据交互时，就建立一个连接，数据发送完成后，则断开此连接，即每次连接只完成一项业务的发送。像WEB网站的HTTP服务一般都用短连接。

*短连接应用程序开发可以参考基本开发流程一章*


### 4、长连接

长连接，指在一个连接上可以连续发送多个数据包。

注意：长连接应用必须加[心跳](../faq/heartbeat.md)，否则连接可能由于长时间不活跃而被路由节点防火墙断开。

长连接多用于操作频繁，点对点的通讯的情况。每个TCP连接都需要三步握手，这需要时间，如果每个操作都是先连接，再操作的话那么处理速度会降低很多。所以长连接在每个操作完后都不断开，下次处理时直接发送数据包就OK了，不用建立TCP连接。例如：数据库的连接用长连接，如果用短连接频繁的通信会造成socket错误，而且频繁的socket 创建也是对资源的浪费。

*当需要主动向客户端推送数据时，例如聊天类、即时游戏类、手机推送等应用需要长连接。*
*长连接应用程序开发可以参考Gateway/Worker开发流程*


### 5、平滑重启

一般的重启的过程是把所有进程全部停止后，再开始创建全新的服务进程。在这个过程中会有一个短暂的时间内是没有进程对外提供服务的，这就会导致服务暂时不可用，这在高并发时势必会导致请求失败。

而平滑重启则不是一次性的停止所有进程，而是一个进程一个进程的停止，每停止一个进程后马上重新创建一个新的进程顶替，直到所有旧的进程都被替换为止。

平滑重启WorkerMan可以使用 ```php your_file.php reload```命令，能够做到在不影响服务质量的情况下更新应用程序。

**注意：只有在on{...}回调中载入的文件平滑重启后才会自动更新，启动脚本中直接载入的文件或者写死的代码运行reload不会自动更新。**


## 三、区分主进程和子进程
有必要注意下代码是运行在主进程还是子进程，一般来说在```Worker::runAll();```调用前运行的代码都是在主进程运行的，onXXX回调运行的代码都属于子进程。注意写在```Worker::runAll();```后面的代码永远不会被执行。

例如下面的代码
```php
use Workerman\Worker;
use Workerman\Connection\TcpConnection;
require_once __DIR__ . '/vendor/autoload.php';

// 运行在主进程
$tcp_worker = new Worker("tcp://0.0.0.0:2347");
// 赋值过程运行在主进程
$tcp_worker->onMessage = function(TcpConnection $connection, $data)
{
    // 这部分运行在子进程
    $connection->send('hello ' . $data);
};

Worker::runAll();
```

**注意：** 不要在主进程中初始化数据库、memcache、redis等连接资源，因为主进程初始化的连接可能会被子进程自动继承（尤其是使用单例的时候），所有进程都持有同一个连接，服务端通过这个连接返回的数据在多个进程上都可读，会导致数据错乱。同样的，如果任何一个进程关闭连接(例如daemon模式运行时主进程会退出导致连接关闭)，都导致所有子进程的连接都被一起关闭，并发生不可预知的错误，例如mysql gone away 错误。

推荐在onWorkerStart里面初始化连接资源。
