### Create Kind cluster

##### Preparation

Make MetalLB IPs directly reachable from any device on your LAN, using macvlan so that the KinD cluster nodes appear like real hosts on the network.

###### Get LAN interface name

Find your LAN NIC (for example eth0, enp3s0, or wlp3s0 for WI-FI)

We'll call it `LAN_IFACE` below, use `ip a` or `nmcli connection show` command:

```console
$ ip a

1: enp0s41f2d2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether AA:c3:54:14:2c:28 brd ff:ff:ff:ff:ff:ff
    altname enx8ed481612c34
    inet 192.168.1.6/24 brd 192.168.1.255 scope global dynamic noprefixroute enp0s41f2d2
       valid_lft 70590sec preferred_lft 70590sec
2: wlp1e31f1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether fc:52:ac:89:23:51 brd ff:ff:ff:ff:ff:ff
    altname wlxfe58ca453121
    inet 192.168.1.7/24 brd 192.168.1.255 scope global dynamic noprefixroute wlp1e31f1
       valid_lft 86621sec preferred_lft 86621sec

```

OR

```console
$ nmcli connection show
NAME                        UUID                                  TYPE      DEVICE
Connexion filaire 1         xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  ethernet  ethernet-interface-name
Wifi-NAME 1                 xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  wifi      wifi-interface-name
```

###### Create a macvlan network in Docker

Replace `LAN_IFACE` with your actual interface name:

```console
docker network create -d macvlan \
    --subnet=192.168.1.0/24 \
    --gateway=192.168.1.1 \
    -o parent=LAN_IFACE kind-macvlan
```

- 192.168.1.0/24 -> your LAN range
- 192.168.1.1 -> your router's IP

###### Create the kind cluster

```console
king create cluster --config ./cluster.yaml
```

#### Attach kind nodes to the macvlan network

After the cluster is running

```console
docker network connect kind-macvlan kind-control-plane \
 && docker network connect kind-macvlan kind-control-plane2 \
 && docker network connect kind-macvlan kind-control-plane3 \
 && docker network connect kind-macvlan kind-worker \
 && docker network connect kind-macvlan kind-worker2
docker network connect kind-macvlan kind-worker3
```

(_Adjust worker names if you used a different ones_)

###### Install MetalLB

Follow instructions in [metallb/README.md](./metallb/READMED.md)

###### Add macvlan interface to host (so host can reach metalLB IP range[.240 - .250])

Pick an unused IP outside MetalLB's range -- e.g., 192.168.1.200:

```console
sudo ip link add macvlan0 link LAN_IFACE type macvlan mode bridge \
  && sudo ip addr add 192.168.1.200/24 dev macvlan0 \
  && sudo ip link set macvlan0 up
```

<detail id="markdown">
<summary>Diagram showing how this setup works</summary>

```diagram
┌─────────────────────────────────────────────────────────────────┐
│                    Your LAN (192.168.1.0/24)                    │
│                                                                 │
│   ┌───────────────────────────────────────────────────────┐     │
│   │                     Router/Gateway                    │     │
│   │                     192.168.1.1                       │     │
│   └───────▲───────────────────▲────────────────────▲──────┘     │
│           │                   │                    │            │
│           │                   │                    │            │
│           │                   │                    │            │
│   ┌───────┴───────┐  ┌────────┴─────────┐  ┌───────┴──────┐     │
│   │ Host Machine  │  │ Other LAN Device │  │ Another Host │ ... │
│   │ 192.168.1.7   │  │ 192.168.1.50     │  │ 192.168.1.51 │     │
│   │               │  │                  │  │              │     │
│   │ + macvlan0    │  │                  │  │              │     │
│   │ 192.168.1.200 │  │                  │  │              │     │
│   └─────┬─────────┘  └──────────▲───────┘  └──────▲───────┘     │
│         │                       │                 │             │
│         │   ARP + L2 traffic    │                 │             │
│         │                       │                 │             │
│   ┌─────▼───────────────────────┴─────────────────┴───────────┐ │
│   │         Kind Cluster (Docker containers)                  │ │
│   │ ┌─────────────────────────┐  ┌─────────────────────────┐  │ │
│   │ │ kind-control-plane      │  │ kind-worker             │  │ │
│   │ │ on kind-macvlan network │  │ on kind-macvlan network │  │ │
│   │ │ Responds to .240–.250   │  │ Responds to .240–.250   │  │ │
│   │ └─────────────────────────┘  └─────────────────────────┘  │ │
│   │                                                           │ │
│   │          MetalLB (Layer 2 mode) announces ARP:            │ │
│   │           "192.168.1.240 → MAC of this worker"            │ │
│   └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Flow explanation:

1. **MetalLB L2 mode** runs inside the KinD containers (macvlan network) and sends ARP replies directly to the LAN:
   1. _"Hey everyone .240 is at my MAC address."_
2. **Other LAN devices** hear the ARP reply and send packets directly to the container's macvlan interface.
3. Your host machine normally wouldn't see the macvlan network (macvlan isolation rule), but macvlan0 (With IP .200) puts it in the same broadcast domain so it can also talk to .240
4. No `extraPortMappings` are needed -- any port on .240 is open according to the kubernetes Service definition.

</detail>

###### Install Nginx Ingress Controller

```console
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

Check: the following command should return something like:

```console
$ helm search repo ingress-nginx

NAME                       	CHART VERSION	APP VERSION	DESCRIPTION
ingress-nginx/ingress-nginx	4.12.0       	1.12.0     	Ingress controller for Kubernetes using NGINX a...
```

Then deploy the ingress controller with the following command:

```console
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

###### Optional (Customize Ingress controller)

If you want, you can customize your ingress controller using the chart values of that ingress controller

To get the chart release values, run the following command

```console
helm show values ingress-nginx ingress-nginx/ingress-nginx > ingress-nginx-values.yaml
```

Once edited, you can deploy the ingress controller using the `--values` options

```console
helm upgrade --install ingress-nginx ingress-nginx \
  --values ingress-nginx-values.yaml \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

### Kubernetes dashboard

[Official Docs here](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)

Add kubernetes dashboard helm repository

```console
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
```

Deploy the Helm release of kubernetes dashboard

```console
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard \
    --create-namespace --namespace kubernetes-dashboard
```

After the dashboard chart deployed, we can create needed component (service-account, Secret, ingress etc.) by running the following command:

Create the ingress component for the dashboard

```console
kubectl apply -f dashboard/

```

After Secret is created, we can execute the following command to get the token which is saved in the Secret:

```console
kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath="{.data.token}" | base64 -d
```

---

Now, you should be able to access the dashboard on [https://dashboard.192.168.1.240.nip.io/#/login](https://dashboard.192.168.1.240.nip.io/#/login)

OR

Use port-forward as bellow:

```console
kubectl -n kubernetes-dashboard port-forward svc/k8s-dashboard-kong-proxy 8443:443
```

Then go to [https://localhost:8443/#/login](https://localhost:8443/#/login)

---

### Generate an auto signed TLS certificate

[Find the Docs here](https://kubernetes.github.io/ingress-nginx/user-guide/tls/)

```console
CERT_FILE=cert.pem
KEY_FILE=key.pem
HOST=dashboard.192.168.1.240.nip.io

openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ${KEY_FILE} -out ${CERT_FILE} -subj "/CN=${HOST}/O=${HOST}" -addext "subjectAltName = DNS:${HOST}"
```

Then create the secret:

```console
CERT_NAME=dashboard-tls-cert
kubectl create secret tls ${CERT_NAME} --key ${KEY_FILE} --cert ${CERT_FILE}
```
