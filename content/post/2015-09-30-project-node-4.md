---
date: 2015-09-30
title: Some project notes(4)
categories: [tech]
tags: [Django]
---
## boostrap of django

### Class Apps
Everything begins from django.setup(). In this method, it will import django.apps, and as a side effect, `apps` is created.

`apps` is an instance of django.registry.Apps. This class holds several important fields, `all_models` which hold all the mapping of from app labels to model. `app_configs` is all the configurations of app in django. When bootstrapping, `apps.populate` will be called passing all the apps listed in settings.INSTALLED_APPS. It traverse the apps one by one, and create an `AppConfig` for each app, and insert it to `app_configs`. When bootstrapped finished, it will call `ready` of each app. So, if we what some work to be done after app started, we can do it in ready method of `AppConfig`. 

`apps` has a `register_model` method, which will insert a model to `all_models` of apps. The method is called at `ModelBase` of django.db.models.base. We know that for creation of every
model class, `ModelBase` will be called, and inside its `__new__` method, it will call register_model to register itself.

In `populate`, for each `AppConfig`, it also use `import_models` to load models. But here is the misleading part, in the call `import_models(all_models)`, all_models is actually None. In `import_models`, it will check if this app has a model module, if so it will import the model. When the model is imported, every model class in model module will be created, and `ModelBase` will be called. As mentioned above, `all_models` will be filled in `ModelBase`.

### Class AppConfig
This class hold the informations of an app, eg. name, label, models, etc. AppConfig has a factory method to create itself. When we insert an entry in settings.INSTALLED_APPS, if the
entry is moudle name, then `create` will try to import it. If the imported module has a `default_app_config` attribute which is a subclass of AppConfig, then it will be imported, and become the runtime class of `AppConfig` instance. Otherwise, the default implmentation of AppConfig will be used.

### Wrap up
When jdango finishes bootstrap, its `django.apps` will contains a `app_configs` containing the config infomations for each installed_apps, and `all_models` containing all the models.

## From request to view
We start the story from `django.core.handlers.wsgi.WSGIHandler`, from experience of werkzeug, we know that this handler is for the handling of request. And it has our old friend `__call__(self, environ, start_response)` -- a standard WSGI app function, as mentioned in [this post]({{< ref "2015-05-21-http-service-in-python.md" >}}). In this function, it does two things: generate a request object, get a response object from request.


### Generate a request
The code `request = self.request_class(environ)` generates a reqeust object, where request_class is `WSGIRequest`. The request object holds lots of information about this coming request, eg. its method, its path, etc. This request object is what we got in request paremter of `view_handler(request, xxx)` in views.


### Get a response
`BaseHandler.get_response(self, request)` is where response is generated. It will first go through each request middleware, and try to make a response. Global operation like CSRF prevention happens in this stage. If none of the request middleware made a response, then the request will be matched with URLResovler, and try to find a view method that handles the request. This is were our URL router and view functions get used. If any exception happens, exception middleware will take control and try to make response. Then if response has `render` method, template response middleware will get in and render a new response. Finally response middleware proccesses the response and make final response.
