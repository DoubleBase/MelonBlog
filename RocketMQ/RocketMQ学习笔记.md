# 生产者

## 消息发送方式

- 同步发送

  ```java
  SendResult sendResult = producer.send(message);
  ```

- 异步发送

  ```java
  producer.send(message, new SendCallback() {
      public void onSuccess(SendResult sendResult) {
          System.out.println("异步发送成功：" + sendResult.toString());
      }
  
      public void onException(Throwable throwable) {
          System.out.println("异步发送失败");
          throwable.printStackTrace();
      }
  });
  ```

- 单向发送（没有返回结果）

  ```java
  producer.sendOneway(message);
  ```

## 发送成功返回状态码

- SEND_OK：消息发送成功
- FLUSH_DISK_TIMEOUT：消息发送成功，但broker刷盘超时；此时消息在broker内存中，未持久化到磁盘中，会有丢失风险
- FLUSH_SLAVE_TIMEOUT：消息发送成功，但同步给slave broker超时；此时消息在broker中已经持久化，但是未同步到 slave broker 中，一旦master宕机，选举了这个slave当选新的master，会导致消息丢失
- SLAVE_NOT_AVALIABLE：消息发送成功，但slave不可用



# 消息重发场景

- 生产者将消息成功发送给broker，但broker在响应的过程中超时，producer重发，导致消息重复
- 消费端处理消息，将成功ack响应给broker，这个过程中网络超时，broker未收到消费端的成功接收响应，此时broker会重发

# 消费者

- 幂等性：由于消息在RocketMQ中流转，避免不了消息重发，对于消息重复敏感的业务，需要进行**幂等**处理

# Broker

- 刷盘方式：
  - SYNC_FLUSH：同步刷盘
  - ASYNC_FLUSH：异步刷盘

# 功能

## 顺序消息

- 发送方根据订单id将消息推送到queue，同一订单id发到相同的queue上，保证queue上有序
- 每个queue都有唯一的consumer线程来消费，订单对每个queue有序

## 延时消息

- 设置消息延时等级 `message.setDelayTimeLevel(3)`

- 延迟等级对应时间，例如：等级3 对应 延时10秒

  ```java
  private String messageDelayLevel = "1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h";
  ```

## 事务消息

- 创建事务监听器

  ```java
  TransactionListener listener = new TransactionListener() {
  	public LocalTransactionState executeLocalTransaction(Message message, Object o) {
      	// 执行本地事务
          return LocalTransactionState.COMMIT_MESSAGE;
      }
      
      public LocalTransactionState checkLocalTransaction(MessageExt messageExt) {
       	// 校验本地事务执行状态
          return LocalTransactionState.COMMIT_MESSAGE;
      } 
  }
  ```

- 创建事务检查线程池

  ```java
  ExecutorService executorService = new ThreadPoolExecutor(2, 5, 100,
                  TimeUnit.SECONDS, new ArrayBlockingQueue<Runnable>(2000), new ThreadFactory() {
  
        public Thread newThread(Runnable r) {
            Thread thread = new Thread();
            thread.setName("client-transaction-msg-check-thread");
            return thread;
        }
  });
  ```

- 创建事务生产者

  ```java
  TransactionMQProducer producer = new TransactionMQProducer();
  producer.setExecutorService(executorService);
  producer.setTransactionListener(listener);
  ```

- 事务消息参数

  ```java
  // 事务执行超时时间
  private long transactionTimeOut = 6 * 1000;
  // 事务检查次数
  private int transactionCheckMax = 15;
  // 事务检查间隔
  private long transactionCheckInterval = 60 * 1000;
  ```

  