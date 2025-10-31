# cilium-demo

based on [Getting Started with the Star Wars Demo](https://docs.cilium.io/en/stable/gettingstarted/demo/) from cilium's official documentation

## Deploy the apps

```
kubectl create namespace demo
kubectl -n demo apply -f https://raw.githubusercontent.com/kkantonop/cilium-demo/refs/heads/main/apps.yaml
```

## Hubble UI

```
kubectl -n kube-system port-forward services/hubble-ui 8081:http
```

## Request Landing

xwing requests landing
```
kubectl -n demo exec xwing -- curl -s -XPOST deathstar.demo.svc.cluster.local/v1/request-landing
```

tiefighter requests landing
```
kubectl -n demo exec tiefighter -- curl -s -XPOST deathstar.demo.svc.cluster.local/v1/request-landing
```

## Restrict Access to Deathstar Only to Empire

```
kubectl -n demo apply -f https://raw.githubusercontent.com/kkantonop/cilium-demo/refs/heads/main/deathstar-ingress.yaml
```

## Sabotage Attempt

```
kubectl -n demo exec tiefighter -- curl -s -XPUT deathstar.demo.svc.cluster.local/v1/exhaust-port
```

## Prevent Sabotage

```
kubectl -n demo delete -f https://raw.githubusercontent.com/kkantonop/cilium-demo/refs/heads/main/deathstar-ingress.yaml
kubectl -n demo apply -f https://raw.githubusercontent.com/kkantonop/cilium-demo/refs/heads/main/l7.yaml
```

## Block Alliance Comms

```
kubectl -n demo apply -f https://raw.githubusercontent.com/kkantonop/cilium-demo/refs/heads/main/alliance-deny-egress.yaml
```

## Alliance Wants to Get Tatoine Coords to Escape

```
kubectl -n demo exec xwing -- curl -vs https://swapi.dev/api/planets/1
```

## Alliance Last Hope?

```
kubectl -n demo apply -f https://raw.githubusercontent.com/kkantonop/cilium-demo/refs/heads/main/alliance-fqdn.yaml
```


---
Get the cilium pod running on the same node
```
kubectl -n kube-system get pods -l k8s-app=cilium -o wide
```

Get endpoint list
```
kubectl -n kube-system exec <cilium-pod-name> -- cilium-dbg endpoint list
```

Monitor for "drop" verdict
```
kubectl -n kube-system exec <cilium-pod-name> -- cilium-dbg monitor --type=drop
```

Monitor for "l7" verdict
```
kubectl -n kube-system exec <cilium-pod-name> -- cilium-dbg monitor --type=l7
```
