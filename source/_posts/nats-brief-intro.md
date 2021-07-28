---
title: "NATS 與 JetStream 簡易介紹"
date: 2021-07-29 00:23:00
categories: [notes]
tags: [nats, mq, golang]
---

最近因公司業務在玩一套相對新的 MQ: NATS。
因為官方文件不慎清楚且有些地方與直覺不同，造成起步緩慢。

以下簡單紀錄一下剛入門時應知道的事情。

## Guide

相較 RabbitMQ, Kafka 等，[NATS](https://nats.io/) 是一套較為年輕的 MQ。
雖然有部分子專案的版本未達 v1.0，但官方宣稱已經接近 production ready。

NATS 從一開始就是針對 cloud service 設計，cluster mode 的水平擴展，
node 之間的身分驗證及 TLS 通訊設計看起來都還不錯。

NATS 的 message 並無特別限制，在 client library 內任何的 byte sequence 都可以成為 message。

NATS 有以下三個模式(以及其對應的 client library)。

### NATS (NATS Core)

NATS 專案從一開始發展時的基本模式。 支援 **Pub/Sub** pattern 並提供 `at-most-once` 語意。

### NATS Streaming

NATS Streaming 是一套疊在 NATS 上面形成的 solution。

因為設計上的問題，後來又有了 JetStream，所以我們基本上不用理它，只要知道 NATS Streaming 和
JetStream 不一樣，翻文件的時候不要翻錯即可。

### JetStream

JetStream 是後來做在 NATS 內，可選擇是否啟用的子系統。 藉由 JetStream，
可以實作 **Producer/Consumer** pattern 並提供 `at-least-once` 語意。

Server side 沒什麼需要注意的，只要用較新版的 NATS image 並啟用設定即可。
Client 開發則需要注意一些概念。

- Subject: NATS 最初的概念，代表一些 message 的集合。
- Stream: 建立於一或多個 Subject 之上，可將這些 subject 內的 message 統整起來，並放入 persistent storage。
- Consumer: 建立在某個 Stream 之下，可以依序的 consume 屬於此 stream 的特定 message。

需要注意的是，不只 Subject 與 Stream，Consumer 本身也是建立在 NATS server 中的一個物件。
當利用 client library create 一個 Consumer 時，並不是該 process 本身成為一個 consumer，
而是 NATS server 中被創了一個 Consumer 物件，準備去使用 Stream 裡面的 message。

JetStream client library 並沒有提供一個對稱的 producer/consumer API。
基於術語的限制以及為了避免誤會，以下在稱呼一般所稱的 producer/consumer 時，
會特別加上 **role** 後綴來表示。

Producer role: 要使用 NATS library 內的 Publish API，將產生的 message 推送至
某個 Subject 內。

Consumer role: 要使用 JetStream library 內的 Stream API，在 NATS server 上對目標
Subject 建立 Stream，接著使用 JetStream Consumer API，在 NATS server 中
建立屬於該 Stream 的 Consumer。以上都完成之後，即可利用 Consumer 上的 `NextMsg` 來
消耗 message。

## Conclusion

JetStream 的 API 設計並不常見，需要先認知到與既有設計的差別之處才能開始開發。
不過其 cloud native 的架構設計或許可以在維運上面勝過其他老牌的 MQ solution。

今天就先寫到這裡，如果有哪天有興趣再補吧。 :D

## Reference

- [Comparing NATS, NATS Streaming and NATS JetStream | by George Koulouris | Medium](https://gcoolinfo.medium.com/comparing-nats-nats-streaming-and-nats-jetstream-ec2d9f426dc8)

## Appendix

Golang sample code:

```golang
package main

import (
	"context"
	"fmt"
	"log"
	"math/rand"
	"os"
	"testing"
	"time"

	"github.com/nats-io/jsm.go"
	"github.com/nats-io/jsm.go/api"
	"github.com/nats-io/nats.go"
)

var fullSubject string = "report_task.scheduled"
var wildcardSubject string = "report_task.*"

func consumeOne(doneChan chan bool) {
	msg, err := consumer.NextMsg()
	if err != nil {
		fmt.Printf("Fail to get message: %v\n", err)
	}
	fmt.Printf("Consume get task: %s\n", string(msg.Data))
	time.Sleep(2 * time.Second)
	if err := msg.Ack(); err != nil {
		fmt.Printf("Fail to ack the message %s: %v\n", string(msg.Data), err)
	}
	doneChan <- true
}

func TestProduceAndConsume(t *testing.T) {
	producerStopChan := make(chan bool)
	consumerStopChan := make(chan bool)
	var taskCount int = 1

	go func() {
		for idx := 0; idx < taskCount; idx++ {
			taskName := fmt.Sprintf("task #%d", rand.Int())
			nc.Publish(fullSubject, []byte(taskName))

			fmt.Printf("Producer produce: %s\n", taskName)
		}
		producerStopChan <- true
	}()

	for idx := 0; idx < taskCount; idx++ {
		go func() {
			consumeOne(consumerStopChan)
		}()
	}

	<-producerStopChan
	for idx := 0; idx < taskCount; idx++ {
		<-consumerStopChan
	}

	fmt.Println("Done")
}

var ctx context.Context
var cancel context.CancelFunc
var nc *nats.Conn
var stream *jsm.Stream
var consumer *jsm.Consumer

func setup() {
	var err error

	ctx, cancel = context.WithTimeout(context.Background(), 10*time.Second)
	nc, err = nats.Connect(nats.DefaultURL, nats.UserInfo("name", "password"), nats.UseOldRequestStyle())
	if err != nil {
		log.Fatal(err)
	}

	jsmgr, err := jsm.New(nc)
	if err != nil {
		log.Fatal(err)
	}

	streamName := "ReportTask"
	stream, err = jsmgr.LoadOrNewStreamFromDefault(streamName,
		api.StreamConfig{
			Subjects:     []string{wildcardSubject},
			Storage:      api.FileStorage,
			Retention:    api.LimitsPolicy,
			Discard:      api.DiscardOld,
			MaxConsumers: -1,
			MaxMsgs:      -1,
			MaxBytes:     -1,
			MaxAge:       24 * time.Hour,
			MaxMsgSize:   -1,
			Replicas:     1,
			NoAck:        false,
		})
	if err != nil {
		log.Fatal(err)
	}

	consumerName := "Generator"
	consumer, err = stream.LoadOrNewConsumerFromDefault(consumerName,
		api.ConsumerConfig{
			Durable:         consumerName,
			DeliverPolicy:   api.DeliverNew,
			FilterSubject:   fullSubject,
			AckPolicy:       api.AckExplicit,
			AckWait:         30 * time.Second,
			MaxDeliver:      5,
			ReplayPolicy:    api.ReplayInstant,
			SampleFrequency: "0%",
		})
	if err != nil {
		log.Fatal(err)
	}

}

func shutdown() {
	cancel()
}

func TestMain(m *testing.M) {
	setup()
	code := m.Run()
	shutdown()
	os.Exit(code)
}
```
