---
layout: post
title: "Pubsub with asyncio"
description: "Simple pubsub solution using asyncio for different destinations"
categories: asyncio python
---

Pubsub is one of the most recognizable patterns in computer science, employed
as a simple solution to a wide range of problems, with different
implementations. <!--more-->As an example, let's suppose you need to dispatch
some data with different computation times to different destinations, sure the
first thing coming to mind is a broker or a simple job queue, and there's a
plethora of battle-tested solutions out there, from `Apache Kafka` to
`RabbitMQ` to even `Redis` or the solid AWS `SNS/SQS` combination, where topics
can be defined on `SNS` with Lambdas or `SQS` queuing triggered at each
message received. Why? Because why not, it gives solid performances and near
unlimited scalability for just some bucks per month, with low to none
maintenance costs beside your business logic; something to consider if you
already have a part or your entire backend hosted on AWS.

Sometimes though it can be an overkill or all we need is just a prototype, a
proof of concept, in those cases a simple micro-service can do well enough with
little costs.

### Asyncio pubsub

So in our case we'll face a classic pubsub problem, with a bunch of data
sources that have to be delivered to multiple consumers. The first solution
that comes to mind attacks the problem by using a multi-threaded pattern, by
running as many threads as the number of sources, each one with a loop
synchronized with timers to dispatch results.

Asyncio offers two different approaches to this by using a single thread,
making it much simpler in terms of synchronization and flow of the application,
leveraging cooperative concurrency with `coroutines` or using a synchronous
callback-based system.

#### Cooperative concurrency, coroutine solution

Starting with the `coroutines` approach, we define a simple `AsyncDeliver` class
which will act as out `publisher` component, with 3 different sources, running
respectively once every 5, 10 and 20 seconds; we'll need 3 different topics for
these tasks, for simplicity we'll call them, `5sec-delivery`, `10sec-delivery`
and `20sec-delivery` respectively.

Let's move one with the implementation.

<hr>
```python
import asyncio
import collections

class AsyncDeliver:

    """Publisher component, deliver data from different sources to subscribed
    consumers. This version uses a defaultdict with list as main values to
    track subscribers (our consumers) for each topic.
    Subscriber objects are expected to implement a simple interface with just
    an update method for now.
    """

    def __init__(self):
        self.subscribers = collections.defaultdict(list)

    def subscribe(self, subscriber, topic):
        self.subscribers[topic].append(subscriber)

    def deliver(self, data, topic):
        """Call update method of each subscriber for the topic. As long as
        the calls are non-blocking there's no need to make the asynchronous, but
        nothing would've stopped us to do so if needed.
        """
        if topic not in self.subscribers:
            return
        for subscriber in self.subscribers[topic]:
            subscriber.update(data)

    async def run(self):
        """Main entry-point, here we instantiate all the tasks, 5 and 10 seconds
        delivery, using the 20 seconds as our "blocking" call.
        A shutdown method would be called on application stop to cancel all
        running tasks and gracefully close the event loop.
        """
        loop = asyncio.get_running_loop()
        loop.create_task(self.deliver_every_5s())
        loop.create_task(self.deliver_every_10s())
        try:
            await self.deliver_every_20s()
        except KeyboardInterrupt:
            await self.shutdown()

    async def shutdown(self):
        loop = asyncio.get_running_loop()
        tasks = [task for task in asyncio.Task.all_tasks() if task is not
                 asyncio.tasks.Task.current_task()]
        for task in tasks:
            task.cancel()
        await asyncio.gather(*tasks, return_exceptions=True)
        loop.stop()
        loop.close()

    async def deliver_every_5s(self):
        while True:
            try:
                await asyncio.sleep(5)
                self.deliver("5 seconds delivery", "5sec-delivery")
            except asyncio.CancelledError:
                break

    async def deliver_every_10s(self):
        while True:
            try:
                await asyncio.sleep(10)
                self.deliver("10 seconds delivery", "10sec-delivery")
            except asyncio.CancelledError:
                break

    async def deliver_every_20s(self):
        while True:
            try:
                await asyncio.sleep(20)
                self.deliver("20 seconds delivery", "20sec-delivery")
            except asyncio.CancelledError:
                break
```
<hr>

Let's define a simple subscriber interface, it will only need an `update`
method. Sub-classes can easily define some logic or different communication
types such as HTTP clients, Kinesis/Kafka producers.

To be noted that for the sake of the example, all `deliver` methods doesn't
block in any way, but it would be trivial to declare `async` even the deliver
method and await on all subscribers on the `AsyncDeliver` class by calling
`asyncio.gather` coroutine.

<hr>
```python
import abc

class Subscriber(abc.ABC):

    def __init__(self, name):
        self.name = name

    @abc.abstractmethod
    def update(self, data):
        raise NotImplementedError()

class NullSubscriber(Subscriber):

    """Simple subscriber that does nothing but printing the updates it
    receives
    """

    def update(self, data):
        """We expect this method to do non-blocking operations, if something have
        to do with I/O in a blocking manner, we'd better re-define it as a
        coroutine method which can be awaited.
        """
        print(f"[{self.name}] Received data: {data}")

```
<hr>

And finally let's smoke-test it by running a simple scenario, 3 consumers
`Toki`, `Shu` and `Raoh` (names have clearly no interlacing on one another)

<hr>
```python
if __name__ == '__main__':
    deliverboy = AsyncDeliver()
    subscriber_one = NullSubscriber('Toki')
    subscriber_two = NullSubscriber('Shu')
    subscriber_three = NullSubscriber('Raoh')
    deliverboy.subscribe(subscriber_one, "5sec-delivery")
    deliverboy.subscribe(subscriber_two, "10sec-delivery")
    deliverboy.subscribe(subscriber_three, "20sec-delivery")
    try:
        asyncio.run(deliverboy.run())
    except KeyboardInterrupt:
        pass
```
<hr>

#### Callback based approach

In python, `asyncio` implementation support either coroutines as well as
callbacks, this was borrowed from the `Twisted` framework which predates the
`asyncio` coming, and by no surprise, built-in APIs for TCP/UDP communication
offer both patterns, dividing them in [low-level
APIs](https://docs.python.org/3/library/asyncio-protocol.html) and [high-level
APIs](https://docs.python.org/3/library/asyncio-stream.html), the first using
a callback approach and the latter, called streams, using a wrapper around
those low-level callbacks which makes them simple to use as coroutines.

<hr>
```python
import asyncio
import collections

class CallbackDeliver:

    """Callback based implementation, subscriber interface retain the
    implementation, this time we'll use the `AbstractEventLoop.call_later`
    synchronous call to schedule again the run of a callable callback.
    """

    def __init__(self):
        self.subscribers = collections.defaultdict(list)
        self.event = None

    def subscribe(self, subscriber, topic):
        self.subscribers[topic].append(subscriber)

    def deliver(self, data, topic):
        if topic not in self.subscribers:
            return
        for subscriber in self.subscribers[topic]:
            subscriber.update(data)

    async def run(self):
        loop = asyncio.get_running_loop()
        self.event = asyncio.Event(loop=loop)
        loop.call_later(5, self.deliver_every_5s)
        loop.call_later(10, self.deliver_every_10s)
        loop.call_later(20, self.deliver_every_20s)
        await self.event.wait()

    def shutdown(self):
        self.event.set()

    def deliver_every_5s(self):
        self.deliver("5 seconds delivery", "5sec-delivery")
        asyncio.get_running_loop().call_later(5, self.deliver_every_5s)

    def deliver_every_10s(self):
        self.deliver("10 seconds delivery", "10sec-delivery")
        asyncio.get_running_loop().call_later(10, self.deliver_every_10s)

    def deliver_every_20s(self):
        self.deliver("20 seconds delivery", "20sec-delivery")
        asyncio.get_running_loop().call_later(20, self.deliver_every_20s)

```
<hr>

To simplify a bit more it's pretty straight to move some code inside a
decorator to make a method a looping callback.

<hr>
```python
import asyncio
import functools

def run_every(seconds):
    def inner_func(func):
        @functools.wraps(func)
        def func_wrapper(self, *args, **kwargs):
            loop = asyncio.get_running_loop()
            func(self, *args, **kwargs)
            loop.call_later(seconds, func_wrapper, self, *args, **kwargs)
        return func_wrapper
    return inner_func

```
<hr>

This allow us to write our delivery methods with no need to retrieve the
running event loop, a detail in the decorator definition that should be taken
in consideration is that in the `AbstractEventLoop.call_later` call we pass in
as callable argument the `func_wrapper` defined inside the `inner_func` and not
only the `func` like we would've done normally. This for the simple reason that
we need to re-apply the decorator at each scheduling, if we passed just the
`func` the loop would've just run the next iteration of the callable without
re-scheduling it again.

<hr>
```python
@run_every(5)
def deliver_every_5s(self):
    self.deliver("5 seconds delivery", "5sec-delivery")
```

Again we can run a simple main, that should result the same as the previous
coroutine-based implementation

```python
if __name__ == '__main__':
    deliverboy = CallbackDeliver()
    subscriber_one = NullSubscriber('Toki')
    subscriber_two = NullSubscriber('Shu')
    subscriber_three = NullSubscriber('Raoh')
    deliverboy.subscribe(subscriber_one, "5sec-delivery")
    deliverboy.subscribe(subscriber_two, "10sec-delivery")
    deliverboy.subscribe(subscriber_three, "20sec-delivery")
    try:
        asyncio.run(deliverboy.run())
    except KeyboardInterrupt:
        deliverboy.shutdown()
```
<hr>

And that's it, these two snippets should be a good comparison of the two
methods, overall, as we can see, there are not many differences between them,
both systems accomplish to the results we expected, callback approach is probably
neater and require a bit less code to be written, but it assumes that `deliver`
method doesn't block in any situation, so for the majority of cases it expects
some sort of buffering or queues to be involved. The coroutine-based instead
can be easily edited to make asynchronous even the `deliver` method in order to
not block the loop at any time.
