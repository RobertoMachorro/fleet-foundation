# fleet-foundation
Starting point for a Kubernetes GitOps based infrastructure using [Fleet](https://fleet.rancher.io).

This repo can be used as a stand-alone Fleet instance or as a Github Template, to start your own environment (recommended).
A fresh K8s setup is recommended with only Fleet installed, this way, everything you add is in GitOps!
Either [K3S](https://k3s.io) or [RKE2](https://rke2.io) are loved and tested.

**Note:** Replace in the instructions fleet-local namespace with fleet-default as needed, depending on where your automation is going to run (single cluster? downstream clusters? etc). Refer to the [Fleet Proper Namespace](https://fleet.rancher.io/gitrepo-add#proper-namespace) documentation.

## Setup

1. Bootstrap Fleet on a fresh Kubernetes install using [Helm](https://helm.sh):
```
helm repo add fleet https://rancher.github.io/fleet-helm-charts/
helm -n cattle-fleet-system install --create-namespace --wait fleet-crd fleet/fleet-crd
helm -n cattle-fleet-system install --create-namespace --wait fleet fleet/fleet
```

An alternative is to use a HelmChart operator (built into [K3S](https://k3s.io) and [RKE2](https://rke2.io)). By copying your manifests to a folder, such as */var/lib/rancher/rke2/server/manifests/* in RKE2, you don't even need Helm CLI. 

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cattle-fleet-system
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: fleet-crd
  namespace: cattle-fleet-system
spec:
  repo: https://rancher.github.io/fleet-helm-charts/
  chart: fleet-crd
  targetNamespace: cattle-fleet-system
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: fleet
  namespace: cattle-fleet-system
spec:
  repo: https://rancher.github.io/fleet-helm-charts/
  chart: fleet
  targetNamespace: cattle-fleet-system
```

2. Unless you have a public GitOps repo, you will need to generate an SSH Key Pair for Git/Github Authentication. This allows your Fleet setup to access your Git repository:
```
ssh-keygen -f /path/to/your/id_rsa-gitkey -t rsa -b 4096
```
```
kubectl create secret generic gh-ssh-key -n fleet-local --from-file=ssh-privatekey=/path/to/your/id_rsa-gitkey --from-file=ssh-publickey=/path/to/your/id_rsa-gitkey.pub --type=kubernetes.io/ssh-auth
```

Then add the public key to your Git repo for authentication. In Github, go to your repository's **Settings**, **Deploy keys**, **Add deploy key**. Write access *is not* needed by Fleet.

3. Prior to deploying Fleet setups, you should add any Secrets, TLS certs, connection strings, etc. that are required.

4. Review all **fleet.yaml** files to make sure all values and settings are correct. Commit and push changes.

5. Time to deploy! Apply a manifest similar to this, select the paths you want to include in your setup:
```yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: fleet-foundation-gitops
  namespace: fleet-local
spec:
  repo: git@github.com:RobertoMachorro/fleet-foundation.git
  branch: main
  clientSecretName: gh-ssh-key
  pollingInterval: 60s
  paths:
  - infrastructure/longhorn
  - ... paths ...
```

**Note:** Replace fleet-foundation and RobertoMachorro/fleet-foundation above with your own name and repository path. Also replace the clientSecretName to the one you created in step 2.

6. Check resources:
```
kubectl -n fleet-local get fleet
```
```
kubectl top pod -A --sort-by memory --sum
```

## Installing Rancher

If you opted for the infrastructure/rancher path in this repo, take a couple of extra steps:

1. Configure the IP address and other Rancher options at infrastructure/rancher/fleet.yaml .

2. Commit and let the setup redeploy.

3. When setup completes, access Rancher at https://-your ip-.sslip.io/dashboard/ , using the password revealed by the command:
```
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
```
