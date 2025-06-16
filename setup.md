# Production-Ready Kubernetes Logging Architecture

## ✅ Objectives

* Centralized logging for all Kubernetes workloads
* Works on any Kubernetes cluster (EKS, AKS, GKE, on-prem)
* Deployable with a single `helm upgrade --install`
* Developer-friendly: auto-collects `stdout` or file-based logs
* Secure, scalable, and observable

---

## 📦 Stack Components

| Component       | Tool       | Purpose                     |
| --------------- | ---------- | --------------------------- |
| Log Collector   | Fluent Bit | Tail pod/container logs     |
| Aggregation     | Loki       | Store & index logs          |
| Visualization   | Grafana    | Query & view logs           |
| Deployment Tool | Helm       | Easy, repeatable deployment |

---

## 🚀 Installation

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Loki
helm upgrade --install loki grafana/loki-stack \
  --namespace logging --create-namespace \
  --set promtail.enabled=false

# Install Fluent Bit with custom config
helm upgrade --install fluent-bit fluent/fluent-bit \
  --namespace logging \
  -f fluent-bit-values.yaml

# Install Grafana
helm upgrade --install grafana grafana/grafana \
  --namespace logging \
  --set adminPassword=admin \
  --set service.type=LoadBalancer \
  --set "datasources.datasources\.yaml.apiVersion=1" \
  --set "datasources.datasources\.yaml.datasources[0].name=Loki" \
  --set "datasources.datasources\.yaml.datasources[0].type=loki" \
  --set "datasources.datasources\.yaml.datasources[0].url=http://loki:3100" \
  --set "datasources.datasources\.yaml.datasources[0].access=proxy" \
  --set "datasources.datasources\.yaml.datasources[0].isDefault=true"
```

---

## 🛠 Fluent Bit: Sample `fluent-bit-values.yaml`

```yaml
config:
  service: |
    [SERVICE]
        Flush        1
        Daemon       Off
        Log_Level    info
        Parsers_File parsers.conf
        Parsers_File /fluent-bit/etc/conf/custom_parsers.conf
        HTTP_Server  On
        HTTP_Listen  0.0.0.0
        HTTP_Port    2020

  inputs: |
    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Tag               kube.*
        Refresh_Interval  5
        Rotate_Wait       30
        Mem_Buf_Limit     10MB
        Skip_Long_Lines   On
        DB                /var/log/flb_kube.db
        multiline.parser  docker, cri

  filters: |
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Merge_Log_Key       log_processed
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On

  outputs: |
    [OUTPUT]
        Name                    loki
        Match                   *
        Host                    loki.logging.svc.cluster.local
        Port                    3100
        line_format             json
        Labels                  job=fluentbit
        auto_kubernetes_labels  On

backend:
  type: loki
```

---

## 🔍 Grafana LogQL Queries

* All logs:

  ```logql
  {}
  ```
* Filter by container:

  ```logql
  {container="my-app"}
  ```
* Error logs:

  ```logql
  {container="my-app"} |= "ERROR"
  ```

---

## 🧠 Troubleshooting

* No logs in Grafana:

  * Query is too narrow → use \`{}
  * Wrong time range → expand to "Last 1h"
  * Fluent Bit not sending → check logs for `output:loki` 204 status
  * Loki unreachable → test with `curl http://loki.logging.svc.cluster.local:3100/ready`
* Check Fluent Bit config errors with `kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit`
* Common issues:

  * Misconfigured `line_format` (should be `line_format`, not `lineformat`)
  * Invalid `auto_kubernetes_labels` spelling (should match docs exactly)

---

## 🔐 Best Practices

* RBAC limited to `logging` namespace
* Use `resource limits` for Fluent Bit pods
* Secure Grafana with OAuth or access policies
* Use Loki ruler or Alertmanager for log-based alerting
* Keep log volume low with filters or label dropping

---

## 🧪 Future Enhancements

* JSON log parsing & multiline support
* Labels for environment, team, task (via Fluent Bit)
* GitOps integration with ArgoCD/Flux
* Log retention via object storage (MinIO, S3, Azure Blob)

---

✅ **Status**: Fully functional logging stack, reusable across projects.

This Markdown file is ready to push to GitHub.
You can import this setup into Helmfile, Terraform modules, or GitOps pipelines with minimal changes.
