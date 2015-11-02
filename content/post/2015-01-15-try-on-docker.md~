---
layout: post
title: A try on docker
category: [tech]
tags: [docker]
---
Recently(Last year..) I tried docker a little bit. My Goal was to create my deveopment enviroment of MEAN.js.

# Failure..
At first I installed docker on my CentOS 6.5, and miserably failed. It kept telling me that my linux kernal version is not right,
althoug i've already updated it to latest version.

So I had to install a new VM with CentOS 7.0

# Pull the images
Here is what I need: An Image with nodejs and an Image with MongoDB. I downloaded them from offical repository so there's nothing to say.

# Configuration
This is the key part.

First, I need to share the source code between host and docker host. I found that -v option can specify folder sharing.

Second, I don't what the data of mongo be left inside docker host. Again, -v can make /data/db of mongo be a folder in real host.

Third, the docker process of nodejs should be able to see mongo process. --link can solve this. To use link, docker of mongo should have a name first.

Forth, in MEAN.js, it defaults to access localhost for mongo, which doesn't exist here. So I need to modify it so that it accepts DB_HOST enviroment variable as its mongo host, and --env option of docker can set evrioment variables.

# Conclusion
The above consideratin sums up to be the commands as follows:

To start mongodb

```
docker run -it -d -p 27017:27017 -v /develop/mongo_data/:/data/db --name mongodb dockerfile/mongodb mongod
```

To start mean.js

```
docker run -it --rm -p 8081:8081 -v /develop/node/testProject/:/home/meanjs --link mongodb:mongo --name meanjs --env DB_HOST=mongo dockerfile/meanjs forever -w server.js
```

