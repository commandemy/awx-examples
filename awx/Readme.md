# AWX Demo Installation on K3S

## Prerequisites

On your workstation you need:

- A working Kubernetes cluster of any kind (k3s, microk8s, Minikube,…)
  - k3s: `curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" sh -`
  - You may have to execute the following command as well: `chown coder /etc/rancher/k3s/k3s.yaml`
- Helm for the official AWX operator Helm chart: `curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`

## Installing a demo setup

For this demonstration, we will use the **AWX operator**, which is the recommended way to install **AWX**.

First install the official Helm chart by using the following command:

```shell
$ helm install -n awx --create-namespace awx-operator https://github.com/ansible/awx-operator/releases/download/1.1.4/awx-operator-1.1.4.tgz --wait
```

Use the above command to install because the commands inside the [GitHub documentation](https://github.com/ansible/awx-operator) do not work as expected!!!

## Creating an AWX instance

To avoid having to tye `-n awx`every time we want to execute something in Kubernetes let's set the context of your cluster:

```shell
$ kubectl config set-context --current --namespace=awx
```

Now use the `awx-demo.yaml` file, which defines our instance, with the following content:

```yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx-demo
spec:
  service_type: nodeport
```

## Creating and supervising the deployment

Run:

```shell
kubectl apply -f awx-demo.yaml
```

to create the instance.

## In order to observe the deployment you can run the following two commands

- Pods: `kubectl get pods -l "app.kubernetes.io/managed-by=awx-operator"` add `-w` to the command to spectate the deployment and press `^C` to stop watching.
- Services: `kubectl get svc -l "app.kubernetes.io/managed-by=awx-operator"`

## How do we **access** this instance of AWX?

### Accessing the instance on k3s

k8s allows us to access deployments by exposing them in two ways:

- Port forwarding
- Ingress

Port forwarding would be the **quick and dirty** version, which is why we will use Ingress instead.
Therefore create a use hte example file called `awx-ingress.yaml`, which will allow us to access it via **http** and **https**.

### `awx-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: awx-ingresss
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
    - host: awx.vscode-XX.@ENV-ANIMAL.commandemy.training # set the hostname
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: awx-demo-service
                port:
                  number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: awx-tls-ingresss
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  rules:
    - host: awx.vscode-XX.@ENV-ANIMAL.commandemy.training # set the hostname
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: awx-demo-service
                port:
                  number: 80
```

## Visiting the UI

Make sure to change the host within the file to resemble your workstation, then apply it using:

```shell
$ kubectl apply -f awx-ingress.yaml
```

You can the visit the UI by copying the host into the search bar of your browser.

By default, the admin user is admin and the password is available in the <resourcename>-admin-passwordsecret. To retrieve the admin password, run:

```shell
$ kubectl get secret awx-demo-admin-password -o jsonpath="{.data.password}" | base64 --decode ; echo
```
