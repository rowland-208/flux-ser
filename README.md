# Goal

I want a simple cicd setup for building and serving apps on a private network.

The technology I chose for this project:
- k3s: a simple, lightweight kubernetes distro perfect for homelabs
- fluxcd: gitops tool for managing kubernetes, almost went with argocd, but flux seems to be more popular
- tailscale: easy to set local network solution backed by wireguard

I'm running all of this on a small Ubuntu box that I SSH into.

# Lightning guide for flux setup

Install k3s

```
curl -sfL https://get.k3s.io | sh -
```

Install flux

```
curl -s https://fluxcd.io/install.sh | sudo bash
```

Pick a repo you own and create a fine grained personal access token in github.
The token needs read access to metadata,
and read and write access to administration and code.
Configure k3s and flux in ~/.bashrc

```
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export GITHUB_FLUX_REPO=<your-flux-repo-name>
```

Check the flux and k3s are working together

```
sudo -E flux check --pre
```

Connect flux and github

```
sude -E flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=$GITHUB_FLUX_REPO \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal
```

Clone the repo flux is using

```
git clone https://github.com/$GITHUB_USER/fleet-infra
cd fleet-infra
```

Use flux to create a podman deployment configuration.
This is like hello world.
A file ./clusters/my-cluster/podinfo-source.yaml will be created

```
flux create source git podinfo \
  --url=https://github.com/stefanprodan/podinfo \
  --branch=master \
  --interval=1m \
  --export > ./clusters/my-cluster/podinfo-source.yaml
```

Commit the file and push

```
git add -A && git commit -m "Add podinfo GitRepository"
git push
```

Deploy the app

```
sudo -E flux create kustomization podinfo \
  --target-namespace=default \
  --source=podinfo \
  --path="./kustomize" \
  --prune=true \
  --wait=true \
  --interval=30m \
  --retry-interval=2m \
  --health-check-timeout=3m \
  --export > ./clusters/my-cluster/podinfo-kustomization.yaml
```

then

```
git add -A && git commit -m "Add podinfo Kustomization"
git push
```

Check on your deployment

```
flux get kustomizations --watch
```

and 

```
kubectl -n default get deployments,services
```

# Connecting with tailscale

Setup tailscale.
On your main device follow setup guide.
On the linux box running k3s and flux run

```
curl -fsSL https://tailscale.com/install.sh | sh
```

then

```
sudo tailscale up
```

and follow the link to authenticate.

Now in the tailscale menu on your main device you can get the name of the linux box.

tailscale menu -> network devices -> my devices -> <device-name>

Configure k3s to allow network traffic.
Create a file clusters/my-cluster/podinfo-ingress.yaml
and edit the file

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: podinfo-ingress
  namespace: default
spec:
  rules:
  - host: <device-name>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: podinfo
            port:
              number: 9898
```

You'll copy and paste your <device-name> from tailscale.

What's happening here is tailscale maps your device name to ip.
And we are configuring k3s to allow traffic by device name and port.
Traffic can only get through on our tailscale network.

Now you should be able to open http://<device-name> in a browser and see the podinfo ui.

# Modifications for gitea

On a fresh cluster install gitea with helm

```
sudo -E helm install gitea gitea-charts/gitea
```

Now that gitea is set up modify it to allow ingress.
Grab values

```
sudo -E helm show values gitea-charts/gitea > values.yaml
```

Then edit them to enable ingress, replace with your server hostname

```
ingress:
  enabled: true
  className: ""
  pathType: Prefix
  annotations: {}
  hosts:
    - host: <server-hostname>
      paths:
        - path: /
  tls: []
```

Next update to reflect these new values

```
sudo -E helm upgrade gitea gitea-charts/gitea -f values.yaml
```

Now you should be able to access gitea.
Go to http://<server-hostname> in a browser.
Create a new account.
Give it an insecure password on purpose; in the next step you will potentially expose this password.

Now we need to bootstrap flux.
I tried these options with a personal access tokens as discussed in the flux docs for gitea but it failed

```
sudo -E  flux bootstrap gitea --owner=spacetracks --repository=flux --private=false --personal=true --path=./clusters/my-cluster --hostname=spacetracks-ser --insecure-skip-tls-verify
```

The reason is traefik supplies a default cert that doesn't match my hostname.
Then I found this option that worked

```
sudo -E flux bootstrap git --url http://spacetracks-ser/spacetracks/flux --username=spacetracks --password=password --allow-insecure-http=true --token-auth=true --branch=main
```

It's incredibly insecure, using http and exposing the password directly.
But given the server is walled off in a vpn its fine.
The username and password are visible but they aren't part of the security.

# Routing

I spent way too long trying to get gitea to route to an address other than the magic dns address provided by tailscale.
I eventually gave up and went with a non-default port,
but I learned a lot along the way.
The final values.yaml for gitea
```
ingress:
  enabled: false
service:
  http:
    type: NodePort
    port: 3000
    nodePort: 30080
  ssh:
    type: NodePort
    port: 22
    nodePort: 30022
```

K3s will route port 30080 to gitea port 3000.
It doesn't pass through ingress.

The three approaches considered are:
* Path
* Subdomain
* Port

Path almost worked but ultimately failed because gitea can't call its own API.
When it makes the call the hostname gets replaced with localhost.
Then it makes the call to localhost/gitea which doesn't exist.
On the other hand when the user navigates to <magicdns>/gitea traefik correctly routes.

Subdomain is a good approach but just doesn't work with tailscale magicdns.
It's a frequently requested feature.
If not using tailscale for dns you can make subdomains work but you have to configure it yourself.

Ports are arcane but just work.
DNS doesn't touch them.

# Gitea runners

I ran into a lot of networking issues trying to set up gitea actions runners.
It turns out the way I had gitea configured made networking between pods a nightmare.
Setting this in the values.yaml file uses the internal network instead of routing out through tailscale.
If this is left default it will inherit value from ingress.
```
       72 +  # Gitea instance URL - use internal Kubernetes service
       73 +  giteaRootURL: "http://gitea-http:3000"
```

# Gitea ssh

I ran into more networking issues getting ssh working.
One of the issues was silly and not networking related,
I created an ssh key as root and tried using it as user, lesson learned.
The other is the gitea ssh service needs to be exposed to the cluster,
nodeport is the easiest option.
In the gitea values:
```
ssh:
    annotations: {}
    clusterIP: null
    externalIPs: null
    externalTrafficPolicy: null
    hostPort: null
    ipFamilies: null
    ipFamilyPolicy: null
    labels: {}
    loadBalancerClass: null
    loadBalancerIP: null
    loadBalancerSourceRanges: []
    nodePort: 30022
    port: 22
    type: NodePort
```
Also note the ssh config needs to specify the port

```
Host spacetracks-ser
  HostName spacetracks-ser
  Port 30022
  User git
  IdentityFile ~/.ssh/gitea_runner_test
  StrictHostKeyChecking no
```

# Static IP

I ran into issues when my IP rotated.
Cluster nodes should have static IP as a rule of thumb.
It's possible to swap out the IPs in k3s config after the fact,
but way more painful than setting static IP.
This means if you move your cluster to a new network
you will have to manually update the IPs.
But you would have to do something similar if you use dynamic IPs.

First get network info
```
nmcli connection show
```

Then set it
```
sudo nmcli connection modify "LWDavis" ipv4.method manual ipv4.addresses 192.168.1.170/24 ipv4.gateway 192.168.1.1 ipv4.dns 8.8.8.8,8.8.4.4
```

Now reactivate
```
sudo nmcli connection up "LWDavis"
```

Replace "LWDavis" with your network name. And replace 192.168.1.70/24 with the IP you want; and 192.168.1.1 with your router IP. 


# Resources
- https://docs.k3s.io/quick-start
- https://fluxcd.io/flux/get-started
- https://fluxcd.io/flux/cmd/flux_bootstrap_gitea
- https://tailscale.com/kb/1017/install
- https://g.co/gemini/share/07d9ccfd2ac1
