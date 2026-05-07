
# AsciiArtify — MVP Deployment with ArgoCD

## Мета

Після успішного завершення PoC команда AsciiArtify переходить до розгортання MVP.

Мета цього етапу — розгорнути demo application у Kubernetes за допомогою ArgoCD та налаштувати автоматичну синхронізацію з Git repository.

ArgoCD використовується як GitOps-інструмент, який відслідковує зміни у Git repository та автоматично застосовує їх у Kubernetes cluster.

## Source Repository

Для MVP використовується repository:

```text

https://github.com/den-vasyliev/go-demo-app

```

У репозиторії використовується Helm chart, який знаходиться у директорії:

```text

helm

```

## Infrastructure

Для розгортання використовувалась така інфраструктура:

```text

Kubernetes cluster: minikube

Environment: GitHub Codespaces

GitOps tool: ArgoCD

Target namespace: demo

Application name: go-demo-app

```

## ArgoCD Application

Для розгортання MVP було створено ArgoCD Application `go-demo-app`.

Application configuration:

```yaml

apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:

  name: go-demo-app

  namespace: argocd

spec:

  project: default

  source:

    repoURL: https://github.com/den-vasyliev/go-demo-app

    targetRevision: HEAD

    path: helm

  destination:

    server: https://kubernetes.default.svc

    namespace: demo

  syncPolicy:

    automated:

      prune: true

      selfHeal: true

    syncOptions:

      - CreateNamespace=true

```

Application was applied with:

```bash

kubectl apply -f go-demo-app-argocd.yaml

```

## ArgoCD UI Access

ArgoCD was running inside GitHub Codespaces, so the UI was exposed using `kubectl port-forward`.

```bash

kubectl port-forward svc/argocd-server -n argocd 18080:80 --address 0.0.0.0

```

ArgoCD UI was available through GitHub Codespaces forwarded port:

```text

https://organic-guacamole-4w979q4prwvfq7pp-18080.app.github.dev

```

During testing, ArgoCD UI had a session issue through Codespaces proxy. API authentication worked correctly, but UI requests to `/api/v1/applications` and `/api/v1/clusters` returned `401`.

The issue was temporarily resolved by enabling anonymous access with admin permissions for the local demo environment:

```bash

kubectl -n argocd patch configmap argocd-cm \

  --type merge \

  -p '{"data":{"users.anonymous.enabled":"true"}}'

kubectl -n argocd patch configmap argocd-rbac-cm \

  --type merge \

  -p '{"data":{"policy.default":"role:admin"}}'

kubectl -n argocd rollout restart deployment argocd-server

```

This workaround was used only for the local Codespaces demo environment.

After the demo, anonymous admin access should be disabled:

```bash

kubectl -n argocd patch configmap argocd-cm \

  --type merge \

  -p '{"data":{"users.anonymous.enabled":"false"}}'

kubectl -n argocd patch configmap argocd-rbac-cm \

  --type merge \

  -p '{"data":{"policy.default":"role:readonly"}}'

kubectl -n argocd rollout restart deployment argocd-server

```

## ArgoCD Sync Status

After creating the ArgoCD Application, ArgoCD automatically pulled the application manifests from Git and started synchronization.

Verification command:

```bash

kubectl get applications -n argocd

```

Initial result:

```text

NAME          SYNC STATUS   HEALTH STATUS

go-demo-app   Synced        Progressing

```

After fixing the outdated Ambassador image issue, all workloads became healthy:

```text

NAME          SYNC STATUS   HEALTH STATUS

go-demo-app   OutOfSync     Healthy

```

The `OutOfSync` status appeared because the Ambassador image was manually patched in the cluster as a workaround for an obsolete image in the upstream Helm chart.

## Deployed Kubernetes Resources

The application was deployed into the `demo` namespace.

Verification command:

```bash

kubectl get pods -n demo

```

Example result:

```text

NAME                                      READY   STATUS    RESTARTS   AGE

ambassador-897f677db-gkkgp               1/1     Running   0          4m

cache-bd77dcdf6-phh2j                    1/1     Running   0          21m

db-7b645848df-vffq7                      1/1     Running   0          21m

go-demo-app-api-569d9dc695-x4hq4         1/1     Running   0          21m

go-demo-app-ascii-6d748b467d-b8vnl       1/1     Running   0          21m

go-demo-app-data-7966f7c5f-gjqqs         1/1     Running   0          21m

go-demo-app-front-5c6574c476-dzpbw       1/1     Running   0          21m

go-demo-app-img-666b66fd55-nrnkz         1/1     Running   0          21m

go-demo-app-nats-0                       3/3     Running   0          21m

go-demo-app-nats-box-599ff94cb7-bpxkn    1/1     Running   0          21m

```

Services in the `demo` namespace:

```bash

kubectl get svc -n demo

```

Example result:

```text

NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)

ambassador            NodePort    10.105.207.33    <none>        80:31926/TCP

ambassador-admin      ClusterIP   10.103.252.130   <none>        8877/TCP

cache                 ClusterIP   10.107.66.100    <none>        6379/TCP

db                    ClusterIP   10.98.249.26     <none>        3306/TCP

go-demo-app-api       ClusterIP   10.111.55.54     <none>        80/TCP

go-demo-app-ascii     ClusterIP   10.96.36.90      <none>        80/TCP

go-demo-app-data      ClusterIP   10.109.46.195    <none>        80/TCP

go-demo-app-front     ClusterIP   10.103.46.139    <none>        80/TCP

go-demo-app-img       ClusterIP   10.106.56.247    <none>        80/TCP

go-demo-app-nats      ClusterIP   None             <none>        4222/TCP,6222/TCP,8222/TCP,7777/TCP,7422/TCP,7522/TCP

```

## Application Access

Because the Kubernetes cluster was running inside GitHub Codespaces, application access was provided with `kubectl port-forward`.

Frontend service was exposed with:

```bash

kubectl port-forward svc/go-demo-app-front -n demo 8089:80 --address 0.0.0.0

```

Application URL in GitHub Codespaces:

```text

https://organic-guacamole-4w979q4prwvfq7pp-8089.app.github.dev/

```

Ambassador service was exposed with:

```bash

kubectl port-forward svc/ambassador -n demo 8088:80 --address 0.0.0.0

```

Application URL:

```text

https://organic-guacamole-4w979q4prwvfq7pp-8088.app.github.dev/

```

## Issue Found During Deployment

During deployment, the original Helm chart used an outdated Ambassador image:

```text

quay.io/datawire/ambassador:0.51.2

```

Kubernetes could not pull this image because it uses an obsolete Docker manifest format:

```text

unsupported manifest media type:

application/vnd.docker.distribution.manifest.v1+prettyjws

```

The pod status was:

```text

ImagePullBackOff

```

Example diagnostic command:

```bash

kubectl describe pod -n demo ambassador-7d54f5b9d-n5nhx

```

Relevant event:

```text

Failed to pull image "quay.io/datawire/ambassador:0.51.2":

unsupported manifest media type and no default available:

application/vnd.docker.distribution.manifest.v1+prettyjws

```

To complete the MVP deployment, the Ambassador image was temporarily replaced with a newer compatible image:

```bash

kubectl set image deployment/ambassador \

  ambassador=docker.io/datawire/ambassador:1.14.4 \

  -n demo

```

After this change, the Ambassador pod started successfully:

```text

ambassador-897f677db-gkkgp   1/1   Running

```

This workaround made the application workloads healthy, but the ArgoCD Application became `OutOfSync`, because the live Kubernetes state differed from the desired state stored in Git.

Correct long-term GitOps solution:

1. Fork the original repository.

2. Update the outdated Ambassador image in the Helm chart.

3. Point ArgoCD Application to the forked repository.

4. Re-enable `selfHeal`.

5. Let ArgoCD synchronize the fixed configuration from Git.

## Auto-Sync Demonstration

ArgoCD was configured with automated synchronization:

```yaml

syncPolicy:

  automated:

    prune: true

    selfHeal: true

```

This means:

- ArgoCD automatically tracks changes in the Git repository.

- ArgoCD applies changes to the Kubernetes cluster.

- `prune: true` removes resources that were removed from Git.

- `selfHeal: true` restores the desired state if cluster resources are manually changed.

During testing, ArgoCD automatically detected drift when the Ambassador image was manually changed. It attempted to restore the original image from Git, which confirmed that the self-healing mechanism was working.

The self-healing behavior was visible when ArgoCD reverted the manually patched Ambassador deployment back to the original image from Git:

```text

quay.io/datawire/ambassador:0.51.2

```

This confirmed that ArgoCD continuously compares the live cluster state with the desired state stored in Git.

## Demo

Demo materials:

```text

TODO: Add video demo link here

```

Recommended demo content:

1. ArgoCD UI with `go-demo-app`.

2. Git repository source configuration.

3. Automated sync configuration.

4. Kubernetes resources in the `demo` namespace.

5. Running application in browser.

6. Demonstration of ArgoCD detecting and synchronizing changes.

## Useful Commands

Check ArgoCD pods:

```bash

kubectl get pods -n argocd

```

Check ArgoCD Application:

```bash

kubectl get applications -n argocd

kubectl describe application go-demo-app -n argocd

```

Check deployed workloads:

```bash

kubectl get pods -n demo

kubectl get svc -n demo

```

Check Ambassador image:

```bash

kubectl get deployment ambassador -n demo \

  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

```

Port-forward frontend:

```bash

kubectl port-forward svc/go-demo-app-front -n demo 8089:80 --address 0.0.0.0

```

Port-forward ArgoCD UI:

```bash

kubectl port-forward svc/argocd-server -n argocd 18080:80 --address 0.0.0.0

```

## Result

The MVP application was successfully deployed to Kubernetes using ArgoCD.

ArgoCD Application `go-demo-app` was created and configured to track:

```text

https://github.com/den-vasyliev/go-demo-app

```

with source path:

```text

helm

```

The application was deployed to namespace:

```text

demo

```

ArgoCD successfully synchronized the application manifests from Git and deployed the required Kubernetes resources.

A deployment issue was found in the upstream Helm chart because of an obsolete Ambassador image. The issue was diagnosed and temporarily fixed by replacing the image with a newer compatible version. This allowed the application workloads to become healthy and demonstrated troubleshooting of a real deployment problem in a GitOps workflow.

