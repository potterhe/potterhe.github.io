
---
layout: post
title: 位编码
tags: bit, coding
---

https://kubernetes.io/cn/docs/tasks/access-application-cluster/configure-access-multiple-clusters/
kubectl config set-cluster jd_rd --server=https://192.168.17.171 --insecure-skip-tls-verify
kubectl config set-credentials experimenter --username=root --password=jumeiops
kubectl config set-context jd_dev --cluster=jd_rd --namespace=jiedian --user=experimenter
kubectl config use-context jd_dev


kubectl create -f https://k8s.io/docs/tasks/access-application-cluster/two-container-pod.yaml


kubectl create configmap example-dove-config --from-file=dove-cm.yaml
