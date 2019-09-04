---
layout: post
title: Redis Streams and Java Lambdas
maintitle: Streaming From Redis in Java
tags: [java, redis, september, 2019]
---

Recently I've learned about Redis' [streaming feature](https://redis.io/topics/streams-intro).  The benefit of using this over something like Kafka for a log/stream is that managed Redis is quite common across cloud providers.  Using the consumer groups is also an easy way of reading from the stream using multiple consumers without having them step on each other's feet doesn't require much configuration.  Knowing we can stream data, I wanted to use Java's streams and functional constructs to make a mess of things.

## Setup
To interface with Redis, I've found that [Redisson](https://github.com/redisson/redisson/tree/master/redisson-spring-boot-starter#spring-boot-starter) is a great library to use, and integrates with Spring Boot well.  It has some other powerful data structures, but its streaming interface is promising.

To start, I like to have a docker-compose set up for local development using the base images:
```yaml
# docker-compose.yml
version: '3.6'

services:
  redis:
    image: redis:latest
    ports:
        - 6379:6379
```

Now we want to add some test data to Redis.  Start docker using `docker-compose up -d` then `docker ps` to grab the image's id, we can get into it to do some test commands using `docker exec -it <docker-id> /bin/bash`.
Use  the following to populate Redis:
```bash
redis-cli XADD test-stream * test 1234
```

After using the spring boot initializer, just adding this line integrates Redisson:
```groovy
// build.gradle
implementation 'org.redisson:redisson-spring-boot-starter:3.11.2'
```

And with a simple Bean we have a Redis client to interact with:
```java
// RedisClientBean.java
@Configuration
public class RedisClientBean {
            
    @Bean
    public RedissonClient getRedisClient() {
        return Redisson.create();
    }   
}
```

## The Meat

First we can define a simple service that has an event listener to initialize the stream when the Spring Boot Application has finished starting and loading its Beans:
```java
// RedisStreamService.java
@Service
public class RedisStreamService {
    
    private static final Logger logger = LoggerFactory.getLogger(RedisStreamService.class);

    private RedissonClient redisClient;

    public RedisStreamService(RedissonClient redisClient) {
        this.redisClient = redisClient;
    }

    @EventListener(ContextRefreshedEvent.class)
    public void initStream() {
         logger.info("Inside of init stream");
    }
}
```

Okay with that we can assume that most code goes inside of these two functions, and anything in `ALL_CAPS` are pre-defined constants.

Next we need to initialize the Redis stream and create a group:
```java
RStream<String, String> rstream = redisClient.getStream(STREAM_NAME);
rstream.createGroup(GROUP_NAME);
```

Now, sadly `RStream` is not a `Collection`, so it is not 'streamable' despite it's name.  We'll have to generate a stream by reading from the RStream in a lambda and then act on each RFuture returned.  After that, we can expand on some better ways of processing things.
```java
Stream.generate( () -> {
    return rstream.readGroupAsnc(GROUP_NAME, CONSUMER_NAME);
})
.forEach(future -> {
    future.thenAccept(res -> {
        Map<StreamMessageId, Map<String, String>> result =
            (Map<StreamMessageId, Map<String, String>>) res;
        StreamMessageId id = result
            .entrySet()
            .iterator()
            .next()
            .getKey();

        Map<String, String> resultMap = result
            .entrySet()
            .iterator()
            .next()
            .getValue()

        String value = resultMap
            .entrySet()
            .iterator()
            .next()
            .getValue();

        if (!StringUtils.isEmpty(value)) {
            logger.info("Recieved message: {}", value);
            rstream.ack(GROUP_NAME, id);
        }
    }).exceptionally(exception -> {
        logger.error("Exception raised while processing redis stream {}",
            exception.getCause().toString());
    });
});
```

Now, there are a few things wrong with this.  First, it polls redis continually without breaks, which isn't great.  We can impliment a simple timer with a semaphore to take care of that.  The other thing is that the RFuture is blocking.  We can do this processing async!  So we can wrap the above code in an `ExecutorService` and use `.parallel()` to process each `Future` individually.

```java
ExecutorService executorService = Executors.newCachedThreadPool();

...
// Inside of init stream
executorService.submit( () ->
    // Code from above.
    Stream.generate( () ->
        ....
    ));
    .parallel() // This part is new.
    .foreach(future -> {
    })
    ....
```

And for the timer:
```java
private volatile Semaphore readTimer = new Semaphore(1, true);
ScheduledExecutorService scheduler = Executors.newScheduledTreadPool(1);

Runnable unlockRunnable = () -> {
    readTimer.release();
};

...

@EventListener(ContextRefreshedEvent.class)
public void initStream() {
    scheduler.scheduleAtFixedRate(
        unlockRunnable,
        0,
        READ_INTERVAL,
        TimeUnit.SECONDS);

    ....

    executorService.submit( () ->
        Stream.generate( () -> {
	    readTimer.acquireUninterruptibly();
	    return rstream.readGroupAsync(GROUP_NAME, 
                CONSUMER_NAME);
        })
        .parallel()
        .forEach(future -> {
            ....
        }));

    ....
}
```

And there it is.  `RStream`'s `readGroupAsync` has a count parameter, so these can be grabbed in batches and processed.  Using Redis' stream groups we can have multiple consumers to one stream, and if one crashes mid processing then the old data can be grabbed and processed on when a consumer with that id comes back up.

Hope it was helpful.

### EDIT
I've realized that you _don't_ infact have to use a timer with this!  If you provide a timeout to the readGroupAsync then Redis blocks.  Now, it doesn't block in the same manner that it does for the `KEYS` Redis command, where it locks up based on the size of the key space.  This just spins on the client's connection.  So the revised code can be found below:

```java

``ce
public class RedisStreamService {
    
    private static final String BANKRUPTCY_101_STREAM = "testStream";
    private static final String GROUP_NAME = "testGroup";
    private static final String CONSUMER_NAME = "testConsumer"; 
    private static final int READ_INTERVAL = 10000;

    private RedissonClient redisClient;
    
    
    ExecutorService executorService ;
    Runnable unlockSemaphoreRunnable;
    
    public RedisStreamService(RedissonClient redisClient) {
        this.redisClient = redisClient;

        executorService = Executors.newCachedThreadPool();
    }
    
    @EventListener(ContextRefreshedEvent.class)
    public void initStream() {

        RStream<byte[], byte[]> rstream = redisClient.getStream(BANKRUPTCY_101_STREAM);
        RStream<byte[], byte[]> rstream = redisClient.getStream(BANKRUPTCY_101_STREAM);

        try {
            rstream.createGroup(GROUP_NAME);
        } catch  (RedisException e) {
            LOG.info("Redis group {} already exists.", GROUP_NAME);
        }

        executorService.submit( () ->
            Stream.generate( () -> {
                return rstream.readGroupAsync(GROUP_NAME, CONSUMER_NAME, READ_INTERVAL, TimeUnit.SECONDS);
            })
            .parallel()
            .forEach(future -> {
                future.thenAccept(res -> {
                    Map<StreamMessageId, Map<byte[], byte[]>> result =
                        (Map<StreamMessageId, Map<byte[], byte[]>>) res;

                    if (result.isEmpty()) {
                        return;
                    }
                    StreamMessageId id = result
                        .entrySet()
                        .iterator()
                        .next()
                        .getKey();

                    Map<byte[], byte[]> resultMap = result
                        .entrySet()
                        .iterator()
                        .next()
                        .getValue();

                    byte[] value = resultMap
                        .entrySet()
                        .iterator()
                        .next()
                        .getValue();

                     LOG.info("Recieved message from stream: " + new String(value));
                     rstream.ack(GROUP_NAME, id);
                     }).exceptionally(exception -> {
                         LOG.error("Exception raised while processing redis stream {}",
                     exception.getCause().toString());
                                        return null;
                     });
             }));
        }

```

Hopefully _this_ is more helpful.

Matt
