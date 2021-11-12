# Kubecost

Kubecost is a tool to calculate expenses with any kubernetes cluster, in this specific case we will install via helm with AWS EKS and use custom prometheus and grafana.

## Dependencies

Use the helm manager to deploy [prometheus-operator](https://github.com/prometheus-operator/prometheus-operator).

```bash
$helm install prom stable/prometheus-operator --namespace monitoring -f prometheus-values.yaml
```
Note that in the file prometheus-valules.yaml we have some additional rules and scrapeconfigs, we depend on them for the complete functioning of kubecost.

By default kube-proxy exposes its metrics in 127.0.0.1/metrics and prometheus cannot access, so use the command below to change the output.

```bash
$kubectl -n kube-system get cm kube-proxy-config -o yaml |sed 's/metricsBindAddress: 127.0.0.1:10249/metricsBindAddress: 0.0.0.0:10249/' | kubectl apply -f -
       kubectl -n kube-system patch ds kube-proxy -p "{\"spec\":{\"template\":{\"metadata\":{\"labels\":{\"updateTime\":\"`date +'%s'`\"}}}}}"
```

## Installation

We will install kubecost with the helm, we must configure the values.yaml file directing the prometheus(fqdn) and grafana composed as follows:

```yaml
global:

  prometheus:
    enabled: false # If false, Prometheus will not be installed
    fqdn: http://<service-of-promeheus>.<namespace>.svc:<port> #Ignored if enabled: true
    #Example of fqdn: http://prom-prometheus-operator-prometheus.monitoring.svc:9090
    

  grafana:
    enabled: false # If false, Grafana will not be installed
    domainName: <service-of-grafana>.<namespace>.svc #grafana domain ignored if enabled: true
    #Example of domainName: prom-grafana.monitoring.svc
    proxy: false
```

After finishing the assembly of your values, generating a token on [kubecost site](https://www.kubecost.com/install.html#show-instructions) and perform the installation with the command below:

```bash
$kubectl create namespace kubecost

$helm repo add kubecost https://kubecost.github.io/cost-analyzer/

$helm install kubecost kubecost/cost-analyzer --namespace kubecost --set kubecostToken="token" -f values.yaml
```
To test the deploy, use the command below to expose the service locally and validate its operation:

```bash
kubectl port-forward svc/kubecost-cost-analyzer 8080:9090 -n kubecost
```

Access [127.0.0.1;8080](http://127.0.0.1:8080).

If you wish, use an nginx ingress to expose the application as shown below:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubecost
  namespace: kubecost
  annotations:
    nginx.ingress.kubernetes.io/configuration-snippet: |-
      proxy_ssl_server_name on;
      proxy_ssl_name $host;
    nginx.ingress.kubernetes.io/rewrite-target: /
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/proxy-body-size: 12m
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kubecost-cost-analyzer
            port:
              number: 9090
```