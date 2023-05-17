# How to install Prometheus operator with helm chart

### Introduction
Prometheus is an open-source monitoring and alerting system. It has multiple tool components, including:
* the main Prometheus server, which scrapes and stores time series data.
* client libraries that instrument the application code.
* a push gateway for handling short-term tasks.
* special-purpose exporters are provided for services like HAProxy, StatsD, Graphite, etc.
* alertmanager to handle alerts.
* various support tools.
* Graphite or other API consumers can be used to visualize the collected data.

Many of them are optional.

![image](https://github.com/sagarshrestha24/k8s/assets/76894861/83b963f5-0da3-455b-b749-a223ee0dcd94)


### Helm Charts

Helm is package manager for Kubernetes and is a perfect tool to find, share and use software.

Helm Charts therefore help to define, install, and upgrade even the most complex Kubernetes application and allow controlling and maintain it easily by using a template where all Kubernetes deployment is defined.
![image](https://github.com/sagarshrestha24/k8s/assets/76894861/cba0903b-9a6e-438d-b276-5b8f46ddc4b2)


Let’s start and set it up. Here we will be working on mickrok8s k8s cluster, but it applies anywhere.

Install Helm from script like in commands below or other way you want, but keep in mind we will be using Helm 3 here, For Helm 2 you will need Helm server (Tiller) as well.

```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

$ chmod 700 get_helm.sh

$ ./get_helm.sh

```

In this guide, instead of creating our own Prometheus template, we will use existing Helm Charts, which are available on GitHub and are well tested and maintained.

We need to add the Prometheus repository via Helm and update it like below.
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

A message with success status should appear after that.

![image](https://github.com/sagarshrestha24/k8s/assets/76894861/fca0b643-2cd6-4999-b556-5cdc0246ffe8)

We can install Prometheus without creating our own template and only use default values:
```
helm install prometheus prometheus-community/prometheus
```
 
When we want to upgrade the configuration,
```
git clone https://github.com/osmsolutions/mbm-infra
cd mbm-infra/monitoring
helm upgrade prometheus prometheus -n monitoring
```

 
Access to Prometheus UI
```
kubectl port-forward svc/prometheus-server 9090:80 --address 0.0.0.0 -n monitoring
```



## Prometheus Blackbox Exporter

Blackbox Exporter is used to probe endpoints like HTTPS, HTTP, TCP, DNS, and ICMP. After you define the endpoint, Blackbox Exporter generates metrics that can be visualized using tools like Grafana. One of the most important feature of Blackbox Exporter is measuring the response time of endpoints.

The following diagram shows the flow of Blackbox Exporter monitoring an endpoint.
![image](https://github.com/sagarshrestha24/k8s/assets/76894861/5abd9868-c9fd-4d90-8f97-374f84bd8e53)

Clone the blackbox-exporter helm chart with
```
git clone https://github.com/osmsolutions/mbm-infra
cd mbm-infra/monitoring
```

Install BlackBox exporter with helm charts with command:
```
helm install prometheus-blackbox-exporter ./prometheus-blackbox-exporter -n monitoring
```
verify after BlackBox exporter installation on k8s

![image](https://github.com/sagarshrestha24/k8s/assets/76894861/ab2721f2-f946-44f0-942e-e57b391f35a0)

Configure Prometheus for Blackbox exporter, you can add the configuration under additionalScrapeConfigs[] in value.yaml in kube-prometheus-stack helm chart.

We will majorly be adding configs for the following in Prometheus for our endpoint monitoring.

    Probing external targets
    Probing services via the Blackbox Exporter
    Probing ingresses via the Blackbox Exporter
    Probing pods via Blackbox Exporter

In the kube-prometheus-stack’s values.yaml, add the following blocks under additionalScrapeConfigs[] section:
```
additionalScrapeConfigs: 
      - job_name: 'uptime-check-endpoint'
        metrics_path: /probe
        params:
          module: [http_2xx]
        static_configs:
          - targets:
            - https://google.com
            - https://abc.com:8080
            - <add-external-and-internal-endpoint>
                
        relabel_configs:
          - source_labels: [__address__]
            target_label: __param_target
          - source_labels: [__param_target]
            target_label: instance
          - target_label: __address__
            replacement: prometheus-blackbox-exporter:9115
```

After adding above config ,upgrade helm charts :
```
helm upgrade prometheus prometheus  -n monitoring
```

Verify the generated metrics in Prometheus

Once the changes are applied and the resources for the Blackbox Exporter are deployed, we can verify the status of targets in Prometheus. We can check whether the Blackbox Exporter is up with the registered targets by navigating to the Status tab and then selecting Targets in the Prometheus UI.

Here you can see we are using https://www.google.com as an external target for reference with its state UP. We can also check if metrics are getting populated by looking for metrics starting with probe_

![image](https://github.com/sagarshrestha24/k8s/assets/76894861/5dbca9d0-9d86-42e4-9974-e918d2e142d5)



