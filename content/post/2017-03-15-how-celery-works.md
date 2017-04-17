---
title: "How celery works(2)"
categories: ["tech"]
date: 2017-03-15
tags: ["celery", "kombu", "python"]
---

# Blueprint
The consumer of Celery is organized with Blurpint. This `Blurprint`, together with `Step` defines the dependency relationship between all the
components of consumer. Each component may have its own blueprint. This will form a directed graph.

For instance, `Worker` depends on `Hub`, `Pool`, `Beat`, `Timer`, `StateDB`, `Consumer` and `WorkerComponent`. `Consumer` component also has its
own bluprint and dependency.

When start, celery worker will sort all the dependencies topilogically, and start them one by one. Each component aslo holds a reference to its
parent, so when component is created, it can be (and often be) attached to its parent.

The logic works as follows:

- in blueprint class, the call to apply will invoke `include` method of each step

```
def apply(self, parent, **kwargs)
   ....
   for step in order:
       step.include(parent)
```

- in step, if the step can be included, then the call to its `create` method will potentially(depend on implmentation) attach itself to parent
  for instance, `Hub` component's `create` looks like this:

```
def create(self, w):
   """
   w is the parent of this blueprint
   """
   w.hub = get_event_loop()
```

The responsibilities of each component are:

- Hub: for event loop
- Pool: for multi process(thread, greenlet, ..)
- Beat: for beat service, controls scheduled tasks
- Timer: as the name
- StateDB: for saveing worker state
- Consumer: for actually consume message from queue
- WorkerComponent: for autoscale

## Beat, the scheduling service
This component is for scheduling crontab like tasks. We define task schedule with:
```
app.conf.beat_schedule = {
    # Executes every Monday morning at 7:30 a.m.
    'add-every-monday-morning': {
        'task': 'tasks.add',
        'schedule': crontab(hour=7, minute=30, day_of_week=1),
        'args': (16, 16),
    },
}
```
This tells celery to run `tasks.add` task on first day of every week at 7:3jjj0. 

So how to do such thing with program. On the high level, there shall be another thread/process that is a while loop. In this loop, the
thread/process checks all the available scheduled tasks periodically, and see if there's a task should be executed, if so it will send the
task just as we call task manually with `async_apply`.

In the celery's implemenation, it involves `Beat`, `Service`, `Scheduler` and `ScheduleEntry`.

The `Service` is the while loop which resides off the main process. In `Service` 's loop, it calles `Scheduler` 's `tick` method to fetch a 
most recent coming task and the time until the task should be called. Then it sleeps for that time long, and start loop again. The actual 
works are done in `tick`.

The `Scheduler` holds a priority queue, this queue sort all the tasks by 1. remaining peirod to be run, 2. priority. `tick` will fetch the top
entry from the queue, if the entry is due to run now, send the task to message queue. If the entry should run later, then return the remaining
period to the `Service`, so that `Service` knows how long to sleep(wait).

The `ScheduleEntry` is the entry holded in the priority queue. It holds task and its schedule, so it knows wether the task should is due or not.

## Hub, consume rabbitmq messages with event loop
Hub is an event loop. In Celery, it has many purposes like supporting asyncrounous consuming of messages of queue, managing events of pooled
processes, etc.

When celery wants to consume from queues, the `transport`(pyamqp by default)'s `register_with_event_loop` is called.  This makes a callback
is called everytime thers's message available from queue.


# Celery Extension
Recent celery provides some way to customize consumers ourself. As mentioned in the previous post, the consumer of celery is a wrapper of the
`app.task` decorated function. This makes celery an asynchounous task handler. But in our project, we sometimes don't want to write the handler
code in the main project. We may only want to publish some message when something happens, and let other project to consume the message. The is 
actually the original pub-sub mechanism.

In order to do this thing, we use the kombu's `publish` method to publish message, instead of using `delay` of celery. And in the worker, we
need to provide the celery with our own consumer. Celery documents give us the example: `app.steps[consumer].append(MyConsumer)`. This
MyConsumer should extends Celery's `ConsumerStep`. And it looks very similiar to kombu's consumer. With this code, instead of loading celery's 
default steps, it will load `MyConsumer` instead. By doing this, we can easily make use of the existing worker management of celery while
provide our own logics.


# Celery Worker
Celery can make use of process pools to achieve concurrency. The archtecture is, one main process and several worker processes. The main process
is the one that has consumer, subscribes to queues, the worker process just receive tasks dispatched from main process and do the task.

Celery doesn't use standard `multiprocessing.pools`, instead it maintains its own implmentation called `billiard` which does similar things. The
way how multiple process are cooperated with each other is beyond my understandings. I will look into it later.

# Misc

## rate limit
In celery's consumer, there's a rate limit mechanism. Each task can specify its rate limit with `rate_limit`, with this parameter specified as
n, the stask can be executed n times every seconds.

The algorithm to implement this is called `token bucket`. It is implemented in `kombu.utils.limits`. The main idea of the algorithm is: There's
a bucket with capacity `c`, and every seconds it will accumulate `r` tokens. For a task to execute, it will consume `t` tokens. If there's
not enough tokens in the bucket, the task will be postoned until enough tokens are accumulated.
