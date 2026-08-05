# AI-Powered SRE Incident Automation Platform

**System Documentation**
Environment: WSL2 / Minikube / Istio / n8n / Ollama / Telegram

---

## 1. Executive Summary

I have created an event-driven SRE incident response pipeline on Kubernetes. Istio sidecars capture service traffic metrics, Prometheus records failures, Grafana fires alerts, and an n8n workflow routes each alert to a local Ollama model for analysis. The model produces a root-cause summary and sends an actionable alert card to Telegram.

## 2. System Architecture

Traffic flows from the gateway api into the Bookinfo microservices, where Envoy sidecars report metrics to Prometheus. Grafana alerting rules watch these metrics and send a webhook to n8n when thresholds are breached. n8n forwards the alert payload to Ollama, which generates an incident report that is posted to a Telegram bot. All of the services — n8n, Ollama, and Kubernetes (Minikube) — are running on Docker.

```
Gateway → Bookinfo Services (Envoy) → Prometheus → Grafana Alerting → n8n Webhook → Ollama → Telegram
```

## 3. Service Mesh & Sidecar Injection

A service mesh solves the problem of managing communication between the many small services that make up a modern application — things like routing traffic, encrypting it, retrying failed requests, controlling which services can talk to each other, and monitoring traffic health — without having to build that logic into every service's code. It does this by attaching a lightweight proxy (a "sidecar") next to each service, so all traffic passes through these proxies instead of directly between services.

Istio is the most widely used service mesh: it automatically adds these sidecars to every service running in Kubernetes and gives you traffic control, security (encrypting traffic and enforcing which services can call which), and real-time visibility into how traffic is flowing — all without touching application code. In this project, it's Istio's sidecars that report traffic metrics like error rates, which is what triggers the alerting and automated incident response pipeline.

The default namespace was labeled `istio-injection=enabled`, so every pod deployed into it automatically receives an Envoy sidecar. This was confirmed by checking pod status: each Bookinfo pod shows 2/2 READY containers, one for the application and one for the Envoy proxy. The dedicated gateway proxy for the Kubernetes Gateway API resource (bookinfo-gateway-istio) also runs in the default namespace as the actual entry point for traffic. The istio-system namespace was checked separately to confirm istiod, Prometheus, and Kiali were running; it also still shows a standing istio-ingressgateway, which is installed by default with Istio's demo profile but is unused here since ingress is handled through the Gateway API instead.

![kubectl get pods showing 2/2 READY](screenshots/01-kubectl-get-pods-bookinfo.png)

![kubectl get pods -n istio-system](screenshots/02-kubectl-get-pods-istio-system.png)

## 4. Observability & Traffic Graph

Kiali was used to visualize real-time traffic between services. The mesh graph confirms the core istio-system infrastructure is healthy, and the traffic graph shows normal green traffic flowing from productpage through reviews to ratings, indicating successful requests across the mesh.

![Kiali mesh graph](screenshots/03-kiali-mesh-graph.png)

![Kiali traffic graph - green/healthy traffic](screenshots/04-kiali-traffic-graph-green.png)

## 5. Zero-Trust Security & Authorization Policies

An Istio `AuthorizationPolicy` named `protect-ratings` was applied to enforce zero-trust access control on the ratings service. Cross-namespace testing confirmed that requests from the default namespace were blocked with `403 Forbidden` until the policy was updated to explicitly allow traffic from only the ratings namespace, after which normal traffic resumed.

![AuthorizationPolicy YAML](screenshots/05-authorizationpolicy-yaml.png)

![Kiali traffic graph - blocked traffic (403)](screenshots/06-kiali-traffic-graph-blocked-403.png)

## 6. Event-Driven AI Incident Response Pipeline

n8n is a workflow automation tool that lets you connect different systems together visually, without writing custom integration code for each one. You build a workflow as a chain of nodes — one node listens for a trigger (like an incoming webhook), the next node processes or transforms that data, and following nodes send the result somewhere else (an API, a chat app, a database, etc.).

My n8n workflow was built with three connected nodes: a webhook listener that receives Grafana alerts, a node that sends the alert payload to the local Ollama model for analysis, and a node that delivers the formatted response as a message to Telegram. This automates the path from a raw metric alert to a readable incident report.

![n8n workflow canvas](screenshots/07-n8n-workflow-canvas.png)

## 7. End-to-End Validation & Final Alert

To validate the full pipeline, a fault was injected into the ratings service using an Istio `VirtualService`, forcing a 100% HTTP 500 error rate. Grafana detected the resulting metric spike and fired the webhook, which triggered Ollama to generate an incident report. The final alert was delivered to Telegram with the affected service, status, a summary of the issue, and a suggested debug command.

![Fault injection command](screenshots/08-fault-injection-command.png)

![Grafana alert firing](screenshots/09-grafana-alert-firing.png)

![n8n node output being sent to Telegram](screenshots/10-n8n-telegram-node-output.png)

![n8n execution succeeded](screenshots/11-n8n-execution-succeeded.png)

![Final Telegram AI SRE alert card](screenshots/12-telegram-alert-final.png)

## Conclusion

This lab gave me hands-on experience with the full lifecycle of an incident, from detection to response — something I had only understood in theory before. Working with Istio taught me how a service mesh actually operates in practice: injecting sidecars, reading traffic in Kiali, and enforcing zero-trust access control with AuthorizationPolicies, including deliberately breaking traffic to see the enforcement happen in real time. Building the automation side with n8n and a local Ollama model changed how I think about incident response, showing me how an AI model can do the first pass of root-cause analysis and hand the on-call engineer a head start the moment an alert fires. Overall, this project pushed me past just knowing the names of these tools and into actually operating them together as a system, giving me a stronger foundation for SRE and platform engineering work and a clearer sense of how AI can support incident response rather than replace the judgment of the engineer on call.
