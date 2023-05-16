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
helm upgrade prometheus prometheus
```

 
Access to Prometheus UI
```
kubectl port-forward svc/prometheus-server 9090:80 --address 0.0.0.0 -n monitoring
```
