# AsciiArtify: MVP

Розгортання застосунку [go-demo-app](https://github.com/chumachenko-vy/go-demo-app) у локальному кластері k3d за допомогою ArgoCD з автоматичною синхронізацією з Git. Вибір k3d обґрунтовано в [Concept](Concept.md).

## Інфраструктура

| Компонент | Значення |
| --- | --- |
| Кластер | k3d v5.9.0, k3s v1.35.5 |
| GitOps | ArgoCD (маніфест `stable`), неймспейс `argocd` |
| Джерело | форк `chumachenko-vy/go-demo-app`, гілка `main`, каталог `helm` |
| Застосунок | Helm-чарт: front, api, ascii, img, data, NATS, Redis, MySQL, Ambassador |
| Неймспейс застосунку | `demo` |
| Синхронізація | automated, `prune`, `selfHeal`, інтервал опитування Git 30 с |

## Встановлення ArgoCD

```bash
k3d cluster create asciiartify
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd patch configmap argocd-cm --type merge \
  -p '{"data":{"timeout.reconciliation":"30s"}}'
kubectl -n argocd rollout restart statefulset argocd-application-controller
kubectl -n argocd rollout restart deployment argocd-repo-server
```

Доступ до UI:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

## Application

Маніфест: [`argocd/go-demo-app.yaml`](../argocd/go-demo-app.yaml).

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/chumachenko-vy/go-demo-app
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

```bash
kubectl apply -f argocd/go-demo-app.yaml
```

## Відомі обмеження

| Проблема | Причина | Рішення |
| --- | --- | --- |
| `ambassador` у `ImagePullBackOff` | образ `quay.io/datawire/ambassador:0.51.2` має маніфест Docker schema 1, який containerd ≥ 2.1 не підтримує | конвертація `skopeo` у v2s2 та імпорт у вузли k3d; потрібно повторювати після перестворення кластера |
| `/ascii/` повертає `upstream connect error` | образ застосунку не обслуговує цей ендпоінт | для демо використовуються `/api/` та `/img/` |

Обхід для Ambassador:

```bash
skopeo copy --format v2s2 docker://quay.io/datawire/ambassador:0.51.2 \
  docker-archive:/tmp/ambassador.tar:quay.io/datawire/ambassador:0.51.2
k3d image import /tmp/ambassador.tar -c asciiartify
kubectl -n demo delete pod -l service=ambassador
```

## Демо: функціонал застосунку

Застосунок доступний через Ambassador (`kubectl -n demo port-forward svc/ambassador 8088:80`). `/api/` повертає ім'я сервісу та версію образу, `/img/` перетворює зображення на ASCII-арт.

![Демо функціоналу](mvp-app.gif)

## Демо: автоматична синхронізація

Зміна `api.replicas` з 1 на 3 у `helm/values.yaml` форку, `git push`, після чого ArgoCD переводить Application у `OutOfSync`, синхронізує його та розгортає три поди `demo-api` без ручного втручання. Далі ручне зменшення реплік через `kubectl scale` повертається до стану з Git завдяки `selfHeal`.

![Демо автосинхронізації](mvp-autosync.gif)
