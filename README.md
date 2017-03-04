# chaoskube
[![Build Status](https://travis-ci.org/linki/chaoskube.svg?branch=master)](https://travis-ci.org/linki/chaoskube)
[![Coverage Status](https://coveralls.io/repos/github/linki/chaoskube/badge.svg?branch=master)](https://coveralls.io/github/linki/chaoskube?branch=master)
[![Go Report Card](https://goreportcard.com/badge/github.com/linki/chaoskube)](https://goreportcard.com/report/github.com/linki/chaoskube)

`chaoskube` periodically kills random pods in your Kubernetes cluster.

## Why

Test how your system behaves under arbitrary pod failures.

## Example

Running it will kill a pod in any namespace every 10 minutes by default.

```console
$ chaoskube
...
INFO[0000] Targeting cluster at https://kube.you.me
INFO[0001] Killing pod kube-system/kube-dns-v20-6ikos
INFO[0601] Killing pod chaoskube/nginx-701339712-u4fr3
INFO[1201] Killing pod kube-system/kube-proxy-gke-earthcoin-pool-3-5ee87f80-n72s
INFO[1802] Killing pod chaoskube/nginx-701339712-bfh2y
INFO[2402] Killing pod kube-system/heapster-v1.2.0-1107848163-bhtcw
INFO[3003] Killing pod kube-system/l7-default-backend-v1.0-o2hc9
INFO[3603] Killing pod kube-system/heapster-v1.2.0-1107848163-jlfcd
INFO[4203] Killing pod chaoskube/nginx-701339712-bfh2y
INFO[4804] Killing pod chaoskube/nginx-701339712-51nt8
...
```

`chaoskube` allows to filter target pods by namespaces, labels and annotations.
[See below](#filtering-targets) for details.

## How

Get `chaoskube` via go get, make sure your current context points to your target cluster and use the `--deploy` flag.

```console
$ go get -u github.com/linki/chaoskube
$ chaoskube --deploy
INFO[0000] Dry run enabled. I won't kill anything. Use --no-dry-run when you're ready.
INFO[0000] Targeting cluster at https://kube.you.me
INFO[0000] Deployed quay.io/linki/chaoskube:v0.4.0
```

By default `chaoskube` will be friendly and not kill anything. When you validated your target cluster you may disable dry-run mode. You can also specify a more aggressive interval and other supported flags for your deployment.

```console
$ chaoskube --interval=1m --no-dry-run --debug --deploy
DEBU[0000] Using current context from kubeconfig at /Users/you/.kube/config.
INFO[0000] Targeting cluster at https://kube.you.me
DEBU[0000] Deploying quay.io/linki/chaoskube:v0.4.0
INFO[0000] Deployed quay.io/linki/chaoskube:v0.4.0
```

Otherwise use the following equivalent manifest file or let it serve as an inspiration.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: chaoskube
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: chaoskube
    spec:
      containers:
      - name: chaoskube
        image: quay.io/linki/chaoskube:v0.4.0
        args:
        - --in-cluster
        - --interval=1m
        - --no-dry-run
        - --debug
```

If you're running in a Kubernetes cluster and want to target the same cluster use the `--in-cluster` flag as shown.

If you want to target a different cluster or want to run it locally provide a valid kubeconfig via `--kubeconfig` and drop the `--in-cluster` flag. By default, it uses your standard kubeconfig path in your home. Whatever is the current context in there will be targeted.

If you want to increase or decrease the amount of chaos change the interval between killings with the `--interval` flag. Alternatively, you can increase the number of replicas of your `chaoskube` deployment.

Remember that `chaoskube` by default kills any pod in all your namespaces, including system pods and itself.

## Filtering targets

However, you can limit the search space of `chaoskube` by providing label, annotations and namespace selectors.

```console
$ chaoskube --labels 'app=mate,chaos,stage!=production'
...
INFO[0000] Filtering pods by labels: app=mate,chaos,stage!=production
```

This selects all pods that have the label `app` set to `mate`, the label `chaos` set to anything and the label `stage` not set to `production` or unset.

You can filter target pods by namespace selector as well.

```console
$ chaoskube --namespaces 'default,testing,staging'
...
INFO[0000] Filtering pods by namespaces: default,staging,testing
```

This will filter for pods in the three namespaces `default`, `staging` and `testing`.

You can also exclude namespaces and mix and match with the label and annotations selectors.

```console
$ chaoskube \
  --labels 'app=mate,chaos,stage!=production' \
  --annotations '!scheduler.alpha.kubernetes.io/critical-pod' \
  --namespaces '!kube-system,!production'
...
INFO[0000] Filtering pods by labels: app=mate,chaos,stage!=production
INFO[0000] Filtering pods by annotations: !scheduler.alpha.kubernetes.io/critical-pod
INFO[0000] Filtering pods by namespaces: !kube-system,!production
```

This further limits the search space of the above label selector by also excluding any pods in the `kube-system` and `production` namespaces as well as ignore all pods that are marked as critical.

The annotation selector can also be used to run `chaoskube` as a cluster addon and allow pods to opt-in to being terminated as you see fit. For example, you could run `chaoskube` like this:

```console
$ chaoskube --annotations 'chaos.alpha.kubernetes.io/enabled=true'
...
INFO[0000] Filtering pods by annotations: chaos.alpha.kubernetes.io/enabled=true
INFO[0000] No victim could be found. If that's surprising double-check your label and namespace selectors.
```

Unless you already use that annotation somewhere, this will initially ignore all of your pods. You could then selectively opt-in individual deployments to chaos mode by annotating their pods with `chaos.alpha.kubernetes.io/enabled=true`.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  template:
    metadata:
      annotations:
        chaos.alpha.kubernetes.io/enabled: "true"
    spec:
      ...
```

## Contributing

Feel free to create issues or submit pull requests.
