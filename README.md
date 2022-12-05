# go-fastermq
***
我对kafka sdk进行了一系列有效的封装，下面我将介绍一下如何使用sdk 在使用之前我们需要进行client配置,你可以在任何地方进行配置
```go
var (
	hosts   = []string{"127.0.0.1:9092"}
	topic   = "test"
	groupId = "test-group"
)
```

## 创建消费者
***
这里我们使用了消费者组

```go
func main() {
    _, err := kafka.StartKafkaConsumer(hosts, []string{topic}, groupId, nil, msgHandler)
    if err != nil {
        fmt.Println(err)
    }
    signals := make(chan os.Signal, 1)
    signal.Notify(signals, syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)
    select {
        case s := <-signals:
        	kafka.KafkaStdLogger.Println("kafka test receive system signal:", s)
    		return
	}
}

func msgHandler(message *sarama.ConsumerMessage) (bool, error) {
    fmt.Println("消费消息:", "topic:", message.Topic, "Partition:", message.Partition, "Offset:", message.Offset, "value:", string(message.Value))
    msg := Msg{}
    err := json.Unmarshal(message.Value, &msg)
    if err != nil {
        //解析不了的消息怎么处理？
        logger.Error("Unmarshal error", zap.Error(err))
        return true, nil
    }
    fmt.Println("msg : ", msg)
    return true, nil
}
```

## 创建生产者
***

你可以根据你的具体业务定义自己的消息体，这里我给出了两个示例，你可以使用同步生产和异步生产。当然一般情况下我们推荐异步生产者，
因为他的效率更高，应用更为广泛。我采用了map[name]的方式来创建和管理生产者，通过`InitAsyncKafkaProducer`或者`InitSyncKafkaProducer`方法来创建生产者
我们会把他放在一个map当中，然后通过`GetKafkaAsyncProducer`或者`GetKafkaSyncProducer`来获取生产者

```go
// 消息体
type Msg struct {
	ID       int64  `json:"id"`
	Name     string `json:"name"`
	CreateAt int64  `json:"create_at"`
}

func produceAsyncMsg() {
	err := kafka.InitAsyncKafkaProducer(kafka.DefaultKafkaAsyncProducer, hosts, nil)
	if err != nil {
		fmt.Println("InitAsyncKafkaProducer error", err)
	}
	msg := Msg{
		ID:       1,
		Name:     "test name async",
		CreateAt: time.Now().Unix(),
	}
	msgBody, _ := json.Marshal(msg)

	err = kafka.GetKafkaAsyncProducer(kafka.DefaultKafkaAsyncProducer).Send(&sarama.ProducerMessage{Topic: topic, Value: kafka.KafkaMsgValueEncoder(msgBody)})
	if err != nil {
		fmt.Println("Send msg error", err)
	} else {
		fmt.Println("Send msg success")
	}
	//异步提交需要等待
	time.Sleep(3 * time.Second)
}
func produceSyncMsg() {
	err := kafka.InitSyncKafkaProducer(kafka.DefaultKafkaSyncProducer, hosts, nil)
	if err != nil {
		fmt.Println("InitSyncKafkaProducer error", err)
	}

	msg := Msg{
		ID:       2,
		Name:     "test name sync",
		CreateAt: time.Now().Unix(),
	}
	msgBody, _ := json.Marshal(msg)
	partion, offset, err := kafka.GetKafkaSyncProducer(kafka.DefaultKafkaSyncProducer).Send(&sarama.ProducerMessage{Topic: topic, Value: kafka.KafkaMsgValueEncoder(msgBody)})
	if err != nil {
		fmt.Println("Send msg error", err)
	} else {
		fmt.Println("Send msg success partion ", partion, "offset", offset)
	}
}
```