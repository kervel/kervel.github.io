---
layout: post
title:  "UDP loadbalancer on amazon EKS"
date:   2021-11-18 20:45:21 +0200
categories: kubernetes
---

I spent some time figuring out how to get an UDP loadbalancer service working on AWS EKS, maybe this document could save some time for somebody in the future.
UDP loadbalancers are now supported by the EKS network load balancers, but it is still a bit tricky to set them up:

* UDP loadbalancers also need health checks to work but UDP health checks don't really exist.
* UDP loadbalancers require some special settings that took me a while figuring out.

# Creating a health check for the UDP load balancer.

You will need a health check that amazon can use to find which endpoints are alive and which endpoints are dead. The easiest way is to have a tcp or http service listening on the same port (but then tcp), in the same pod as the pods that serve the UDP workload. I wanted to do this using a sidecar container with very low overhead, so i used a small rust (actix) based service that checks the health and responds over http. I used the rust musl toolchain to compile the service so i could have it as a static binary and deploy it in a distroless container, resulting in a docker image of 6 megabytes taking up less than 1MB of ram. Not bad.

After this, my workload has two containers, one running on an UDP port and the other one on the same port but TCP.

# Creating the service

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
  annotations:
    "external-dns.alpha.kubernetes.io/hostname": "test.example.com"
    "external-dns.alpha.kubernetes.io/ttl": "160"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-eip-allocations: "example1,example2,example3"
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  ports:
  - port: 1234
    protocol: UDP
    targetPort: 1234
    name: example
  selector:
    app: example

```

Some things to consider:

* We set the externalTrafficPolicy to Local, so that kubernetes doesn't do cross node routing. Amazon will use the health checks we defined to find out what nodes carry the workload and what nodes don't. You will see that in the target group of your nlb in the AWS console.

* Because of this, we need to enable cross-zone-load-balancing otherwise the IP addresses that are associated with an availability zone that doesn't carry the workload will remain silent (off course this is not needed if you deployed your workload as a DaemonSet and you have nodes in every AZ). What you will see is that your loadbalancer will work sometimes (based on which IP it chooses) but not always.

* Here we needed to have fixed IPs for our loadbalancer so we used eip allocations. This might not be needed in your case.

* The service does not mention the TCP port used for the health check. This is not possible because a loadbalancer service cannot have multiple protocols mixed (as of kubernetes 1.22). But i found that amazon still manages to find the health check even if we didnt' do this.

