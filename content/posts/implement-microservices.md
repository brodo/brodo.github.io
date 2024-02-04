+++
title = "Implement Microservices Now"
date = "2024-02-04T00:35:30+02:00"
author = "Julian Dax"
authorTwitter = "" #do not include @
cover = ""
tags = ["Microservices", "Scaling", "Satire"]
keywords = ["Microservices", "Scaling", "Architecture"]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

## Implement Microservices Now

95% of backends deal with less than 100 requests per second. 80% of what they do is CRUD. So, let's build distributed
systems.

## You Need Horizontal Scaling

[SQLite can do millions of inserts per second.](https://avi.im/blag/2021/fast-sqlite-inserts/) Popular Javascript (yes,
Javascript) frameworks can handle [at least 10,000 requests per second](https://fastify.dev/). That's why we need horizontal scaling via microservices.

## More Complexity Is Just More Fun

Have you ever been bored at work working with the same old technology? Is it just a stupid server talking to a database and rendering HTML or JSON? You are missing out on lots of microservice fun! Some of your older colleagues might doubt the whole microservice trend, but that's all just fear of new technologies. They will come up with all sorts of ifs and buts, so here are some things that won't be a problem:

1.  **Cache invalidation:** This classic problem becomes easy in distributed systems. Just send messages!
2.  **Fault tolerance**: Your 50 microservices will always be up and respond quickly.
3.  **Monitoring:** You can't debug your application locally anymore; therefore, you will add Open Telemetry and Prometheus to your services. You will run Grafana, Influx, Clickhouse, MinIO, Kibana, and ElasticSearch to store and view your traces, metrics, and logs. As a rule, have at least one k8s pod running in your monitoring namespace per microservice you operate.
4.  **Serializing and Deserializing JSON:** By turning every function call into an HTTP query, your CPUs finally have something valuable to do: serializing and deserializing JSON. It's a fun pastime, occupying at least 60% of their cycles.
5.  **Validating JSON:** Do you like writing JSON schemas? Me too! The validation will also take another 10% CPU time.
6.  **Reimplementing SQL:** As any good developer knows, SQL does not scale. **We can fix this by adding an ad hoc, informally-specified, bug-ridden, slow implementation of half of SQL to all of our services.** We'll keep these in sync using Kafka!
7.  **"Scaling":** Scaling means how many services you can run in parallel, **not** how much throughput you have. In the unlikely case that you have a lot of users (or do load tests), everything will go just as planned.
8.  **Service granularity**: The answer to "how big should my service be" is always smaller. There is no overhead in having additional services. Maintaining invariants in data shared between several services is a breeze.
9.  **Finding the right people**: Hiring people good at distributed systems is straightforward. Programmers can also finally fulfill their dream of becoming infrastructure experts.
10.  **Money**: You can just rent a few k8s clusters from AWS for the low price of a single-family home per month. In the unlikely case that our company needs to save money on that, remember that microservices are equally great at scaling down as they are at scaling up.
11.  **The Environment**: As software developers, we don't go outside anyway.

If your colleagues are still against implementing microservices, remind them about all the new career opportunities that
will open up for them. CV-driven development always results in the best products. And lastly, doing this will create jobs! A problem requiring 50 microservices must be complex, so management will hire more people. The people already with the company might even get one of the new tech lead positions.

Are your colleagues still not convinced? Then it's time for action: introduce Clean Code. Those beautiful 
AbstractProxyFactoryFacories will ensure that your server will handle ten requests per second maximum, and then you'll
**need** horizontal scaling. All your super complicated business logic (that you could express as database constraints)
will be wonderfully isolated from unimportant details. You get extra points if you use your favorite ORM and Dependency
Injection container in all your new microservices. This way, you can combine the majestic slowness of Clean Code with
the heart- and CUP-warming compute requirements of microservices. Also, write your own "SDK" that everyone has to use.
Doing this will keep the teams moving fast, as everyone depends on the SDK team for everything.
