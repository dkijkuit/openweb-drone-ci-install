# Installation

# Helm
Install helm, the kubernetes package manager: https://helm.sh/docs/intro/install/
Install kubectl, the command line client tool for Kubernetes: https://kubernetes.io/docs/tasks/tools/
Optional, install `k9s`, one of the best kubernetes cluster managers: https://k9scli.io/topics/install/

## IMPORTANT!!!!!!!!!!!! Adjust to your local IP address, do NOT use localhost or 127.0.0.1
## Add fake 'local' domain names to your `Windows\System32\drivers\etc\hosts` file
``` bash
192.168.1.63 drone.internal
192.168.1.63 gitea.internal
192.168.1.63 demo.internal
192.168.1.63 core.harbor.internal
192.168.1.63 notary.harbor.internal
```

# KinD (Kubernetes in Docker, Powershell)
``` bash
curl.exe -Lo kind-windows-amd64.exe https://kind.sigs.k8s.io/dl/v0.18.0/kind-windows-amd64

Move-Item .\kind-windows-amd64.exe c:\development\tools\kind\kind.exe
```

## Create cluster with ingress
``` bash
kind create cluster --config=kind/cluster-config.yaml --name droneci
kubectl create namespace drone
```

## Create Ingress
``` bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

# Harbor setup (login `admin/Harbor12345`)
``` bash
helm repo add harbor https://helm.goharbor.io
helm install harbor harbor/harbor -f .\Harbor\values.yaml --namespace drone
```

- Go to `http://core.harbor.internal/harbor/projects`
- Create a new project `openweb` and make it public

# Choose: GITEA or GITHUB

## Option 1: GITEA: **Warning**, the webhook randomly fails because both Gitea and Drone are running on localhost. This can be resolved by using a real DNS server instead of using a modified hosts file.

### Deploy Gitea
``` bash
helm repo add gitea-charts https://dl.gitea.io/charts/
helm install gitea gitea-charts/gitea -f .\Gitea\values.yaml --namespace drone
```

### Gitea user
Add a new Gitea user:
Username: `david`
Password: `4ee@k@rEqp4WjFk`

## Create Gitea application
- Go to `http://gitea.internal`
- Create a new repository and push your code
- Go to `Settings` (top right)
- Go to `Applications`
- Scroll to `Manage OAuth2 Applications`
- Add a new OAuth2 Application named `drone-ci`
- Write down the client id and the secret

## Option2: GITHUB
Github must be able to access your local Drone instance using a webhook. Therefore you need to setup a temporary tunnel. Ngrok offers a free service for setting up a secure tunnel, but you can use any other alternative as well.

## Public tunnel
- Go to `https://ngrok.com/` and signup
- Install the `ngrok` binary somewhere in your `$PATH` and start the tunnel:
``` bash
ngrok http 80
```
- The console shows the name of the tunnel, copy the name without the protocol, ie: `0f12-10-109-101-101.ngrok-free.app`

## Setup Github
- Login to `https://github.com`
- Create a new repository and push your code
- Go to `Settings` (top right)
- Scroll down and click on `Developer settings`
- Click on `OAuth apps` and `New OAuth App`
- Name the instance `Drone CI`
- Set its homepage url to the `ngrok` tunnel, example: `https://0f12-10-109-101-101.ngrok-free.app`
- Seth the authorization url to: `https://0f12-10-109-101-101.ngrok-free.app/login`
- Click on `Register application`
- Write down the client id and the secret (only shown once)

# Drone
Edit the `drone/drone-server-values.yaml` and replace the Gitea or Github credentials:

## Option 1: Gitea
Uncomment the following lines and set the corresponding values:
``` properties
DRONE_GITEA_CLIENT_ID: your-gitea-client-id
DRONE_GITEA_CLIENT_SECRET: your-gitea-secret
DRONE_GITEA_SERVER: http://gitea.internal
```

## Option 2: Github
Uncomment the following lines and set the corresponding values:
``` properties
DRONE_GITHUB_CLIENT_ID: your-github-client-id
DRONE_GITHUB_CLIENT_SECRET: your-github-secret
```

## Set the hostname
Also modify the hostname for Drone to your tunnel name:
```
... line 74
- host: 0f12-10-109-101-101.ngrok-free.app

... line 191
  DRONE_SERVER_HOST: "0f12-10-109-101-101.ngrok-free.app"

```

## Deploy Drone
``` bash
helm repo add drone https://charts.drone.io
helm repo update
helm install drone drone/drone -f Drone\drone-server-values.yaml --namespace drone
kubectl apply -f Drone\drone-k9s-runner.yaml -n drone
```

## Run test app
docker run --rm -it -p8080:8080 core.harbor.internal/openweb/drone-demo:0.1