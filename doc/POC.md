
# AsciiArtify — ArgoCD Proof of Concept

## 1. Мета PoC

Мета цього Proof of Concept — перевірити технічну можливість розгортання GitOps-системи для проєкту **AsciiArtify** у Kubernetes-кластері.

У межах PoC потрібно:

- розгорнути Kubernetes-кластер;

- встановити ArgoCD;

- перевірити стан компонентів ArgoCD;

- налаштувати доступ до графічного інтерфейсу ArgoCD;

- описати інструкцію для команди щодо доступу до ArgoCD UI.

Для GitOps-системи використовується **ArgoCD**.

---

## 2. Обраний Kubernetes-інструмент

Для локального PoC використовується **k3d**.

k3d — це легкий інструмент для запуску Kubernetes-кластерів на базі k3s у Docker-контейнерах.

Причини вибору k3d:

- швидке створення локального Kubernetes-кластера;

- менше споживання ресурсів у порівнянні з повноцінними Kubernetes-кластерами;

- зручний для локального PoC та MVP;

- підтримує стандартні Kubernetes-інструменти;

- добре підходить для демонстрації GitOps-процесу.

---

## 3. Передумови

Перед початком потрібно мати встановлені:

- Docker або сумісний container runtime;

- k3d;

- kubectl;

- GitHub Codespaces або локальне середовище розробки.

Перевірка інструментів:

```bash

docker --version

k3d version

kubectl version --client

```

---

## 4. Створення Kubernetes-кластера

Створюємо локальний Kubernetes-кластер:

```bash

k3d cluster create asciiartify

```

Перевіряємо, що кластер створений і доступний:

```bash

kubectl cluster-info

kubectl get nodes

```

Очікуваний результат:

```text

Kubernetes node має статус Ready

```

---

## 5. Встановлення ArgoCD

Створюємо namespace для ArgoCD:

```bash

kubectl create namespace argocd

```

Встановлюємо ArgoCD з офіційного manifest-файлу:

```bash

kubectl create -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

```

> Примітка: для цього PoC використовується `kubectl create`, а не `kubectl apply`, оскільки при використанні `apply` для CRD може виникати помилка `metadata.annotations: Too long`.

Перевіряємо статус pod-ів ArgoCD:

```bash

kubectl get pods -n argocd

```

Очікуваний результат:

```text

Усі pod-и ArgoCD мають статус Running

```

---

## 6. Налаштування доступу до ArgoCD UI у GitHub Codespaces

Під час виконання PoC використовувалось середовище GitHub Codespaces.

Для коректного відкриття ArgoCD UI через forwarded port у Codespaces ArgoCD server було переведено в insecure HTTP mode.

Для локального PoC це допустимо, оскільки доступ використовується лише для демонстрації.

Увімкнення insecure HTTP mode:

```bash

kubectl patch configmap argocd-cmd-params-cm -n argocd \

  --type merge \

  -p '{"data":{"server.insecure":"true"}}'

```

Перезапуск ArgoCD server:

```bash

kubectl rollout restart deployment argocd-server -n argocd

kubectl rollout status deployment argocd-server -n argocd

```

Запуск port-forward:

```bash

kubectl port-forward --address 0.0.0.0 svc/argocd-server -n argocd 8080:80

```

Альтернативний запуск port-forward у background mode:

```bash

nohup kubectl port-forward --address 0.0.0.0 svc/argocd-server -n argocd 8080:80 > /tmp/argocd-port-forward.log 2>&1 &

```

Перевірка port-forward:

```bash

curl -I http://localhost:8080

```

Очікуваний результат:

```text

HTTP/1.1 200 OK

```

Після запуску port-forward у GitHub Codespaces потрібно:

1. Відкрити вкладку **Ports**.

2. Знайти порт `8080`.

3. Натиснути **Open in Browser**.

4. Відкрити ArgoCD UI у браузері.

---

## 7. Отримання пароля адміністратора

Початковий користувач ArgoCD:

```text

admin

```

Пароль отримується з Kubernetes secret:

```bash

kubectl -n argocd get secret argocd-initial-admin-secret \

  -o jsonpath="{.data.password}" | base64 -d; echo

```

---

## 8. Вхід в ArgoCD UI

Дані для входу:

```text

Username: admin

Password: <password from argocd-initial-admin-secret>

```

Після входу відкривається графічний інтерфейс ArgoCD.

---

## 9. Перевірка роботи ArgoCD

Перевірка pod-ів:

```bash

kubectl get pods -n argocd

```

Перевірка доступності UI:

```bash

curl -I http://localhost:8080

```

Очікуваний результат:

```text

HTTP/1.1 200 OK

```

Також було перевірено успішний вхід у вебінтерфейс ArgoCD через GitHub Codespaces forwarded port.

---

## 10. Результат PoC

У межах PoC було виконано:

- створено локальний Kubernetes-кластер за допомогою k3d;

- встановлено ArgoCD у namespace `argocd`;

- перевірено стан pod-ів ArgoCD;

- налаштовано доступ до ArgoCD UI через GitHub Codespaces Ports;

- отримано початковий пароль адміністратора;

- виконано успішний вхід у вебінтерфейс ArgoCD.

ArgoCD готовий для наступного етапу — реалізації MVP AsciiArtify.

---

## 11. Repository

```text

https://github.com/viktorshlapak/AsciiArtify

```

