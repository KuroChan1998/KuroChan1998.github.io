---
layout:     post
title:      rabbitMQ的使用入门
subtitle:   看b站视频学习rabbitMQ，有点蒙，还是先整理了下可能质量不高，借鉴的东西比较多~ 
date:       2019-10-03
author:     Kuro
header-img: img/tag-bg.jpg
catalog: true
tags:
    - java
    - rabbitMQ

---

# rabbitMQ

## 使用场景

定义rabbitMQ工具类

```java
public class ConnectionUtils {
    public static Connection getConnection() throws IOException, TimeoutException {
        //定义连接工厂
        ConnectionFactory factory=new ConnectionFactory();
        //设置服务地址
        factory.setHost("127.0.0.1");
        //amqp 5672
        factory.setPort(5672);
        //vhost
        factory.setVirtualHost("/mysql8.0.13");
        //用户名
        factory.setUsername("mgszjz");
        //密码
        factory.setPassword("123");

        return factory.newConnection();
    }
}
```

### HelloWorld（简单模式）

![img](https://www.rabbitmq.com/img/tutorials/python-one-overall.png)

使用等同于只有一个消费者的Work Queue

### Work Queue（工作队列）

![img](https://www.rabbitmq.com/img/tutorials/python-two.png)

- 生产者

  ```java
  public class Sender {
      public static final String QUEUE_NAME="test_work_queue";
  
      public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
          Connection connection=ConnectionUtils.getConnection();
          //从链接中获取 通道
          Channel channel=connection.createChannel();
          //声明一个队列
          channel.queueDeclare(QUEUE_NAME,false,false,false,null);
  
          for (int i = 0; i < 50; i++) {
              String msg="hello"+i;
              channel.basicPublish("",QUEUE_NAME,null,msg.getBytes());
              Thread.sleep(i*20);
          }
  
          channel.close();
          connection.close();
      }
  }
  ```

- 消费者

  - 1

    ```java
    public class Receiver1 {
        public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
            Connection connection=ConnectionUtils.getConnection();
    
            Channel channel=connection.createChannel();
    
            //新版本定义消费者
            //声明一个队列
            channel.queueDeclare(Sender.QUEUE_NAME,false,false,false,null);
            Consumer consumer=new DefaultConsumer(channel){
                //一旦有消息就会触发方法
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                    String msg=new String(body,"utf-8");
                    System.out.println(msg);
                    try {
                        Thread.sleep(2000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        System.out.println("[1] done");
                    }
                }
            };
            //监听队列
            channel.basicConsume(Sender.QUEUE_NAME,true,consumer);
    
    
        }
    }
    ```

  - 2

    ```java
    public class Receiver2 {
        public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
            Connection connection=ConnectionUtils.getConnection();
    
            Channel channel=connection.createChannel();
    
            //新版本定义消费者
            //声明一个队列
            channel.queueDeclare(Sender.QUEUE_NAME,false,false,false,null);
            Consumer consumer=new DefaultConsumer(channel){
                //一旦有消息就会触发方法
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                    String msg=new String(body,"utf-8");
                    System.out.println(msg);
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        System.out.println("[2] done");
                    }
                }
            };
            //监听队列
            channel.basicConsume(Sender.QUEUE_NAME,true,consumer);
    
        }
    }
    ```

  消费者1和消费者2获取到的消息的数量是相同的，一个是奇数一个是偶数。这也叫**轮询分发**

#### 使用`channel.basicQos(1)`

每个消费者发送确认之前，消息队列不发送下一个消息到消费者，消费者必须**关闭自动应答ack**，改成手动。

- 生产者

  ```java
  public class Sender {
      public static final String QUEUE_NAME="test_work_queue";
  
      public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
          Connection connection=ConnectionUtils.getConnection();
          //从链接中获取 通道
          Channel channel=connection.createChannel();
          //声明一个队列
          channel.queueDeclare(QUEUE_NAME,false,false,false,null);
  
          //每个消费者发送确认之前，消息队列不发送下一个消息到消费者，消费者必须关闭自动应答ack，改成手动
          channel.basicQos(1);
  
          for (int i = 0; i < 50; i++) {
              String msg="hello"+i;
              channel.basicPublish("",QUEUE_NAME,null,msg.getBytes());
              Thread.sleep(i*20);
          }
  
          channel.close();
          connection.close();
      }
  }
  ```

- 消费者

  - 1

    ```java
    public class Receiver1 {
        public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
            Connection connection=ConnectionUtils.getConnection();
    
            final Channel channel=connection.createChannel();
    
            //每个消费者发送确认之前，消息队列不发送下一个消息到消费者，消费者必须关闭自动应答ack，改成手动
            channel.basicQos(1);
    
            //新版本定义消费者
            //声明一个队列
            channel.queueDeclare(Sender.QUEUE_NAME,false,false,false,null);
            Consumer consumer=new DefaultConsumer(channel){
                //一旦有消息就会触发方法
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                    String msg=new String(body,"utf-8");
                    System.out.println(msg);
                    try {
                        Thread.sleep(2000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        System.out.println("[1] done");
                        //关闭自动应答
                        channel.basicAck(envelope.getDeliveryTag(),false);
                    }
                }
            };
    
            //监听队列
            boolean autoAck=false;
            channel.basicConsume(Sender.QUEUE_NAME,autoAck,consumer);
        }
    }
    
    ```

  - 2

    类比1

  当使用channel.basicQos(1)，执行会发现，休眠时间短的消费者执行的任务多。这叫**公平分发**。

### Publish/Subscribe（发布订阅）

![img](https://www.rabbitmq.com/img/tutorials/bindings.png)

　在发布订阅模式中，消息需要发送到MQ的交换机exchange上，exchange根据配置的路由方式发到相应的Queue上，Queue又将消息发送给consumer，消息从queue到consumer， 消息队列的使用过程大概如下：

1. 客户端连接到消息队列服务器，打开一个channel。

2. 客户端声明一个exchange，并设置相关属性。

3. 客户端声明一个queue，并设置相关属性。

4. 客户端在exchange和queue之间建立好绑定关系。

5. 客户端投递消息到exchange。

- 生产者

  ```java
  public class Sender {
      public static final String EXCHANGE_NAME="test_exchange_fanout";
  
      public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
          Connection connection=ConnectionUtils.getConnection();
          //从链接中获取 通道
          Channel channel=connection.createChannel();
  
          //声明交换机
          channel.exchangeDeclare(EXCHANGE_NAME,"fanout");//分发
  
  
          String msg="hello exchange";
          channel.basicPublish(EXCHANGE_NAME,"",null,msg.getBytes());
  
  
          channel.close();
          connection.close();
      }
  }
  
  ```

- 消费者

  - 1

    ```java
    public class Receiver1 {
        public static final String QUEUE_NAME="test_queue_fanout_1";
    
        public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
            Connection connection=ConnectionUtils.getConnection();
    
            final Channel channel=connection.createChannel();
    
            //新版本定义消费者
            //声明一个队列
            boolean durable=false;
            channel.queueDeclare(QUEUE_NAME,durable,false,false,null);
    
            //队列绑定交换机
            channel.queueBind(QUEUE_NAME,Sender.EXCHANGE_NAME,"");
    
            //每个消费者发送确认之前，消息队列不发送下一个消息到消费者，消费者必须关闭自动应答ack，改成手动
            channel.basicQos(1);
    
            Consumer consumer=new DefaultConsumer(channel){
                //一旦有消息就会触发方法
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                    String msg=new String(body,"utf-8");
                    System.out.println(msg);
                    try {
                        Thread.sleep(2000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        System.out.println("[1] done");
                        //关闭自动应答
                        channel.basicAck(envelope.getDeliveryTag(),false);
                    }
                }
            };
    
            //监听队列
            boolean autoAck=false;
            channel.basicConsume(QUEUE_NAME,autoAck,consumer);
        }
    }
    ```

  - 2

    类比1

### Routing（路由）

`channel.exchangeDeclare(EXCHANGE_NAME,"fanout");`

fanout表示不处理路由键，只要有订阅的消费者都会收到消息；故有direct——路由

![img](https://www.rabbitmq.com/img/tutorials/python-four.png)

根据消费者拥有的key来发送对应的消息

- 生产者

  ```java
  public class Sender {
      public static final String EXCHANGE_NAME="test_exchange_direct";
  
      public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
          Connection connection=ConnectionUtils.getConnection();
          //从链接中获取 通道
          Channel channel = connection.createChannel();
  
          //声明交换机
          channel.exchangeDeclare(EXCHANGE_NAME,"direct");//路由
  
  
          String msg="hello exchange direct";
          String routingKey="info";
          channel.basicPublish(EXCHANGE_NAME,routingKey,null,msg.getBytes());
  
  
          channel.close();
          connection.close();
      }
  }
  ```

- 消费者

  - 1，定义1个

    ```java
    public class Receiver1 {
        public static final String QUEUE_NAME="test_queue_direct_1";
    
        public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
            Connection connection=ConnectionUtils.getConnection();
    
            final Channel channel=connection.createChannel();
    
            //新版本定义消费者
            //声明一个队列
            boolean durable=false;
            channel.queueDeclare(QUEUE_NAME,durable,false,false,null);
    
            //队列绑定交换机，绑定一个key
            String routingKey="error";
            channel.queueBind(QUEUE_NAME,Sender.EXCHANGE_NAME,routingKey);
    
            //每个消费者发送确认之前，消息队列不发送下一个消息到消费者，消费者必须关闭自动应答ack，改成手动
            channel.basicQos(1);
    
            Consumer consumer=new DefaultConsumer(channel){
                //一旦有消息就会触发方法
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                    String msg=new String(body,"utf-8");
                    System.out.println(msg);
                    try {
                        Thread.sleep(2000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        System.out.println("[1] done");
                        //关闭自动应答
                        channel.basicAck(envelope.getDeliveryTag(),false);
                    }
                }
            };
    
            //监听队列
            boolean autoAck=false;
            channel.basicConsume(QUEUE_NAME,autoAck,consumer);
        }
    }
    ```

  - 2，定义3个key

    ```java
            ......
    		//队列绑定交换机,绑定3个key
            String routingKey1="error";
            String routingKey2="info";
            String routingKey3="warning";
            channel.queueBind(QUEUE_NAME,Sender.EXCHANGE_NAME,routingKey1);
            channel.queueBind(QUEUE_NAME,Sender.EXCHANGE_NAME,routingKey2);
            channel.queueBind(QUEUE_NAME,Sender.EXCHANGE_NAME,routingKey3);
    		......
    ```

### Topics（主题通配符）

![img](https://www.rabbitmq.com/img/tutorials/python-five.png)

- “#”：表示匹配一个或多个词；（lazy.a.b.c）
- “*”：表示匹配一个词；（a.orange.b）

- 生产者

  ```java
  public class Sender {
      public static final String EXCHANGE_NAME="test_exchange_topic";
  
      public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
          Connection connection=ConnectionUtils.getConnection();
          //从链接中获取 通道
          Channel channel = connection.createChannel();
  
          //声明交换机
          channel.exchangeDeclare(EXCHANGE_NAME,"topic");//主题通配符
  
  
          String msg="hello exchange topic";
          String routingKey="goods.add";
          channel.basicPublish(EXCHANGE_NAME,routingKey,null,msg.getBytes());
  
  
          channel.close();
          connection.close();
      }
  }
  ```

- 消费者

  - 1，绑定"goods.update"

    ```java
    public class Receiver1 {
        public static final String QUEUE_NAME="test_queue_topic_1";
    
        public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
            Connection connection=ConnectionUtils.getConnection();
    
            final Channel channel=connection.createChannel();
    
            //新版本定义消费者
            //声明一个队列
            boolean durable=false;
            channel.queueDeclare(QUEUE_NAME,durable,false,false,null);
    
            //队列绑定交换机，绑定一个key,若要更改routingKey需要先将之前绑定的键unbind
            String routingKey="goods.update";
            channel.queueBind(QUEUE_NAME,Sender.EXCHANGE_NAME,routingKey);
    
            //每个消费者发送确认之前，消息队列不发送下一个消息到消费者，消费者必须关闭自动应答ack，改成手动
            channel.basicQos(1);
    
            Consumer consumer=new DefaultConsumer(channel){
                //一旦有消息就会触发方法
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                    String msg=new String(body,"utf-8");
                    System.out.println(msg);
                    try {
                        Thread.sleep(2000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        System.out.println("[1] done");
                        //关闭自动应答
                        channel.basicAck(envelope.getDeliveryTag(),false);
                    }
                }
            };
    
            //监听队列
            boolean autoAck=false;
            channel.basicConsume(QUEUE_NAME,autoAck,consumer);
        }
    }
    ```

  - 2，绑定"goods.#"

    ```java
            ......
    	    //队列绑定交换机,绑定goods下的所有键
            String routingKey="goods.#";
            channel.queueBind(QUEUE_NAME,Sender.EXCHANGE_NAME,routingKey);
    		......
    ```

**注意**：如果要更改一个消费者和交换机绑定的key，需要先将原来的key解绑！！！

### RPC（远程调用）

如果我们需要在远程计算机上运行一个函数并等待结果，这种模式通常称为远程过程调用或RPC；

![img](https://www.rabbitmq.com/img/tutorials/python-six.png)

我们的RPC的处理流程：

1. 当客户端启动时，创建一个匿名的回调队列。
2. 客户端为RPC请求设置2个属性：replyTo，设置回调队列名字；correlationId，标记request。
3. 请求被发送到rpc_queue队列中。
4. RPC服务器端监听rpc_queue队列中的请求，当请求到来时，服务器端会处理并且把带有结果的消息发送给客户端。接收的队列就是replyTo设定的回调队列。
5. 客户端监听回调队列，当有消息时，检查correlationId属性，如果与request中匹配，那就是结果了。

- client

  ```java
  public class RPCClient {
  
      private Connection connection;
      private Channel channel;
      private String requestQueueName = "test_rpc_queue";
      private String replyQueueName;
  
      public RPCClient() throws IOException, TimeoutException {
          connection = ConnectionUtils.getConnection();
          channel = connection.createChannel();
          //创建一个匿名的回调队列
          replyQueueName = channel.queueDeclare().getQueue();
      }
  
      public String call(String message) throws IOException, InterruptedException {
          final String corrId = UUID.randomUUID().toString();
  
          //客户端为RPC请求设置2个属性：replyTo，设置回调队列名字；correlationId，标记request
          AMQP.BasicProperties props = new AMQP.BasicProperties
                  .Builder()
                  .correlationId(corrId)
                  .replyTo(replyQueueName)
                  .build();
  
          //请求被发送到rpc_queue队列中
          channel.basicPublish("", requestQueueName, props, message.getBytes("UTF-8"));
  
          final BlockingQueue<String> response = new ArrayBlockingQueue<String>(1);
  
          channel.basicConsume(replyQueueName, true, new DefaultConsumer(channel) {
              @Override
              public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                  //客户端监听回调队列，当有消息时，检查correlationId属性，如果与request中匹配，那就是结果了
                  if (properties.getCorrelationId().equals(corrId)) {
                      response.offer(new String(body, "UTF-8"));
                  }
              }
          });
  
          //close();
          return response.take();
      }
  
      public void close() throws IOException {
          connection.close();
      }
  
      public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
          RPCClient rpcClient = new RPCClient();
          System.out.println(rpcClient.call("2"));
      }
  
  }
  ```

- server

  ```java
  public class RPCServer {
  
      private static final String RPC_QUEUE_NAME = "test_rpc_queue";
  
      public static void main(String[] args) throws IOException, TimeoutException {
          Connection connection = ConnectionUtils.getConnection();
          final Channel channel = connection.createChannel();
          channel.queueDeclare(RPC_QUEUE_NAME, false, false, false, null);
          channel.basicQos(1);
  
          Consumer consumer = new DefaultConsumer(channel) {
              @Override
              public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                  super.handleDelivery(consumerTag, envelope, properties, body);
                  AMQP.BasicProperties properties1 = new AMQP.BasicProperties.Builder().correlationId(properties.getCorrelationId()).build();
                  String mes = new String(body, "UTF-8");
                  int num = Integer.valueOf(mes);
                  System.out.println("接收数据：" + num);
                  num = fib(num);
                  //把带有结果的消息发送给客户端
                  channel.basicPublish("", properties.getReplyTo(), properties1, String.valueOf(num).getBytes());
                  channel.basicAck(envelope.getDeliveryTag(), false);
              }
          };
          channel.basicConsume(RPC_QUEUE_NAME, false, consumer);
          while (true) {
              synchronized (consumer) {
                  try {
                      consumer.wait();
                  } catch (InterruptedException e) {
                      e.printStackTrace();
                  }
              }
          }
      }
  
      /*
          获取斐波那契数列的第n个值得大小
       */
      private static int fib(int n) {
          System.out.println(n);
          if (n == 0)
              return 0;
          if (n == 1)
              return 1;
          return fib(n - 1) + fib(n - 2);
      }
  }
  ```

## 整合spring

### 路由模式

- spring-rabbitmq-send.xml

  ```xml
  <beans xmlns="http://www.springframework.org/schema/beans"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:rabbit="http://www.springframework.org/schema/rabbit"
         xsi:schemaLocation="http://www.springframework.org/schema/rabbit
  	http://www.springframework.org/schema/rabbit/spring-rabbit-1.4.xsd
  	http://www.springframework.org/schema/beans
  	http://www.springframework.org/schema/beans/spring-beans-4.1.xsd">
  
      <!-- 配置连接工厂 -->
      <rabbit:connection-factory id="connectionFactory"
                                 host="127.0.0.1" port="5672" username="mgszjz" password="123" virtual-host="/mysql8.0.13"/>
  
      <!-- 定义mq管理 -->
      <rabbit:admin connection-factory="connectionFactory" />
  
      <!-- 声明队列 -->
      <rabbit:queue name="que_cat" auto-declare="true" durable="true" />
      <rabbit:queue name="que_pig" auto-declare="true" durable="true" />
  
      <!-- 定义交换机绑定队列（路由模式） -->
      <rabbit:direct-exchange name="IExchange"
                              id="IExchange">
          <rabbit:bindings>
              <rabbit:binding queue="que_cat" key="que_cat_key" />
              <rabbit:binding queue="que_pig" key="que_pig_key" />
          </rabbit:bindings>
      </rabbit:direct-exchange>
  
      <!-- 消息对象json转换类 -->
      <bean id="jsonMessageConverter"
            class="org.springframework.amqp.support.converter.Jackson2JsonMessageConverter" />
  
      <!-- 定义模版 -->
      <rabbit:template id="rabbitTemplate"
                       connection-factory="connectionFactory" exchange="IExchange"
                       message-converter="jsonMessageConverter" />
  
  </beans>
  ```

- spring-rabbitmq-receive.xml

  ```xml
  <beans xmlns="http://www.springframework.org/schema/beans"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:rabbit="http://www.springframework.org/schema/rabbit"
         xsi:schemaLocation="http://www.springframework.org/schema/rabbit
  	http://www.springframework.org/schema/rabbit/spring-rabbit-1.4.xsd
  	http://www.springframework.org/schema/beans
  	http://www.springframework.org/schema/beans/spring-beans-4.1.xsd">
  
      <!-- 配置连接工厂 -->
      <rabbit:connection-factory id="connectionFactory"
                                 host="127.0.0.1" port="5672" username="mgszjz" password="123" virtual-host="/mysql8.0.13"/>
  
      <!-- 定义mq管理 -->
      <rabbit:admin connection-factory="connectionFactory" />
  
      <!-- 声明队列 -->
      <rabbit:queue name="que_cat" auto-declare="true" durable="true" />
      <rabbit:queue name="que_pig" auto-declare="true" durable="true" />
  
      <!-- 定义消费者 -->
      <bean name="catHandler" class="com.springmvc.rabbitMQ.CatHandler" />
      <bean name="pigHandler" class="com.springmvc.rabbitMQ.PigHandler" />
  
      <!-- 定义消费者监听队列 -->
      <rabbit:listener-container
              connection-factory="connectionFactory">
          <rabbit:listener ref="catHandler" queues="que_cat" />
          <rabbit:listener ref="pigHandler" queues="que_pig" />
      </rabbit:listener-container>
  
  
  </beans>
  ```

- CatHandler.java

  ```java
  import java.io.IOException;
  
  import org.springframework.amqp.core.Message;
  import org.springframework.amqp.core.MessageListener;
  
  import com.fasterxml.jackson.databind.JsonNode;
  import com.fasterxml.jackson.databind.ObjectMapper;
  
  public class CatHandler implements MessageListener {
  
      private static final ObjectMapper MAPPER = new ObjectMapper();
  
      public void onMessage(Message msg) {
  
          try {
              //msg就是rabbitmq传来的消息
              // 使用jackson解析
              JsonNode jsonData = MAPPER.readTree(msg.getBody());
              System.out.println("我是可爱的小猫,我的id是" + jsonData.get("id").asText()
                      + ",我的名字是" + jsonData.get("name").asText());
  
          } catch (IOException e) {
              e.printStackTrace();
          }
      }
  
  }
  ```

- PigHandler.java

  ```java
  import org.springframework.amqp.core.Message;
  import org.springframework.amqp.core.MessageListener;
  
  import com.fasterxml.jackson.databind.JsonNode;
  import com.fasterxml.jackson.databind.ObjectMapper;
  
  import java.io.IOException;
  
  public class PigHandler implements MessageListener {
  
      private static final ObjectMapper MAPPER = new ObjectMapper();
  
      public void onMessage(Message msg) {
          try {
              //msg就是rabbitmq传来的消息，需要的同学自己打印看一眼
              // 使用jackson解析
              JsonNode jsonData = MAPPER.readTree(msg.getBody());
              System.out.println("我是可爱的小猪,我的id是" + jsonData.get("id").asText()
                      + ",我的名字是" + jsonData.get("name").asText());
  
          } catch (IOException e) {
              e.printStackTrace();
          }
      }
  
  }
  ```

- Controller中发送消息

  ```java
  @Controller
  public class RabbitmqController {
  
      @Autowired
      private RabbitTemplate rabbitTemplate;
  
      @RequestMapping(value = "/rabbit1", method=RequestMethod.GET)
      public void test1(){
  
          HashMap<String, String> map = new HashMap<>();
          map.put("id", "1");
          map.put("name", "pig");
          //根据key发送到对应的队列
          rabbitTemplate.convertAndSend("que_pig_key", map);
  
          map.put("id", "2");
          map.put("name", "cat");
          //根据key发送到对应的队列
          rabbitTemplate.convertAndSend("que_cat_key", map);
  
      }
  }
  ```

### 主题模式

- spring-rabbitmq-send.xml修改：

  ```xml
      <!-- 定义交换机绑定队列（主题模式） -->
      <rabbit:topic-exchange name="IExchange"
                              id="IExchange">
          <rabbit:bindings>
              <rabbit:binding queue="que_cat" pattern="cat.#"/>
              <rabbit:binding queue="que_pig" pattern="pig.#" />
          </rabbit:bindings>
      </rabbit:topic-exchange>
  ```

- Controller中发送消息

  ```java
  @Controller
  public class RabbitmqController {
  
      @Autowired
      private RabbitTemplate rabbitTemplate;
  
      @RequestMapping(value = "/rabbit1", method=RequestMethod.GET)
      public void test1(){
  
          HashMap<String, String> map = new HashMap<>();
          map.put("id", "1");
          map.put("name", "pig");
          //根据key发送到对应的队列
          rabbitTemplate.convertAndSend("que_pig_key", map);
  
          map.put("id", "2");
          map.put("name", "cat");
          //根据key发送到对应的队列
          rabbitTemplate.convertAndSend("que_cat_key", map);
  
      }
  }
  ```

注意：调试时，修改交换机模式需要删除交换机和队列重新调试！



## 消息应答

```java
boolean autoAck=true;  //自动确认模式
```

一旦rabbitMQ将消息分发给消费者，就从内存中删除。如果杀死正在执行的消费者，就会丢失正在处理的消息.

```java
boolean autoAck=false;  //手动确认模式
```

如果有一个消费者挂掉，就会交付给其他消费者。rabbitMQ收到消费者的消息应答，才从内存中删除消息。

## 消息持久化(解决rabbitMQ服务器宕机的数据丢失问题)

```java
boolean durable=false; //不持久化
channel.queueDeclare(Sender.QUEUE_NAME,durable,false,false,null);
```

队列一旦声明，无法更改durable的属性（如名叫`Sender.QUEUE_NAME`的队列直接已经定义不持久化，即使当前程序重新定义`durable=true`，会报错。

```java
boolean durable=true; //持久化
```

## 消息确认(解决进入队列之前的数据丢失问题)

生产者发出消息，rabbitMQ有否收到无从得知

### AMQP事务机制

1. channel.txSelect()声明启动事务模式；
2. channel.txComment()提交事务；
3. channel.txRollback()回滚事务；

- 生产者

  ```java
  public class Sender {
      public static final String QUEUE_NAME="test_tx_queue";
  
      public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
          Connection connection=ConnectionUtils.getConnection();
          //从链接中获取 通道
          Channel channel=connection.createChannel();
  
          //声明一个队列
          boolean durable=false;
          channel.queueDeclare(QUEUE_NAME,durable,false,false,null);
  
          //每个消费者发送确认之前，消息队列不发送下一个消息到消费者，消费者必须关闭自动应答ack，改成手动
          //channel.basicQos(1);
  
          try {
              channel.txSelect();
  
              String msg="hello tx";
              channel.basicPublish("",QUEUE_NAME,null,msg.getBytes());
  
              int xx=1/0;
  
              channel.txCommit();
          } catch (Exception e) {
              channel.txRollback();
              System.out.println("txRollBack");
          }
  
          channel.close();
          connection.close();
      }
  }
  ```

- 消费者

  ```java
  public class Receiver1 {
      public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
          Connection connection=ConnectionUtils.getConnection();
  
          final Channel channel=connection.createChannel();
  
          //每个消费者发送确认之前，消息队列不发送下一个消息到消费者，消费者必须关闭自动应答ack，改成手动
  //        channel.basicQos(1);
  
          //新版本定义消费者
          //声明一个队列
          boolean durable=false;
          channel.queueDeclare(Sender.QUEUE_NAME,durable,false,false,null);
          Consumer consumer=new DefaultConsumer(channel){
              //一旦有消息就会触发方法
              @Override
              public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                  String msg=new String(body,"utf-8");
                  System.out.println(msg);
              }
          };
  
          //监听队列
          boolean autoAck=true;
          channel.basicConsume(Sender.QUEUE_NAME,autoAck,consumer);
      }
  }
  ```

由于使用协议，性能较差

### Confirm发送方确认模式

#### channel.waitForConfirms()普通发送方确认模式

我们只需要在推送消息之前，channel.confirmSelect()声明开启发送方确认模式，再使用channel.waitForConfirms()等待消息被服务器确认即可。

- 发送者

  ```java
  public class Sender {
      public static final String QUEUE_NAME="test_confirm_queue";
  
      public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
          Connection connection=ConnectionUtils.getConnection();
          //从链接中获取 通道
          Channel channel=connection.createChannel();
  
          //声明一个队列
          boolean durable=false;
          channel.queueDeclare(QUEUE_NAME,durable,false,false,null);
  
          //每个消费者发送确认之前，消息队列不发送下一个消息到消费者，消费者必须关闭自动应答ack，改成手动
          //channel.basicQos(1);
  
          //设成confirm模式
          channel.confirmSelect();
  
          String msg="hello tx";
          channel.basicPublish("",QUEUE_NAME,null,msg.getBytes());
  
          if (!channel.waitForConfirms()){ //若接收端没有确认，发送失败
              System.out.println("fail");
          } else {
              System.out.println("success");
          }
  
          channel.close();
          connection.close();
      }
  }
  ```

- 接受者

  同AMQP事务机制的消费者代码

#### channel.waitForConfirmsOrDie()批量确认模式

- 发送者

  ```java
  public class Sender2 {
      public static final String QUEUE_NAME="test_confirm_queue2";
  
      public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
          Connection connection=ConnectionUtils.getConnection();
          //从链接中获取 通道
          Channel channel=connection.createChannel();
  
          //声明一个队列
          boolean durable=false;
          channel.queueDeclare(QUEUE_NAME,durable,false,false,null);
  
          //每个消费者发送确认之前，消息队列不发送下一个消息到消费者，消费者必须关闭自动应答ack，改成手动
          //channel.basicQos(1);
  
          //设成confirm模式
          channel.confirmSelect();
  
          for (int i = 0; i < 10; i++) {
              String msg="hello tx";
              channel.basicPublish("",QUEUE_NAME,null,msg.getBytes());
          }
  
          channel.waitForConfirmsOrDie(); //直到所有信息都发布，只要有一个未确认就会IOException
  
  //        if (!channel.waitForConfirms()){ //若接收端没有确认，发送失败
  //            System.out.println("fail");
  //        } else {
  //            System.out.println("success");
  //        }
  
          channel.close();
          connection.close();
      }
  }
  ```

- 接受者

  同AMQP事务机制的消费者代码，更改`Sender.QUEUE_NAME`为`Sender2.QUEUE_NAME`即可。

#### channel.addConfirmListener()异步监听发送方确认模式

- 发送者

  ```java
  public class Sender3 {
      public static final String QUEUE_NAME="test_confirm_queue3";
  
      public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
          Connection connection=ConnectionUtils.getConnection();
          //从链接中获取 通道
          Channel channel=connection.createChannel();
  
          //声明一个队列
          boolean durable=false;
          channel.queueDeclare(QUEUE_NAME,durable,false,false,null);
  
          //每个消费者发送确认之前，消息队列不发送下一个消息到消费者，消费者必须关闭自动应答ack，改成手动
          //channel.basicQos(1);
  
          //设成confirm模式
          channel.confirmSelect();
  
          //未确认的消息标识
          final SortedSet<Long> confirmSet=Collections.synchronizedSortedSet(new TreeSet<Long>());
  
          //异步监听确认和未确认的消息
          channel.addConfirmListener(new ConfirmListener() {
              @Override
              public void handleNack(long deliveryTag, boolean multiple) throws IOException {
                  if (multiple) {
                      System.out.println("未确认消息，标识：" + deliveryTag);
                      confirmSet.headSet(deliveryTag+1).clear();
                  } else {
                      System.out.println("multiple false---未确认消息，标识：" + deliveryTag);
                      confirmSet.remove(deliveryTag);
                  }
              }
  
              //没有问题
              @Override
              public void handleAck(long deliveryTag, boolean multiple) throws IOException {
                  if (multiple) {
                      System.out.println(String.format("已确认消息，标识：%d，多个消息：%b", deliveryTag, multiple));
                      confirmSet.headSet(deliveryTag+1).clear();
                  } else {
                      System.out.println(String.format("multiple false---已确认消息，标识：%d，多个消息：%b", deliveryTag, multiple));
                      confirmSet.remove(deliveryTag);
                  }
              }
          });
  
          for (int i = 0; i < 10000; i++) {
              String msg="hello tx";
              channel.basicPublish("",QUEUE_NAME,null,msg.getBytes());
              long seqNo=channel.getNextPublishSeqNo();
              confirmSet.add(seqNo);
          }
  
          channel.close();
          connection.close();
      }
  }
  ```

- 接受者

  同AMQP事务机制的消费者代码，更改`Sender.QUEUE_NAME`为`Sender3.QUEUE_NAME`即可

结果如下：

```
已确认消息，标识：13，多个消息：true
已确认消息，标识：20，多个消息：true
multiple false---已确认消息，标识：21，多个消息：false
已确认消息，标识：28，多个消息：true
已确认消息，标识：45，多个消息：true
已确认消息，标识：66，多个消息：true
已确认消息，标识：87，多个消息：true
已确认消息，标识：108，多个消息：true
已确认消息，标识：117，多个消息：true
已确认消息，标识：128，多个消息：true
已确认消息，标识：140，多个消息：true
已确认消息，标识：146，多个消息：true
已确认消息，标识：155，多个消息：true
已确认消息，标识：163，多个消息：true
已确认消息，标识：173，多个消息：true
已确认消息，标识：182，多个消息：true
已确认消息，标识：198，多个消息：true
multiple false---已确认消息，标识：199，多个消息：false
multiple false---已确认消息，标识：200，多个消息：false
```

