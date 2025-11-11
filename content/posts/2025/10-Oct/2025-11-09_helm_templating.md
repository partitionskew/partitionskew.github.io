+++
title = 'Helm Templating - Basics'
summary = 'A quick tutorial on Helm charts'
tags = ["kubernetes", "helm", "templating", "helm charts"]
date = 2025-11-09
showToc = true
draft = false
+++

## Introduction

Helm is a package manager and helm charts are packages for deploying software in Kubernetes (k8s) clusters. Helm charts make it easy to install applications, data stores such as Redis or Vitess, and even core infrastructure such as Prometheus and service meshes in a k8s cluster in a DRY manner.

In this article, we introduce the basics of helm charts. We'll discuss what helm charts are and how to create one. In the next article, we'll cover the Helm templating language and go through some common patterns/bugs. 

_Note that this article assumes you understand the basics of Kubernetes._

## What are helm charts?

A helm chart consists of several files and folders. These all get bundled up and can be shared with other developers via internal or external Helm repositories. A typical helm chart has the following folder structure:

```
Chart.yaml
README.md
values.yaml
templates/
```

This is by no means an exhaustive list. There are other files and folders that are omitted for brevity. From top to bottom:
* `Chart.yaml` contains basic information about the chart, and some of the definitions can be used when rendering the final k8s manifests from the templates folder
* `values.yaml` contains default values for any variables or definitions used in the templates of the `templates/` folder
* the `templates/` folder is where all the k8s manifests are found

The `Chart.yaml` has some mandatory and optional fields [2]:
* **apiVersion**: v3 (required)
  * version of Helm used by the chart (e.g. v1, v2, v3)
* **name**: example-chart (required)
* **version**: 0.1.0 (required)
  * use semantic versioning and it is convention to start from 0.1.0 for pre-release charts
* **type**: application (optional)
  * one of application or library to indicate what type of chart it is
* **appVersion**: "1.2.3" (optional)
  * indicates what version of the app is being deployed by an application chart (e.g. MySQL chart deploys a v8 MySQL DB)

Again, the list presented above is not exhaustive.

Most charts are `application` charts meaning that they install or deploy an application to a k8s cluster. Some charts are library charts meaning that they serve as dependencies for other charts. We'll cover an example of a library chart in the next article where we discuss Helm templating.

## What happens when we install a chart?

To create a k8s object, you submit a manifest to the Kube API server. The manifest contains the specification for the type of object we want to create as well as its properties. As an example, we can write 3 separate manifests to deploy an application in our development, staging, and production k8s clusters, respectively. Typically one manifest isn't enough to represent a complex application. It is very likely each environment would require a collection of manifests to represent a single deployment of an application. It's also likely you'll deploy the same app more than once in a given environment. For example, you may be partitioning your data using various dimensions such as the region or by some QoS tier.

To DRY (do not repeat yourself) this process, use a template to represent the common bits of the manifest. Similar to a function, the variable bits of the manifest can be replaced using values from a values file or passed in via the command line. Given a set of templates and one or more values files, we can therefore render a complete set of manifests for a given deployment of our application.

Here's an example of a basic Helm template for a pod and the corresponding values file to deploy it in the `development` cluster.

`pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ .Values.environment }}-my-app-pod
  labels:
    environment: {{ .Values.environment }}
spec:
  containers:
  - name: `httpbin`
    image: "kennethreitz/httpbin:{{ .Values.version }}"
    ports:
      - containerPort: 80
```

`values.yaml`
```yaml
environment: "dev"
version: "latest"
```

In the directory of the helm chart, you can render the manifests by running the following command:
`helm template --debug . --values values.yaml --set environment=prod`

The `.` specifies the current directory where the `helm` command line tool can find the templates. We'll replace any variables in the templates, denoted by the double curly braces `{{ }}`, with the corresponding value from the base values file. We can specify more than one values file or pass in the variable values directly. If there are overlapping variables, then the last file takes precedence.

What we get in return is the final output which we can save into a file and submit to the k8s cluster:
```yaml
apiVersion: v1
metadata:
  name: prod-my-app-pod
  labels:
    environment: prod
spec:
  containers:
  - name: `httpbin`
    image: "kennethreitz/httpbin:latest"
    ports:
      - containerPort: 80
```

To install this manifest in our cluster, use either `kubectl` (e.g. kubectl apply -f pod.yaml) or via Helm command line tool (e.g. helm install <release-name> <chart-path>). You can also use CICD tools like ArgoCD to install the helm charts from a Git repository.

## Using public charts

Use public charts by adding the corresponding repo to your Helm repo list: `helm repo add <repo_name> <repo_url>`.

Be sure to update the repo to ensure you have the latest versions available: `helm repo update <repo_name>`.

List the charts in the repo using: `helm search repo <repo_name>`.

For a specific chart, check all of the available versions using `helm search repo <repo_name>/<chart_name> --versions`

Download a helm chart and inspect it locally using: `helm pull <repo_name>/<chart_name> --version <version> --untar`

## Conclusion

In this brief tutorial, we learned that Helm is a package manager for Kubernetes manifests. Helm charts are the packages that collect and templatize Kubernetes manifests.

We covered the basic structure of a Helm chart and how to use the `helm` command line tool to download charts, render charts, and install charts in a k8s cluster.

In the next article, we'll dive into the Helm templating language in depth as well as share some basic tips & tricks.

## References

* [1] https://helm.sh/docs/chart_template_guide/
* [2] https://helm.sh/docs/topics/charts/#the-chartyaml-file
