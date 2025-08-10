Docs: https://metallb.universe.tf/installation/

Install metallb using helm

```console
helm repo add metallb https://metallb.github.io/metallb
```

```console
helm upgrade --install metallb metallb/metallb --namespace metallb-system --create-namespace
```

Install metallb manifest

```console
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-frr.yaml
```

Once metallb deployed !
In order to assign an IP to the services, MetalLB must be instructed to do so via the IPAddressPool

```console
kubectl apply -f ip-address-pool.yaml
```
