---
layout: post
title:  "Apply different values.yaml for different deployments in integration helm chart"
date:   2020-09-27 16:30:00
categories: kubernetes helm helm2
---

Helm is very popular package management application for kubernetes. Helm apps are described as charts which can be deployed in a cluster with helm.
An application is generally made of multiple such packages. We write an integration chart (aka umbrella chart) that combines multiple components and overrides necessary configuration via values.yaml. An example of such application is Wordpress. It contains wordpress frontend and a mariadb backend. The chart is available at [github](https://github.com/bitnami/charts/tree/master/bitnami/wordpress).

But a chart can be deployed in different scenarios where the configuration options should be slightly different. For example, wordpress app deployed on a development cluster does not need lot of resources (CPU, Mem) or multiple replicas. This means, we need to have multiple values.yaml file to deploy the wordpress app in different scenarios.
The downside of keeping multiple values.yaml file (which are identical expect few config changes) is that developers need to maintain both the files to keep them in sync. If there is a change in a common property that is used in both production and development then both values.yaml need to be updated.

This problem can be easily solved by Helm feature of applying multiple values.yaml on a chart installation. So, here we will keep the [standard values.yaml](https://github.com/bitnami/charts/blob/master/bitnami/wordpress/values.yaml) untouched and only the production specific configuration will be maintained in another values.yaml -

```yaml
# production_values.yaml
replicaCount: 3
resources:
  limits:
     cpu: 2
     memory: 3Gi
  requests:
    memory: 1Gi
    cpu: 500m
mariadb:
  resources:
    limits:
      cpu: 2
      memory: 5Gi
    requests:
      cpu: 1
      memory: 2Gi
```

In development environment the standard values.yaml can be used (helm v2 is used).
```shell
# $chart_directory contains the path to the directory of wordpress chart in local disk
helm install my-release $chart_directory
```

For production deployment, wordpress can be installed as following -
```shell
# $chart_directory contains the path to the directory of wordpress chart in local disk
helm install my-release $chart_directory -f $chart_directory/values.yaml -f $chart_directory/production_values.yaml
```
This way Helm will override the standard values with production values, only when second production_values.yaml is passed. Benefit of this approach is "no duplication of properties in multiple values.yaml". It is much earier to maintain.
