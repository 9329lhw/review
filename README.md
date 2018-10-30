<h1 align="center">初识rabbitMQ</h1>

## 一,RabbitMQ概述
- RabbitMQ是一个由Erlang开发的AMQP（AdvancedMessage Queue ）的开源实现，支持多种客户端，如：Python、Ruby、.NET、Java、JMS、C、PHP、ActionScript、XMPP、STOMP等，支持AJAX。用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面表现不俗
- AMQP，即Advanced Message Queuing Protocol,一个提供统一消息服务的应用层标准高级消息队列协议,是应用层协议的一个开放标准,为面向消息的中间件设计。基于此协议的客户端与消息中间件可传递消息，并不受客户端/中间件不同产品，不同的开发语言等条件的限制。Erlang中的实现有 RabbitMQ等
- RabbitMQ的结构图如下：
   
![](https://i.imgur.com/0ZrhTsf.png)

- 几个概念说明：
   		
 	- Broker：简单来说就是消息队列服务器实体。
 	- Exchange：消息交换机，它指定消息按什么规则，路由到哪个队列。
	- Queue：消息队列载体，每个消息都会被投入到一个或多个队列。
	- Binding：绑定，它的作用就是把exchange和queue按照路由规则绑定起来。
	- Routing Key：路由关键字，exchange根据这个关键字进行消息投递。
	- vhost：虚拟主机，一个broker里可以开设多个vhost，用作不同用户的权限分离。
	- producer：消息生产者，就是投递消息的程序。
	- consumer：消息消费者，就是接受消息的程序。
	- channel：消息通道，在客户端的每个连接里，可建立多个channel，每个channel代表一个会话任务。
- 消息队列的使用过程大概如下：
	
	- 客户端连接到消息队列服务器，打开一个channel。
	- 客户端声明一个exchange，并设置相关属性。
	- 客户端声明一个queue，并设置相关属性。
	- 客户端使用routing key，在exchange和queue之间建立好绑定关系。
	- 客户端投递消息到exchange。
	
		- exchange接收到消息后，就根据消息的key和已经设置的binding，进行消息路由，将消息投递到一个或多个队列里。
		- exchange也有几个类型，完全根据key进行投递的叫做Direct交换机，例如，绑定时设置了routing key为”abc”，那么客户端提交的消息，只有设置了key为”abc”的才会投递到队列。对key进行模式匹配后进行投递的叫做Topic交换机，符号”#”匹配一个或多个词，符号”*”匹配正好一个词。例如”abc.#”匹配”abc.def.ghi”，”abc.*”只匹配”abc.def”。还有一种不需要key的，叫做Fanout交换机，它采取广播模式，一个消息进来时，投递到与该交换机绑定的所有队列。
		- RabbitMQ支持消息的持久化，也就是数据写在磁盘上，为了数据安全考虑，我想大多数用户都会选择持久化。消息队列持久化包括3个部分：
		
		　　	（1）exchange持久化，在声明时指定durable => 1

		　　	（2）queue持久化，在声明时指定durable => 1

		　　	（3）消息持久化，在投递时指定delivery_mode => 2（1是非持久化）
			如果exchange和queue都是持久化的，那么它们之间的binding也是持久化的。如果exchange和queue两者之间有一个持久化，一个非持久化，就不允许建立绑定。

## 二, rabbitmq使用场景
	
### 1.异步处理
- 场景：发送手机验证码，邮件
	- 传统古老处理方式如下图
	
	![](https://i.imgur.com/HIujlI2.png)

	这个流程，全部在主线程完成，注册-》入库-》发送邮件-》发送短信，由于都在主线程，所以要等待每一步完成才能继续执行。由于每一步的操作时间响应时间不固定，所以主线程的请求耗时可能会非常长，如果请求过多，会导致IIS站点巨慢，排队请求，甚至宕机，严重影响用户体验。
	
	- 现在大多数的处理方式如下图
	
	![](https://i.imgur.com/e8tO2eq.png)

	这个做法是主线程只做耗时非常短的入库操作，发送邮件和发送短信，会开启2个异步线程，扔进去并行执行，主线程不管，继续执行后续的操作，这种处理方式要远远好过第一种处理方式，极大的增强了请求响应速度，用户体验良好。缺点是，由于异步线程里的操作都是很耗时间的操作，一个请求要开启2个线程，而一台标准配置的ECS服务器支撑的并发线程数大概在800左右，假设一个线程在10秒做完，这个单个服务器最多能支持400个请求的并发，后面的就要排队。出现这种情况，就要考虑增加服务器做负载，尴尬的增加成本。
- 消息队列RabbitMq的处理方式

	![](https://i.imgur.com/dCz7odg.png)

	这个流程是，主线程依旧处理耗时低的入库操作，然后把需要处理的消息写进消息队列中，这个写入耗时可以忽略不计，非常快，然后，独立的发邮件子系统，和独立的发短信子系统，同时订阅消息队列，进行单独处理。处理好之后，向队列发送ACK确认，消息队列整条数据删除。这个流程也是现在各大公司都在用的方式，以SOA服务化各个系统，把耗时操作，单独交给独立的业务系统，通过消息队列作为中间件，达到应用解耦的目的，并且消耗的资源很低，单台服务器能承受更大的并发请求。

### 2.应用解耦

- 以电商的下订单为例子，假设中间的流程为下单=》减库存=》发货
- 第一种方式，通过连续操作表，在单一系统中，通过主线程，连续操作。呵呵哒，这种做法，相信很多人刚入门，甚至几年经验了，由于项目小，也在继续使用吧。用户量少，或者都是内部人使用，必然没问题，因为不会在意出的问题，这种做法，只要一个环节出问题，请求直接报错，导致用户懵逼，假设在执行到减库存操作报错了，整个流程没有用事务回滚的话，还会造成数据不一致。
- 第二种方式，把这三个业务，拆成三个独立系统，通过JSON方式相互调用请求。这个做法，其实已经很不错了，起码独立出来，各自做各自的事情，一定程度上减小了整个系统的耦合性。但是问题是，就算是通过API形式请求，发送请求的系统一般情况下会等待被请求方的响应，如果响应错了，整个程序还是会终止，前面的业务系统假如已经做了入库操作，就必须要混滚了。很麻烦。如果说不等待被请求方响应的话，如果出错，如果还要保证数据一致性，就要做更多的操作，去补全数据，比如，下单成功，减库存失败，发货成功，这样当减库存系统修复后，就要通过订单数据，去补库存表的对应数据。先对麻烦，难处理。
- 第三种方式，把消息队列作为中间件，当订单系统下完单后，把数据消息写入消息队列中，库存系统和发货系统同时订阅这个消息队列，思想上和纯API系统调用类似，但是，消息队列RabbitMq本身的强大功能，会帮我们做大量的出错善后处理，还是，假设下单成功，库存失败，发货成功，当我们修复库存的时候，不需要任何管数据的不一致性，因为库存队列未被处理的消息，会直接发送到库存系统，库存系统会进行处理。实现了应用的大幅度解耦。

### 3.流量削峰
- 这个主要用在团购，秒杀活动中

 	![](https://i.imgur.com/pvxhCJR.png)

	这个主要原理就是，控制队列长度，当请求来了，往队列里写入，超过队列的长度，就返回失败，给用户报一个可爱的错误页的等等。

### 4.日志处理
- 这个场景应该都很熟悉，一个系统有大量的业务需要各种日志来保证后续的分析工作，而且实时性要求不高，用队列处理再好不过了

### 5.消息通讯
   - 现在上线的各大社交通讯项目中，实际上是没有用消息队列做即时通讯的，但是它确实可以用来做，有兴趣的不妨去试试吧
    
   ![](https://i.imgur.com/vCUtoXB.png)

   - 这个是点对点通信，消费者A和B同时订阅消息队列，又同时是制造者，就能实现点对点通信群聊的做法，所有客户端同时订阅队列，又同时是发送，制造者。

   ![](https://i.imgur.com/iWa9DN1.png)
	
   上述大致的5种RabbitMq的应用场景，下面来介绍几个消息队列的区别
	- ActiveMq:这个应用于JAVA中间件较多
	- ZeroMq:这个是分发效率最高的队列，是其他队列的十倍以上，缺点是不能数据持久化。
	- kafka：这是一种高吞吐量的发布订阅消息系统，当每秒达到10W+的分发要求时，可以用这个尝试，新浪微博就是用这个做分发。

## 三, 常见的MQ
	
![常见的mq](https://i.imgur.com/jlp6qu4.png)

## 四, 五种队列模式
### 简单模式Hello World
  
  ![简单模式](https://i.imgur.com/rsbU4v4.png)
	
- 功能：一个生产者P发送消息到队列Q,一个消费者C接收
- 生产者实现思路：
	创建连接工厂ConnectionFactory，设置服务地址127.0.0.1，端口号5672，设置用户名、密码、virtual host，从连接工厂中获取连接connection，使用连接创建通道channel，使用通道channel创建队列queue，使用通道channel向队列中发送消息，关闭通道和连接。

- 消费者实现思路：
    创建连接工厂ConnectionFactory，设置服务地址127.0.0.1，端口号5672，设置用户名、密码、virtual host，从连接工厂中获取连接connection，使用连接创建通道channel，使用通道channel创建队列queue, 创建消费者并监听队列，从队列中读取消息。

### 工作队列模式Work Queue
  ![work](https://i.imgur.com/ivYJRIl.jpg)	

- 功能：一个生产者，多个消费者，每个消费者获取到的消息唯一，多个消费者只有一个队列
- 任务队列：避免立即做一个资源密集型任务，必须等待它完成，而是把这个任务安排到稍后再做。我们将任务封装为消息并将其发送给队列。后台运行的工作进程将弹出任务并最终执行作业。当有多个worker同时运行时，任务将在它们之间共享。
- 生产者实现思路：
	创建连接工厂ConnectionFactory，设置服务地址127.0.0.1，端口号5672，设置用户名、密码、virtual host，从连接工厂中获取连接connection，使用连接创建通道channel，使用通道channel创建队列queue，使用通道channel向队列中发送消息，2条消息之间间隔一定时间，关闭通道和连接。

- 消费者实现思路：
    创建连接工厂ConnectionFactory，设置服务地址127.0.0.1，端口号5672，设置用户名、密码、virtual host，从连接工厂中获取连接connection，使用连接创建通道channel，使用通道channel创建队列queue，创建消费者C1并监听队列，获取消息并暂停10ms，另外一个消费者C2暂停1000ms，由于消费者C1消费速度快，所以C1可以执行更多的任务。

### 发布/订阅模式 Publish/Subscribe
  ![](https://i.imgur.com/QmQEZ7q.jpg)	

- 功能：一个生产者发送的消息会被多个消费者获取。一个生产者、一个交换机、多个队列、多个消费者
- 生产者：可以将消息发送到队列或者是交换机。
- 消费者：只能从队列中获取消息。如果消息发送到没有队列绑定的交换机上，那么消息将丢失。
交换机不能存储消息，消息存储在队列中
- 生产者实现思路：
	创建连接工厂ConnectionFactory，设置服务地址127.0.0.1，端口号5672，设置用户名、密码、virtual host，从连接工厂中获取连接connection，使用连接创建通道channel，使用通道channel创建队列queue，使用通道channel创建交换机并指定交换机类型为fanout，使用通道向交换机发送消息，关闭通道和连接。
- 消费者实现思路：
    创建连接工厂ConnectionFactory，设置服务地址127.0.0.1，端口号5672，设置用户名、密码、virtual host，从连接工厂中获取连接connection，使用连接创建通道channel，使用通道channel创建队列queue，绑定队列到交换机，设置Qos=1，创建消费者并监听队列，使用手动方式返回完成。可以有多个队列绑定到交换机，多个消费者进行监听。

### 路由模式Routing
  ![routing](https://i.imgur.com/Pyqjd8I.jpg)	

- 说明：生产者发送消息到交换机并且要指定路由key，消费者将队列绑定到交换机时需要指定路由key
- 生产者实现思路：
	创建连接工厂ConnectionFactory，设置服务地址127.0.0.1，端口号5672，设置用户名、密码、virtual host，从连接工厂中获取连接connection，使用连接创建通道channel，使用通道channel创建队列queue，使用通道channel创建交换机并指定交换机类型为direct，使用通道向交换机发送消息并指定key=b，关闭通道和连接。
- 消费者实现思路：
    创建连接工厂ConnectionFactory，设置服务地址127.0.0.1，端口号5672，设置用户名、密码、virtual host，从连接工厂中获取连接connection，使用连接创建通道channel，使用通道channel创建队列queue，绑定队列到交换机，设置Qos=1，创建消费者并监听队列，使用手动方式返回完成。可以有多个队列绑定到交换机,但只要绑定key=b的队列key接收到消息，多个消费者进行监听。

### 通配符模式Topic

  ![topic](https://i.imgur.com/f56FHSC.jpg)
- 说明：生产者P发送消息到交换机X，type=topic，交换机根据绑定队列的routing key的值进行通配符匹配；
	- 符号#：匹配一个或者多个词 lazy.# 可以匹配 lazy.irs或者lazy.irs.cor
	- 符号*：只能匹配一个词 lazy.* 可以匹配 lazy.irs或者lazy.cor
- 生产者实现思路：
	创建连接工厂ConnectionFactory，设置服务地址127.0.0.1，端口号5672，设置用户名、密码、virtual host，从连接工厂中获取连接connection，使用连接创建通道channel，使用通道channel创建队列queue，使用通道channel创建交换机并指定交换机类型为topic，使用通道向交换机发送消息并指定key=key.1，关闭通道和连接。
- 消费者实现思路：
    创建连接工厂ConnectionFactory，设置服务地址127.0.0.1，端口号5672，设置用户名、密码、virtual host，从连接工厂中获取连接connection，使用连接创建通道channel，使用通道channel创建队列queue，绑定队列到交换机，设置Qos=1，创建消费者并监听队列，使用手动方式返回完成。可以有多个队列绑定到交换机,凡是绑定规则符合通配符规则的队列均可以接收到消息，比如key.*,key.#，多个消费者进行监听。











		https://www.jianshu.com/p/80eefec808e5
		https://blog.csdn.net/bjo2008cn/article/details/55504896
		http://blog.51cto.com/ylcodes01/1971409
		https://www.cnblogs.com/saltlight-wangchao/p/6214334.html
		https://blog.csdn.net/Dome_/article/details/80028087



	
