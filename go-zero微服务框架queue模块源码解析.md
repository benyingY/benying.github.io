![image-20201030175836722](/Users/yangzhiquan/Library/Application Support/typora-user-images/image-20201030175836722.png)

https://github.com/tal-tech/go-zero/tree/master/core/queue



**go-zero是晓黑板开源的微服务框架，经过晓黑板实战检验，值得一学。**今天先尝试找个简单的模块源码来写一下。

queue是个简单的队列，代码链接和结构如上。



#### 1 主要的结构就是队列queue，生产者producer和消费者consumer。生产者和消费者是接口，所以主要实现在队列queue里。

```go
package queue

type (
	Producer interface {
    //给生产者添加监听
		AddListener(listener ProduceListener)
		Produce() (string, bool)
	}

	ProduceListener interface {
    //通过这两个方法给生产者发监听消息
		OnProducerPause()
		OnProducerResume()
	}

	ProducerFactory func() (Producer, error)
)
```

```go
package queue

type (
	Consumer interface {
		Consume(string) error
    //消费者响应Broadcast广播的消息
		OnEvent(event interface{})
	}

	ConsumerFactory func() (Consumer, error)
)
```



#### 2 下面主要看一下队列queue

**2.1 队列的数据结构**

```go
	Queue struct {
		name                 string
		metrics              *stat.Metrics
		producerFactory      ProducerFactory
		producerRoutineGroup *threading.RoutineGroup
		consumerFactory      ConsumerFactory
		consumerRoutineGroup *threading.RoutineGroup
		producerCount        int                //生产者数量
		consumerCount        int                //消费者数量
		active               int32              //活跃的生产者数量
		channel              chan string        //真正的队列
		quit                 chan struct{}
		listeners            []Listener         //生产者的监听
		eventLock            sync.Mutex					//锁
		eventChannels        []chan interface{} //消费者监听广播事件的channels
	}
```



**2.2 启动生产者和消费者**

```go
func (queue *Queue) Start() {
	queue.startProducers(queue.producerCount)
	queue.startConsumers(queue.consumerCount)

	//当生产者都关闭后，队列的channel就可以关闭了。然后消费者都关闭。
	queue.producerRoutineGroup.Wait()
	close(queue.channel)
	queue.consumerRoutineGroup.Wait()
}
```



**启动生产者的循环**

```go
func (queue *Queue) startProducers(number int) {
	for i := 0; i < number; i++ {
		//用RoutineGroup来启动所有生产者，RoutineGroup是用sync.WaitGroup实现的
		queue.producerRoutineGroup.Run(func() {
			queue.produce()
		})
	}
}
```



**启动生产者的具体实现**

- 用工厂方法producerFactory初始化生产者

- 原子增加队列的活跃生产者数量queue.active
- 生产者添加监听producer.AddListener。参数routineListener实现了ProduceListener接口。

- 监听queue.quit通道关闭。默认情况下生产消息produceOne，发送到queue.channel。produceOne调用了producer.Produce，是生产者接口的方法，需要自己实现。

```go
func (queue *Queue) produce() {
	var producer Producer

	for {
		var err error
		//用工厂方法初始化生产者。如果出错，休息1s，然后不断重试，不死不休。
		if producer, err = queue.producerFactory(); err != nil {
			logx.Errorf("Error on creating producer: %v", err)
			time.Sleep(time.Second)
		} else {
			break
		}
	}

	atomic.AddInt32(&queue.active, 1)
	producer.AddListener(routineListener{
		queue: queue,
	})

	for {
		select {
		//quit通道关闭
		case <-queue.quit:
			logx.Info("Quitting producer")
			return
		default:
			//默认情况下生产
			if v, ok := queue.produceOne(producer); ok {
				queue.channel <- v
			}
		}
	}
}
```

```go
func (queue *Queue) produceOne(producer Producer) (string, bool) {
	// avoid panic quit the producer, just log it and continue
	defer rescue.Recover()

	//调用生产者接口的实际生产方法
	return producer.Produce()
}
```



**生产者的监听**

- routineListener实现了ProduceListener接口。ProduceListener接口，看方法字面意思是实现了生产者的生产速度调节，并且是同时调节所有监听的生产者。
- OnProducerPause当active生产者的数量小于等于1的时候，调用queue.pause。OnProducerResume当active的数量等于0的时候，调用queue.resume。queue.pause和queue.resume 都会调用监听当前队列的生产者的方法。

```go
type routineListener struct {
	queue *Queue
}

func (rl routineListener) OnProducerPause() {
	if atomic.AddInt32(&rl.queue.active, -1) <= 0 {
		rl.queue.pause()
	}
}

func (rl routineListener) OnProducerResume() {
	if atomic.AddInt32(&rl.queue.active, 1) == 1 {
		rl.queue.resume()
	}
}

func (queue *Queue) pause() {
	for _, listener := range queue.listeners {
		listener.OnPause()
	}
}

func (queue *Queue) resume() {
	for _, listener := range queue.listeners {
		listener.OnResume()
	}
}
```



**启动消费者**

```go
func (queue *Queue) startConsumers(number int) {
	for i := 0; i < number; i++ {
		//每个消费者会有自己的事件channel
		eventChan := make(chan interface{})
		queue.eventLock.Lock()
		queue.eventChannels = append(queue.eventChannels, eventChan)
		queue.eventLock.Unlock()
		queue.consumerRoutineGroup.Run(func() {
			queue.consume(eventChan)
		})
	}
}
```



**启动消费者的具体实现**

- 用工厂方法初始化消费者，如果失败，不断重试，不死不休。

- for的死循环。

  - 从队列的channel获取消息，然后调用consumeOne消费消息。consumeOne调用了consumer.Consume，是消费者接口的方法，需要自己实现。如果channel返回不ok，表示channel已经关闭，记录日志，消费者退出。

  - 从事件channel监听事件，并响应。OnEvent是接口的方法，需要自己实现。

```go
func (queue *Queue) consume(eventChan chan interface{}) {
	var consumer Consumer

	for {
		var err error
		//跟生产者一样，用了工厂方法。也是不断重试，不死不休。
		if consumer, err = queue.consumerFactory(); err != nil {
			logx.Errorf("Error on creating consumer: %v", err)
			time.Sleep(time.Second)
		} else {
			break
		}
	}

	for {
		select {
		//从队列的channel取消息
		case message, ok := <-queue.channel:
			if ok {
				//消费消息
				queue.consumeOne(consumer, message)
			} else {
				//如果不ok，表示队列的channel关闭，消费者也退出
				logx.Info("Task channel was closed, quitting consumer...")
				return
			}
			//从事件channel监听事件
		case event := <-eventChan:
			consumer.OnEvent(event)
		}
	}
}
```



**消费的具体实现**

```go
func (queue *Queue) consumeOne(consumer Consumer, message string) {
	//RunSafe加了recover()
	threading.RunSafe(func() {
		startTime := timex.Now()
		//defer用来记录metrics和日志。日志包括消费所耗事件
		defer func() {
			duration := timex.Since(startTime)
			queue.metrics.Add(stat.Task{
				Duration: duration,
			})
			logx.WithDuration(duration).Infof("%s", message)
		}()

		//Consume是接口方法，需要自己实现
		if err := consumer.Consume(message); err != nil {
			logx.Errorf("Error occurred while consuming %v: %v", message, err)
		}
	})
}
```



**2.3 往消费者的eventChannels发广播消息**

```go
func (queue *Queue) Broadcast(message interface{}) {
	go func() {
		queue.eventLock.Lock()
		defer queue.eventLock.Unlock()

		for _, channel := range queue.eventChannels {
			channel <- message
		}
	}()
}
```



#### 3 多生产者

**3.1 平衡的生产者组BalancedPusher**

```go
//直译就是平衡的生产者组
type BalancedPusher struct {
	name    string
	pushers []Pusher //生产者组
	index   uint64   //前一次push的生产者的index
}

func (pusher *BalancedPusher) Push(message string) error {
	size := len(pusher.pushers)

	for i := 0; i < size; i++ {
		//原子操作，防止上层多个goroutine调用Push
		//index自增，然后对生产者的数量取模，取这次push的生产者。类似轮训的策略，达到平均效果。
		index := atomic.AddUint64(&pusher.index, 1) % uint64(size)
		target := pusher.pushers[index]

		//如果失败，记日志，并取下个生产者继续push，直到每个生产者都试一遍。如果成功，直接返回。
		if err := target.Push(message); err != nil {
			logx.Error(err)
		} else {
			return nil
		}
	}

	//最终失败，返回错误信息
	return ErrNoAvailablePusher
}
```



**3.2 普通多生产者MultiPusher**

```
type MultiPusher struct {
	name    string
	pushers []Pusher
}

func (pusher *MultiPusher) Push(message string) error {
	var batchError errorx.BatchError

	for _, each := range pusher.pushers {
		if err := each.Push(message); err != nil {
			//这里会记录每个生产者的错误
			batchError.Add(err)
		}
	}

	//根据错误的数量，返回不同的错误内容
	return batchError.Err()
}
```



先写个简单的，大家见笑了。欢迎吐槽。点赞，在看，转发走起！！！