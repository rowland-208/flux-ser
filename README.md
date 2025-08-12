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

# Resources
- https://docs.k3s.io/quick-start
- https://fluxcd.io/flux/get-started
- https://tailscale.com/kb/1017/install
- https://g.co/gemini/share/07d9ccfd2ac1
