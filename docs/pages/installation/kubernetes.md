---
layout: default
title: Kubernetes
permalink: installation/kubernetes
type: installation
category: kubernetes
redirect_from:
- installation/platforms/kubernetes
display-title: "false"
language: en
list: include
image: /assets/img/platforms/kubernetes.svg
abstract: Install Meshery on Kubernetes. Deploy Meshery in Kubernetes in-cluster or outside of Kubernetes out-of-cluster.
---

<h1>Quick Start with {{ page.title }} <img src="{{ page.image }}" style="width:35px;height:35px;" /></h1>

Manage your Kubernetes clusters with Meshery. Deploy Meshery in Kubernetes [in-cluster](#in-cluster-installation) or outside of Kubernetes [out-of-cluster](#out-of-cluster-installation). **_Note: It is advisable to install Meshery in your Kubernetes clusters_**

<div class="prereqs"><h4>Prerequisites</h4>
  <ol>
    <li>Install the Meshery command line client, <a href="{{ site.baseurl }}/installation/mesheryctl" class="meshery-light">mesheryctl</a>.</li>
    <li>Install <a href="https://kubernetes.io/docs/tasks/tools/">kubectl</a> on your local machine.</li>
    <li>Access to an active Kubernetes cluster.</li>
  </ol>
</div>

## Available Deployment Methods

- [In-cluster Installation](#in-cluster-installation)
  - [Preflight Checks](#preflight-checks)
    - [Preflight: Cluster Connectivity](#preflight-cluster-connectivity)
    - [Preflight: Kubeconfig Permissions](#preflight-kubeconfig-permissions)
  - [Installation: Using `mesheryctl`](#installation-using-mesheryctl)
  - [Installation: Using Helm](#installation-using-helm)
  - [Post-Installation Steps](#post-installation-steps)
- [Out-of-cluster Installation](#out-of-cluster-installation)
  - [Set up Ingress on Minikube with the NGINX Ingress Controller](#set-up-ingress-on-minikube-with-the-nginx-ingress-controller)
  - [Installing cert-manager with kubectl](#installing-cert-manager-with-kubectl)

# In-cluster Installation

Follow the steps below to install Meshery in your Kubernetes cluster.

## Preflight Checks

Read through the following considerations prior to deploying Meshery on Kubernetes.

### Preflight: Cluster Connectivity

Verify your kubeconfig's current context is set to the Kubernetes cluster you want to deploy Meshery to.
{% capture code_content %}kubectl config current-context{% endcapture %}
{% include code.html code=code_content %}

### Preflight: Kubeconfig Permissions

Meshery requires specific Kubernetes permissions to connect to and manage your cluster. The kubeconfig you provide must grant access to the following resources.

#### Minimum Permissions for Cluster Connection

For Meshery to successfully connect to your Kubernetes cluster, the kubeconfig must allow:

- **`GET` on the `kube-system` namespace** – Meshery uses the `kube-system` namespace UID as a unique cluster identifier during the connection process.
- **Discovery API access** – to retrieve cluster server version information.
- **Non-resource URL `/livez`** – to verify cluster connectivity (ping test).

#### Full Permissions for All Features

For Meshery to use all features—including MeshSync (cluster resource discovery), design pattern deployment, and service mesh management—the kubeconfig must have broad, cluster-wide permissions. The following ClusterRole reflects the minimum required for full Meshery functionality:

{% capture code_content %}apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: meshery-role
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs: ["/metrics", "/health", "/ping", "/livez", "/healthz"]
  verbs:
  - get
{% endcapture %}
{% include code.html code=code_content %}

#### Creating a Dedicated ServiceAccount

If you prefer to use a dedicated ServiceAccount rather than your admin kubeconfig, follow these steps:

**Step 1:** Create a namespace, ServiceAccount, and ClusterRoleBinding:
{% capture code_content %}kubectl create namespace meshery
kubectl create serviceaccount meshery -n meshery
kubectl create clusterrolebinding meshery-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=meshery:meshery
{% endcapture %}
{% include code.html code=code_content %}

**Step 2:** Apply the ClusterRole and generate a kubeconfig token:
{% capture code_content %}kubectl apply -f meshery-clusterrole.yaml
kubectl create token meshery -n meshery --duration=8760h
{% endcapture %}
{% include code.html code=code_content %}

#### Verifying Permissions

Use `kubectl auth can-i` to verify that your kubeconfig has the required permissions before connecting Meshery:
{% capture code_content %}# Verify access to the kube-system namespace (required for connection)
kubectl auth can-i get namespaces/kube-system

# Verify access to list pods cluster-wide (required for MeshSync)
kubectl auth can-i list pods --all-namespaces

# Verify access to deploy resources (required for pattern deployment)
kubectl auth can-i create deployments --all-namespaces
{% endcapture %}
{% include code.html code=code_content %}

To verify permissions for a specific ServiceAccount:
{% capture code_content %}kubectl auth can-i get namespaces/kube-system \
  --as=system:serviceaccount:meshery:meshery
{% endcapture %}
{% include code.html code=code_content %}

#### Troubleshooting Permission Issues

If Meshery cannot connect to your cluster or shows a connection error, check the following:

1. **`kube-system` namespace access** – Meshery reads the `kube-system` namespace UID to uniquely identify your cluster. Without `get` access to this namespace, the connection will fail.
2. **Cluster reachability** – Meshery pings `/livez` on the Kubernetes API server. Ensure non-resource URL access is granted.
3. **Discovery API** – Meshery retrieves the server version via the Kubernetes discovery API. Ensure the kubeconfig user has access.

Run the following commands to diagnose permission issues:
{% capture code_content %}kubectl auth can-i get namespaces/kube-system
kubectl auth can-i get --non-resource-url=/livez
{% endcapture %}
{% include code.html code=code_content %}

## Installation: Using `mesheryctl`

Once configured, execute the following command to start Meshery.

Before executing the below command, go to ~/.meshery/config.yaml and ensure that the current platform is set to Kubernetes.
{% capture code_content %}$ mesheryctl system start{% endcapture %}
{% include code.html code=code_content %}

## Installation: Using Helm

For detailed instructions on installing Meshery using Helm V3, please refer to the [Helm Installation](/installation/kubernetes/helm) guide.

## Post-Installation Steps

Optionally, you can verify the health of your Meshery deployment using <a href='/reference/mesheryctl/system/check'>mesheryctl system check</a>.

You're ready to use Meshery! Open your browser and navigate to the Meshery UI.

{% include_cached installation/accessing-meshery-ui.md display-title="true" %}

# Out-of-cluster Installation

Install Meshery on Docker (out-of-cluster) and connect it to your Kubernetes cluster.

<!-- ## Installation: Upload Config File in Meshery Web UI

- Run the below command to generate the _"config_minikube.yaml"_ file for your cluster:

 <pre class="codeblock-pre"><div class="codeblock">
 <div class="clipboardjs">kubectl config view --minify --flatten > config_minikube.yaml</div></div>
 </pre>

- Upload the generated config file by navigating to _Settings > Environment > Out of Cluster Deployment_ in the Web UI and using the _"Upload kubeconfig"_ option. -->

## Set up Ingress on Minikube with the NGINX Ingress Controller
- Run the below command to enable the NGINX Ingress controller for your cluster:

 <pre class="codeblock-pre"><div class="codeblock">
 <div class="clipboardjs">minikube addons enable ingress</div></div>
 </pre>

- To check if NGINX Ingress controller is running
 <pre class="codeblock-pre"><div class="codeblock">
 <div class="clipboardjs">kubectl get pods -n ingress-nginx</div></div>
 </pre>

## Installing cert-manager with kubectl
- Run the below command to install cert-manager for your cluster:

 <pre class="codeblock-pre"><div class="codeblock">
 <div class="clipboardjs">kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.3/cert-manager.yaml</div></div>
 </pre>

{% include related-discussions.html tag="meshery" %}
