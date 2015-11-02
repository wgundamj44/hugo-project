---
layout: post
title: Memo about HTTP service in python
category: [tech]
tags: [python]
---

## General idea of WSGI

As specified by pep-0333, the WSGI aims to make application written in python run on any web server as long as they follow the WSGI.

It includes two parts:

#### The Application/Framework Side

The application object should be callable(a function, a class with __call__ ..), and accepts two arguments. Like:
{% highlight python %}
def simple_app(environ, start_response):
    """Simplest possible application object"""
    status = '200 OK'
    response_headers = [('Content-type', 'text/plain')]
    start_response(status, response_headers)
    return ['Hello world!\n']
{% endhighlight %}

#### The Server/Gateway Side

The server will invoke the app for each request. For example, in werkzeug's BaseWSGIServer, it invokes app with ```app(environ, start_response)```.


## werkzeug

### run_simple: make a simple HTTP server

The interesting part is it supports restart on code change. The mechanism is that, when script executes,
it starts a new process that executes a new process running the same script, but add a new envrioment variable
```WERKZEUG_RUN_MAIN```, so that a process can tell if it is running in forked process or original process, and executes
different logic accordingly.

The main process is a while(1) loop, it forks a subprocess, waits until it terminates, checks its status code. If the code is 3,
it knows code has changed and forks subprocess again, so that server is restarted. If code is not 3, then some error may happended,
the main process itself will terminates.

In subprocess, it just starts a new thread which
runs the server instance, and then runs the reloader. In reloader, it use watchdog to detect file add/remove/modification infinitely, if
any change happens, it breaks from loop and subprocess terminates with status code 3. Notice that, the server thread is started with daemon
options on, so when parent process terminates, the thread terminates too.

### BaseWSGIServer and WSGIRequestHandler

`BaseWSGIServer` inherits Python's `HTTPServer`. It sets its `handler` to `WSGIRequestHandler` which will handle request. For each request, this `WSGIRequestHandler` will be initialized, and handles request.

In `WSGIRequestHandler`'s base class BaseRequestHandler, we can see in its contructor, it calls step(), handle(), finish() in turn. All its child class
will override handle() to give response.

In WSGIRequestHandler's parent class BaseHTTPRequestHandler, it overrides handle() and call handle_one_request() for each request came. And WSGIRequestHandler
overrides handle_one_request() which finally calls run_wsgi(). And in run_wsgi, the app which is the server logic written by user is called with
```app(envrion, start_response)```. envrion is envrioment variables from request. start_response is a function returns write method. write method is where response data
is actually get written. ```start_response``` just set the response_headers and status, and return a function which will finally write response into socket.

So where does this envrion come from? In WSGIRequestHandler there's a make_environ method that copies the header info from request into envrion.

### routing

Routing is implemented with werkzeug.routing. As a general view, routing can be used as:
{% highlight python %}
m = Map([Rule('/', endpoint='hello')], default_subdomain='www')
m.add(Subdomain('kb',[Rule('/', endpoint='hello2')])
m.add(route.Subdomain('kb',[Rule('/test', endpoint='hello2.test')]))
{% endhighlight %}

Now the Map object has the internal data like:
{% highlight python %}
Map([<Rule 'kb|/test' -> hello2.test>,
 <Rule 'www|/' -> hello>,
 <Rule 'kb|/' -> hello2>])
{% endhighlight %}

Rule reprents a single mapping rule, it becomes usefule after being ```bind``` to a map. In bind method,
the regex pattern will be generated. For example, a rule ```r = Rule('/add', endpoint='doAdd')``` after bind to m,
```print(r._regex.pattern)``` has the result ```^www\|\/add$```.

Map should be bind to a host name with ```ad = m.bind('example.com')```, which returns a MapAdapter. MapAdapter is in charge
for URL matching. Note that for the above ```ad``` object, it only matches url with subdomain ```www```. To match url with subdomain
```kb```, it should be bind with ```m.bind('example.com', subdomain='kb')```.

The url dispatch is also done by MapAdapter, with ```dispatch``` method.


## flask

flask looks like a showbox of how to use werkzeug. For a code snippet like:

{% highlight python %}
from flask import Flask
app = Flask(__name__)

@app.route("/say/<name>")
def say(name):
    return "hello %s" % name


if __name__ == '__main__':
    app.run(host="localhost", port=5000, debug=True)
    
{% endhighlight %}

@app.route will register a routing entry, with werkzeug's Routing module. For the above example, it will route ```'/say/xxx'``` url to ```say(name)``` function.
