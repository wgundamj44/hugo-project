---
title: "How celery works(1)"
categories: ["tech"]
date: 2017-03-09
tags: ["celery", "kombu", "python"]
---
Kombu is a messaging library for python. It can support messaging with AMPQ protocal. Celery uses many functionalities of Kombu.

# From Kombu to Celery

## Register a task
Kombu is a publisher-conumer architecture. It publishes some messages to queues, and some consumer will fetch these messages from queue, and 
execute some logics. While celery is a asyncrounous task library. It registers some task, and has the ability to call the task asyncrounously.
They seem to do totally diffrent things, how they work togother?

When we register a task to celery, we do something like:
```python
app = Celery()
@app.task
def test():
    pass
```
So the magics happends in the decorator `task`. Look at the implementation of `task`, it basically creates an instance of `celery.app.task:Task` class. This class when created, will save itself to the `app._task` dict.
We can easily verify this by inspecting `app.tasks` which returns `_task`. From this dict, we can see all the tasks that has been registered.

## apply_async
This method make the asyncrounous call. Inside this method, it calls actually publish a message. By default, it will publish to message to a queue called 'celery', with exchange 'celery' and routing_key 'celery', the exchange type is 'direct'. All the entities used here like 'queue',
'producer' etc. are just extensions of the original kombu entity. When publishing message, the body includes task name, task id, parameters, links, etc. So roughly, `apply_async` equals to publish message.

## consumer
As we publish message with `apply_async`, the worker should have consumers that subscribed to this message. 
Celery worker has a `consumer` component, this component is further consisted of several other components: `Evloop`, `Tasks`, etc. Here the 
`Tasks` component will register all the tasks to worker, and create the task consumer. `Evloop` starts a event loop, make the consumer consumes
from queues. The consumer's callback will be `on_task_received` method. This method has the signature `def on_task_received(body, message)`.
We can see it's just a normal kombu consumer callback. Inside this method, it will call the registered task to process the message. This is how 
our task is called finally.

## from test to task
Look inside of `on_task_received` method, we can see it calls the task with 
```python
strategies[name](message, body, message.ack_log_error, message.reject_log_error, callbacks)
```
The signature of the task is obviously different from our original function eg. `def test()`. So the Celery must have converted our function to the final task.
The `strategies` here is dict of 'Strategy'. By default, each Strategy is an instance of `celery.worker.strategy:default` class. Note that, `stratigies` is different from `tasks` we mentioned before. `tasks` belongs to celery app itself, and contains `celery.app.task.Task` instances. While `stragties` belongs to consumer, and it is created from `tasks` by calling `start_strategy` method of each task in `tasks`.

When we look at `task_message_handler`, we find that, it is actually in the form of 
```python
def task_message_handler(message, body, ack, reject, callbacks, to_timestamp=to_timestamp)
```
Exactly matches the how strategy is called. So where is our `test` goes?
It is wrapped inside of a `celery.worker.job::Request` object. The `Request` obj deals with lots of things like making the function execute in 
a process pool, tracing the execution, etc. Anyway, if trace all the way down this `Request`, in the `trace.py` 's `build_tracer` method, 
we will find `retval = fun(*args, **kwargs)`. Where the `fun` is our lovely `test`.

## simply put
So here is how celery works, in our main application, we use `app.task` to register tasks. When we call task asyncrounously, we publish message
containing the task name and other information. In celery worker, its consumer consume from the same queues as the main application. When message arrives, it just find the correct stratigy with task's name, and calls it.

