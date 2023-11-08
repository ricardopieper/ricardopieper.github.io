---
title: Deploying an ASP.NET gRPC service on GKE with GCE Ingress
author:
  name: Ricardo Pieper
  link: https://github.com/ricardopieper
date: 2021-12-11 03:00:00 -0300
categories: [Kubernetes, GKE, GCE]
tags: [kubernetes gke gce]
pin: true
---

## Context

At Monomyto Game Studios, we use Kubernetes to manage our infrastructure. Our backend team consists, currently, of just one person (me), so I'd like to keep things as simple as possible, while still having a way to define all of our infrastructure in a single place, and make it deployable quickly using `kubectl`. Our development team is mostly composed of Unity programmers, some of them having done a little bit of backend development but not much.

Our game, Gunstars, has a gRPC service written in C# and ASP.NET. We also use MagicOnion as an abstraction layer on top of gRPC, as it works well on Unity and makes it possible to simply define interfaces instead of writing `.proto` files. In order to manage and scale this application, we use Kubernetes. Specifically, we decided to deploy it on GKE (Google Kubernetes Engine). Our requirements for this application are:

 - Handle realtime communications
 - Manage player sessions
 - Manage squads
 - Perform matchmaking functions
 - Manage friend requests

As you might know, gRPC uses HTTPS/2 under the hood to establish a bidirectional communications channel. This application is exposed
to the internet, and therefore needs HTTPS to make the communication secure.

## Configuring HTTPS + HTTP/2

For a simple service that does not need HTTPS, we can simply use a `LoadBalancer` service on kubernetes:

```yml
apiVersion: v1
kind: Service
metadata:
  name: gunstars-realtime-service
  labels:
    app: gunstars-realtime-service
spec:
  ports:
    - port: 5000
      targetPort: 5000
  selector:
    app: gunstars-realtime
  type: LoadBalancer
```

The load balancer will be exposed to the internet, and GKE will automatically assign a public IP to this service. However, our options are limited here in terms of HTTPS: There seems to be no way to configure the load balancer with this kind of service.

Thankfully, we have another kind of resource on kubernetes: Ingress. An external ingress also exposes a service to the internet, but we have more options to play with. Let's take a look at our deployment first:

```yml
# Some details ommited for brevity
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gunstars-realtime
  labels:
    app: gunstars-realtime
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gunstars-realtime
  template:
    metadata:
      labels:
        app: gunstars-realtime
    spec:
      containers:
      - name: gunstars-realtime
        image: "..."
        imagePullPolicy: "Always"
        env:
          - name: ASPNETCORE_URLS
            value: "http://+:5000;http://+:5001"
        ports:
          - containerPort: 5000
          - containerPort: 5001
```

This deployment exposes 2 ports: 5000 and 5001. 5000 will be used for the gRPC HTTP/2 connections, while 5001 is only for health checking and will not be exposed to the internet. Notice that our ASPNETCORE_URLS also configures some HTTP listeners. This is where my mistake lives, as you'll soon find out.


Now it's time to define our service. Let's take a look:

```yml
apiVersion: v1
kind: Service
metadata:
  annotations:
    cloud.google.com/app-protocols: '{"grpc":"HTTP2", "health":"HTTP"}'
    cloud.google.com/backend-config: '{"default": "gunstars-realtime-healthcheck"}'
    cloud.google.com/neg: '{"ingress": true, "exposed_ports": {"5000":{}}}'
  name: gunstars-realtime-service
  labels:
    app: gunstars-realtime-service
spec:
  ports:
    - port: 5000
      targetPort: 5000
      protocol: TCP
      name: grpc
    - port: 5001
      targetPort: 5001
      protocol: TCP
      name: health
  selector:
    app: gunstars-realtime
  type: NodePort
```

Notice that we define the ports of the service. The `cloud.google.com/app-protocols` annotation lets us define the protocol for each port, which have been named as `grpc` or `health`.

Also notice the `cloud.google.com/backend-config` annotation: we need to tell GKE how to perform health checking on our service, otherwise the backend will report its status as `NOT_READY` or `UNKNOWN`. We also explicitly tell which ports are exposed. Maybe some of this configuration isn't really necessary, but I haven't tested yet.

Our backend config looks like this:

```yml
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: gunstars-realtime-healthcheck
spec:
  healthCheck:
    checkIntervalSec: 15
    port: 5001
    type: HTTP
    requestPath: /health
  connectionDraining:
    drainingTimeoutSec: 15
```

### Ingress definition

To define an ingress, we have a choice to make. We can use a load balancer such as Nginx, or use Google's load balancer. Each one has its pros and cons: Nginx has tons of configurations to choose and is very flexible, while GCE Load balancer is not very configurable. However, due to the size and inexperience of the team regarding backend services, the tradeoff I'm choosing is letting GCE manage most things. The backend is a responsability of the entire company, and everyone should have some knowledge about how it works. Linking a certificate with Nginx automatically would require a ton of configuration with `cert-manager.io`, which would use Letsencrypt to generate a certificate, and renew it. However, the amount of configuration needed can be overwhelming. Therefore, we chose GCE's load balancer (GCLB) which is the default load balancer on GKE.

Therefore, we can just declare a `ManagedCertificate` like this:

```yml
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: gunstars-realtime-cert
spec:
  domains:
    - ...(ommited)...
```

This certificate can be linked to the ingress:

```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gunstars-realtime-https-ingress
  annotations:
    networking.gke.io/managed-certificates: "gunstars-realtime-cert"
    kubernetes.io/ingress.global-static-ip-name: gunstars-realtime-ingress #a reserved IP
spec:
  defaultBackend:
    service:
      name: gunstars-realtime-service-https
      port:
        number: 5000
```

And success! Everything should work, right?

### Incorrect assumptions

... and there goes 3 or 4 days investigating why it doesnt. Turns out that, whenever I sent a request to the backend, I would be met with a 502 error, while the access logs on the Ingress would just say `failed_to_connect_to_backend`. Why would that be the case?

I tried deploying an HTTP1.1 application, it worked fine. I tried using the `gcr.io/google-containers/echoserver` to test HTTP/2 specifically, it worked fine. What gives?

In my experience, the applications inside our network (behind a load balancer) can just listen to unencrypted traffic, while the load balancer takes care of HTTPS. It seems like this works for HTTP 1.1, but GKE is quite picky regarding HTTP/2. It is assumed that HTTP/2 will always use HTTPS, but that's not entirely true: it is possible to use unencrypted HTTP/2 (for instance, when testing things locally).

Let's take a look at our deployment again, specifically our environment variables:

```yml
env:
  - name: ASPNETCORE_URLS
    value: "http://+:5000;http://+:5001"
```

That seems to be following what's been true in my experience. My ASP.NET application is listening on port 5000 unencrypted.

**However**, for some reason, that doesn't work: we need to use HTTPS **in the deployment** as well! So I changed our deployment to this:

```yml
env:
  - name: ASPNETCORE_URLS
    value: "https://+:5000;http://+:5001"
  - name: ASPNETCORE_Kestrel__Certificates__Default__Path
    value: "./https/aspnetapp.pfx"
  - name: ASPNETCORE_Kestrel__Certificates__Default__Password
    value: "..."
```

Now Kestrel is listening on 5000 for HTTPS traffic, and I just used the development certificate I already had in hand.
GCLB will not check whether the certificate is valid or not.

I also defined the URLs in the `appsettings.json` file:

```js
{
  "Kestrel": {
    "Endpoints": {
      "Grpc": {
        "Url": "https://0.0.0.0:5000",
        "Protocols": "Http2"
      },
      "Http": {
        "Url": "http://0.0.0.0:5001",
        "Protocols": "Http1"
      }
    }
  }
}
```

And that works!

## Conclusion

After googling for 3 days on and off for a solution to `failed_to_connect_to_backend`, I couldn't find anything. Everything seemed to indicate that unencrypted traffic to the backend should work. However, I actually didn't read things carefully. For instance:

[GCP HTTPS LB health check is working but no real traffic getting in, what could explain this?](https://serverfault.com/questions/1013642/gcp-https-lb-health-check-is-working-but-no-real-traffic-getting-in-what-could)

[GCP Load Balancer: 502 Server Error, "failed_to_connect_to_backend"](https://stackoverflow.com/questions/51474928/gcp-load-balancer-502-server-error-failed-to-connect-to-backend)

On these stackoverflow posts, it now seems to be implied that the deployment (or backend instances) should be listening for HTTPS, encrypted traffic.

Other posts are seemingly more thorough, but do not address the issue at all:

[Server behind GCP Load Balancer occalionally gets 502 Server Error, “failed_to_connect_to_backend”](https://stackoverflow.com/questions/63289794/server-behind-gcp-load-balancer-occalionally-gets-502-server-error-failed-to-c)

When you do things in a rush and have to do many context switches in your day, keeping track of everything and give that much attention to detail to what people write can be hard. I think this should be better documented in the GKE guides. As someone who is responsible for the backend services but isn't that much familiar with proxies and load balancing configuration, this can be an unnecessarily time consuming endeavor.

Maybe if I had read things more carefully I would have realized this before. But if you're having the same problem as me (`failed_to_connect_to_backend`), there you go: **just do HTTPS all the way, and use a self-signed certificate on the deployment**.
Another possible fix is to use nginx, which offers far more flexibility and tons of configurations, and `cert-manager.io` to automatically apply SSL certificates to the ingress, just be aware that this can be overwhelming for a small and inexperienced team.

I hope no one wastes 3 days on this. Have a good day!
