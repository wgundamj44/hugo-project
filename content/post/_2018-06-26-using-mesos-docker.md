---
title: "Project running on DCOS"
categories: ["tech"]
date: 2018-06-26
tags: ["docker", "docs"]
---

# VIP
VIP (virtual address) maps traffic from a single virtual address to multiple IP address and ports. DCOS uses VIP to make service to service communication simple, that is, a service can
connect to another service by a well defined name instead of ip address.

`servicePort` for service discovery. For instance, a service called `test` has three instances, `10.0.0.2:31000`, `10.0.0.2:31001`, `10.0.0.2:31002`.
For A client to access this service, we need a single port say `8080` to which client can access, then route the request to one of the instances. This `8080` is service port.
Marathon it self does not use servicePort, it only forwards it to load balancer, and load balancer will use it to accept connections.

Purpose of `VIP_0` is informational, real usage is implmenation dependent. In DCOS, it's used to generate VIP name and port. Eg. for `'VIP_0': '/test:10000'`, a VIP
`test.marathon.l4lb.thisdcos.directory:10000` will be generated.

# health check

# HAProxy

# Questions:
- networks in DCOS, how the tunnel from dcos to cart service is done?
- how to start a new backend service project? template? what happen if template changes, need to regenerate all the projects?
- concourse is too slow?
- how is kafaka used in coati? connector? sink?
