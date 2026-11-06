# Sherlock Holmes - The Adventure of the Vanishing Connections

A Cilium and Hubble troubleshooting demo in the `baker-st` namespace.

| Character | App label |
| --- | --- |
| Sherlock Holmes | `app.kubernetes.io/name=sherlock` |
| Dr Watson | `app.kubernetes.io/name=watson` |
| Professor Moriarty | `app.kubernetes.io/name=moriarty` |
| Inspector Lestrade | `app.kubernetes.io/name=lestrade` |
| Miss Ada Moran | `app.kubernetes.io/name=ada` |

Run all commands from this folder.

## Deploy the apps

```sh
kubectl apply -f 00-namespace.yaml
```

```sh
kubectl apply -f deployments/01-sherlock.yaml \
  -f deployments/02-watson.yaml \
  -f deployments/03-moriarty.yaml \
  -f deployments/04-lestrade.yaml \
  -f deployments/05-ada-v1.yaml
```

```sh
kubectl apply -f policies/01-cluster-dns-egress.yaml \
  -f policies/02-sherlock-egress-v1-watson.yaml \
  -f policies/03-moriarty-ingress.yaml \
  -f policies/04-ada-egress.yaml \
  -f policies/05-watson-egress.yaml
```

```sh
kubectl -n baker-st get pods --show-labels
```

## Hubble

Hubble Relay, for `hubble observe`

```sh
kubectl -n kube-system port-forward service/hubble-relay 4245:80
```

or

```sh
cilium hubble port-forward
```

set as default server the IPv4 address, because IPv6 localhost is slow

```sh
hubble config set server 127.0.0.1:4245
```

## Act 0 - Watson asks Sherlock

Observe

```sh
hubble observe \
  --from-label 'k8s:app.kubernetes.io/name=watson' \
  --to-label 'k8s:app.kubernetes.io/name=sherlock' \
  --follow
```

Watson sends a request to Sherlock

```sh
kubectl -n baker-st exec deploy/watson -- wget -q -T 5 -O - http://sherlock:8080/
```

## Act 1 - Ada cannot find Watson (DNS drop)

Observe

```sh
hubble observe \
  --from-label 'k8s:app.kubernetes.io/name=ada' \
  --port 53 \
  --verdict DROPPED \
  --follow
```

Ada sends a request to Watson (fails)

```sh
kubectl -n baker-st exec deploy/ada -- wget -q -T 5 -O - http://watson:8080/
```

Fix: give Ada the DNS label

```sh
kubectl apply -f deployments/05-ada-v2.yaml
kubectl -n baker-st rollout status deployment/ada
```

Ada sends a request to Watson again

```sh
kubectl -n baker-st exec deploy/ada -- wget -q -T 5 -O - http://watson:8080/
```

## Act 2 - Sherlock cannot reach Lestrade (in-cluster drop)

Observe

```sh
hubble observe \
  --from-label 'k8s:app.kubernetes.io/name=sherlock' \
  --to-label 'k8s:app.kubernetes.io/name=lestrade' \
  --verdict DROPPED \
  --follow
```

Sherlock sends a request to Lestrade (fails)

```sh
kubectl -n baker-st exec deploy/sherlock -- wget -q -T 5 -O - http://lestrade:8080/
```

Fix: allow Sherlock to reach Lestrade

```sh
kubectl apply -f policies/02-sherlock-egress-v2-lestrade.yaml
```

Sherlock sends a request to Lestrade again

```sh
kubectl -n baker-st exec deploy/sherlock -- wget -q -T 5 -O - http://lestrade:8080/
```

## Act 3 - Sherlock cannot read The Strand (external drop)

Observe

```sh
hubble observe \
  --from-label 'k8s:app.kubernetes.io/name=sherlock' \
  --to-identity world \
  --verdict DROPPED \
  --follow
```

Sherlock sends a request to strandmag.com (fails)

```sh
kubectl -n baker-st exec deploy/sherlock -- wget -q -T 5 -O /dev/null -S https://strandmag.com/
```

Fix: allow Sherlock to reach strandmag.com

```sh
kubectl apply -f policies/02-sherlock-egress-v3-strandmag.yaml
```

Sherlock sends a request to strandmag.com again

```sh
kubectl -n baker-st exec deploy/sherlock -- wget -q -T 5 -O /dev/null -S https://strandmag.com/
```

Observe the success

```sh
hubble observe \
  --from-label 'k8s:app.kubernetes.io/name=sherlock' \
  --to-fqdn 'strandmag.com' \
  --follow
```

## Act 4 - Ada reports to Moriarty

Who talks to Moriarty?

```sh
hubble observe \
  --to-label 'k8s:app.kubernetes.io/name=moriarty' \
  --since 5m
```

What does Ada send to Moriarty?

```sh
hubble observe \
  --from-label 'k8s:app.kubernetes.io/name=ada' \
  --to-label 'k8s:app.kubernetes.io/name=moriarty' \
  --protocol http \
  --follow
```
