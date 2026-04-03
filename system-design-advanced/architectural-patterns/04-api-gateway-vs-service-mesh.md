---
slug: api-gateway-vs-service-mesh
title: "API Gateway vs Service Mesh"
readTime: 20 min
orderIndex: 4
premium: false
---

# API Gateway vs Service Mesh

> Two infrastructure patterns that look similar but solve completely different problems — one handles external traffic in, the other handles internal traffic between services.




---

## Key Takeaways

1. **API gateways handle north-south traffic (client to service)** — routing, authentication, rate limiting, and request transformation
2. **Service meshes handle east-west traffic (service to service)** — mTLS, retries, circuit breaking, and observability between internal services
3. **They solve different problems and complement each other** — use an API gateway for external traffic and a service mesh for internal traffic
4. **Istio, Linkerd, and Consul Connect are the major service mesh options** — each trades operational complexity for infrastructure-level reliability features
