+++
date = "2020-11-16T20:00:00+05:00"
draft = false
title = "Scheduling Messages Over a Channel with gosd"
slug = "schedule-message-gosd"
keywords = ["go"]
tags = ["go","messaging","scheduling"]
image = ""
comments = false	# set false to hide Disqus
share = false	# set false to hide share buttons
menu= ""		# set "main" to add this content to the main menu
author = ""
+++

Over the summer of 2020 I spent some time working on a [small utility library](https://github.com/alexsniffin/gosd) in Go to schedule the dispatch of a message. This problem came up when I was working on a new stream processing system and needed to add a small dynamic delay to certain flows.

#### The Problem
My first implementation was a simple goroutine with a timer. Something similar to this:
```go
go func(outCh chan T, msg T) {
    time.Sleep(...)
    outCh <- msg
}(ch, msg)
```

This solution can work for some simpler systems but doesn't scale well. Some questions I asked myself when facing this problem:
 - What happens if more messages than expected pass through this flow?
 - What happens if certain messages require a delay that lasts for a long period?
 - What happens during cancellation or shutdown?

These are some concerns that I had, especially with my situation and having the possibility of having bursts of millions of messages. I needed a better solution to this problem.

Some more context to my constraints; performance is less important for this flow than others, it should be able to process bursts of messages, it needs a way to shutdown, and it should limit the amount of goroutines.

#### The Solution

One solution I thought could work would be to create a pool of goroutines which can provide a bounded amount of messages which can be processed at any given time. Although, like the first solution, this doesn't scale. When all goroutines are busy waiting, new messages will be blocked and unable to process. Their initial delay could be surpassed, and they will fall behind. You could add some logic to swap out messages with delays that are sooner than the soonest message being processed but this got complex quickly. Also, why wait for any message other than the soonest?

![light](https://media.giphy.com/media/4jHXZ9aIKFaUM/giphy.gif#center)

Aha, this is a perfect use-case for a priority queue as it only needs to wait on the soonest message! Go provides an interface for a [heap](https://en.wikipedia.org/wiki/Heap_(data_structure)) in the [standard library](https://golang.org/pkg/container/heap/) which can be used as the underlying data structure for the priority queue.

This can work. Use the priority queue as the mechanism for keeping messages sorted based on the schedule and wrap it with an interface for processing the messages.

![gosd](/images/gosd.png#center)

With this implementation only two goroutines are needed. 
- One which concurrently pushes and pops messages from the priority queue, handles cancellations, and handles the soonest message to wait on. 
- One which waits based on the scheduled time. 

If a new message is processed with a sooner schedule, a signal to stop the soonest message will occur and it will be pushed back into the priority queue, then the new message will be waited on. The amount of messages to store in the heap and channels are all configurable and can be bounded to a finite limit. If a shutdown occurs, messages can be flushed out of the heap. I also added states for pausing and resuming but I will likely remove this in a 2.0.0 release to keep the library simple as possible.

#### Example
```go
// create instance of dispatcher
dispatcher, err := gosd.NewDispatcher(&gosd.DispatcherConfig{
    IngressChannelSize:  100,
    DispatchChannelSize: 100,
    MaxMessages:         100,
    GuaranteeOrder:      false,
})
checkErr(err)

// spawn process
go dispatcher.Start()

// schedule a message
dispatcher.IngressChannel() <- &gosd.ScheduledMessage{
    At:      time.Now().Add(1 * time.Second),
    Message: "Hello World in 1 second!",
}

// wait for the message
msg := <-dispatcher.DispatchChannel()

// type assert
msgStr := msg.(string)
fmt.Println(msgStr)
// Hello World in 1 second!

// shutdown without deadline
dispatcher.Shutdown(context.Background(), false)
```

That's it! More examples can be found on the [github](https://github.com/alexsniffin/gosd/tree/master/examples)! 