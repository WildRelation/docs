# Routing

This page explains how external traffic is routed to services running inside the Kubernetes clusters.

## Overview

Incoming traffic passes through multiple routing layers before reaching workloads in the clusters. This includes DNS resolution, firewall and NAT routing, reverse proxying through NGINX, and internal load balancing handled by MetalLB.

## DNS

DNS records point to the **public IP addresses** of the cloud.
These entries allow users and services to access the system using domain names instead of raw IPs.

There are multiple DNS entries configured, below are some examples:

* **System cluster**

  * `*.cloud.cbh.kth.se` (resolves cloud.cbh.kth.se and api.cloud.cbh.kth.se for example)
  * ...
* **Application clusters**

  * `*.app.cloud.cbh.kth.se`
  * `*.vm-app.cloud.cbh.kth.se`
  * ...

## Firewall / NAT

The firewall handles **Network Address Translation (NAT)** between the cloud’s **public IPs** and the **internal IPs** assigned to NGINX within each Kubernetes cluster.

Each cluster’s NGINX ingress controller is exposed using **MetalLB**, which allocates a load balancer IP from its pool.
This IP must be referenced in the NAT configuration so that public traffic is routed correctly.

You can retrieve the NGINX ingress IP for a cluster using:

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

This command should be run **on any node in every cluster**.

The resulting IPs must then be configured in the **firewall NAT** so that:

* The **public IP** resolved by DNS for the *system cluster* (used by the console and deploy services) or the public ip for the cluster.
* For both **HTTP (port 80)** and **HTTPS (port 443)** traffic.

## Reverse Proxy (NGINX)

Each cluster runs an NGINX ingress controller that acts as the reverse proxy.
It routes traffic based on hostnames and paths to the correct services within the cluster, handling TLS termination and forwarding.

### Cert manager

Cert manager handles the provisioning of TLS certificates for each endpoint.

## MetalLB / Load Balancer

[MetalLB](https://metallb.universe.tf/) is used within each cluster to provide load balancing for services of type `LoadBalancer`.
It assigns a “static”[^1] IP to the NGINX ingress controller, which is then referenced in the firewall NAT configuration as described above.

[^1]: The IP is “static” in normal operation but will change if the `ingress-nginx` service is redeployed.
