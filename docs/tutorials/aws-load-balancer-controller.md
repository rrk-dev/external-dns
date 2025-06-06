# AWS Load Balancer Controller

This tutorial describes how to use ExternalDNS with the [aws-load-balancer-controller][1].

[1]: https://kubernetes-sigs.github.io/aws-load-balancer-controller

## Setting up ExternalDNS and aws-load-balancer-controller

Follow the [AWS tutorial](aws.md) to setup ExternalDNS for use in Kubernetes clusters
running in AWS. Specify the `source=ingress` argument so that ExternalDNS will look
for hostnames in Ingress objects. In addition, you may wish to limit which Ingress
objects are used as an ExternalDNS source via the `ingress-class` argument, but
this is not required.

For help setting up the AWS Load Balancer Controller, follow the [Setup Guide][2].

[2]: https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/deploy/installation/

Note that the AWS Load Balancer Controller uses the same tags for [subnet auto-discovery][3]
as Kubernetes does with the AWS cloud provider.

[3]: https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/deploy/subnet_discovery/

In the examples that follow, it is assumed that you configured the ALB Ingress
Controller with the `ingress-class=alb` argument (not to be confused with the
same argument to ExternalDNS) so that the controller will only respect Ingress
objects with the `ingressClassName` field set to "alb".

## Deploy an example application

Create the following sample "echoserver" application to demonstrate how
ExternalDNS works with ALB ingress objects.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echoserver
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echoserver
  template:
    metadata:
      labels:
        app: echoserver
    spec:
      containers:
      - image: gcr.io/google_containers/echoserver:1.4
        imagePullPolicy: Always
        name: echoserver
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: echoserver
spec:
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
  type: NodePort
  selector:
    app: echoserver
```

Note that the Service object is of type `NodePort`. We don't need a Service of
type `LoadBalancer` here, since we will be using an Ingress to create an ALB.

## Ingress examples

Create the following Ingress to expose the echoserver application to the Internet.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
  name: echoserver
spec:
  ingressClassName: alb
  rules:
  - host: echoserver.mycluster.example.org
    http: &echoserver_root
      paths:
      - path: /
        backend:
          service:
            name: echoserver
            port:
              number: 80
        pathType: Prefix
  - host: echoserver.example.org
    http: *echoserver_root
```

The above should result in the creation of an (ipv4) ALB in AWS which will forward
traffic to the echoserver application.

If the `--source=ingress` argument is specified, then ExternalDNS will create
DNS records based on the hosts specified in ingress objects. The above example
would result in two alias records (A and AAAA) being created for each of the
domains: `echoserver.mycluster.example.org` and `echoserver.example.org`. All
four records alias the ALB that is associated with the Ingress object. As the
ALB is IPv4 only, the AAAA alias records have no effect.

If you would like ExternalDNS to not create AAAA records at all, you can add the
following command line parameter: `--exclude-record-types=AAAA`. Please be
aware, this will disable AAAA record creation even for dualstack enabled load
balancers.

Note that the above example makes use of the YAML anchor feature to avoid having
to repeat the http section for multiple hosts that use the exact same paths. If
this Ingress object will only be fronting one backend Service, we might instead
create the following:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    external-dns.alpha.kubernetes.io/hostname: echoserver.mycluster.example.org, echoserver.example.org
  name: echoserver
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /
        backend:
          service:
            name: echoserver
            port:
              number: 80
        pathType: Prefix
```

In the above example we create a default path that works for any hostname, and
make use of the `external-dns.alpha.kubernetes.io/hostname` annotation to create
multiple aliases for the resulting ALB.

## Dualstack Load Balancers

AWS [supports both IPv4 and "dualstack" (both IPv4 and IPv6) interfaces for ALBs][4]
and [NLBs][5]. The AWS Load Balancer Controller uses the `alb.ingress.kubernetes.io/ip-address-type`
annotation (which defaults to `ipv4`) to determine this. ExternalDNS creates
both A and AAAA alias DNS records by default, regardless of this annotation.
It's possible to create only A records with the following command line
parameter: `--exclude-record-types=AAAA`

[4]: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html#ip-address-type
[5]: https://docs.aws.amazon.com/elasticloadbalancing/latest/network/network-load-balancers.html#ip-address-type

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/ip-address-type: dualstack
  name: echoserver
spec:
  ingressClassName: alb
  rules:
  - host: echoserver.example.org
    http:
      paths:
      - path: /
        backend:
          service:
            name: echoserver
            port:
              number: 80
        pathType: Prefix
```

The above Ingress object will result in the creation of an ALB with a dualstack
interface.
