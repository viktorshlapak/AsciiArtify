# AsciiArtify — Concept

## Вступ

AsciiArtify планує створити ML-застосунок для перетворення зображень у ASCII-art. Для локальної розробки та PoC команда розглядає Kubernetes-інструменти: minikube, kind та k3d.

Мета документа — порівняти ці інструменти, врахувати ризики ліцензування Docker Desktop та оцінити можливість використання Podman як альтернативи.

## Порівняльна таблиця

| Критерій | minikube | kind | k3d |
|---|---|---|---|
| Призначення | Локальний Kubernetes для розробки | Kubernetes-кластери в Docker-контейнерах | Легкі Kubernetes-кластери k3s у Docker-контейнерах |
| ОС | Linux, macOS, Windows | Linux, macOS, Windows | Linux, macOS, Windows |
| Архітектури | amd64, arm64 | amd64, arm64 | amd64, arm64 |
| Runtime | Docker, Podman, VirtualBox, HyperKit, Hyper-V | Docker, частково Podman | Docker, Podman через сумісність |
| Швидкість старту | Середня | Висока | Дуже висока |
| Автоматизація | Добра | Дуже добра | Дуже добра |
| Multi-node | Так | Так | Так |
| LoadBalancer | Через minikube tunnel або NodePort | Потребує додаткового налаштування | Є зручна підтримка через k3s/Traefik |
| Ingress | Через addon | Ручне встановлення | Часто доступний через Traefik |
| Моніторинг | Addons, dashboard | Ручне встановлення | Ручне встановлення |
| Складність | Низька | Середня | Низька |
| Документація | Дуже добра | Добра | Добра |
| Найкраще для | Навчання, локальна розробка, PoC | CI/CD та тестування Kubernetes | Швидкі локальні демо та PoC |

## Minikube

Minikube — це інструмент для запуску локального Kubernetes-кластера на одному комп’ютері. Він добре підходить для навчання, локальної розробки та тестування базових Kubernetes-сценаріїв.

### Переваги

- простий старт;
- хороша документація;
- підтримка addons;
- підтримка Docker, Podman та віртуальних машин;
- зручний для початківців;
- добре підходить для PoC.

### Недоліки

- повільніший старт у порівнянні з kind/k3d;
- менш зручний для CI/CD;
- multi-node сценарії можуть бути складнішими.

## kind

Kind — це Kubernetes IN Docker. Він створює Kubernetes-кластери як Docker-контейнери. Часто використовується для тестування Kubernetes, CI/CD та перевірки Kubernetes-маніфестів.

### Переваги

- швидке створення кластерів;
- зручний для автоматизації;
- добре підходить для CI/CD;
- підтримує multi-node кластери;
- близький до upstream Kubernetes.

### Недоліки

- потребує Docker або сумісного runtime;
- менше вбудованих можливостей для ingress/load balancer;
- не такий дружній для новачків, як minikube.

## k3d

k3d — це інструмент для запуску k3s-кластерів у Docker-контейнерах. k3s є легковаговою версією Kubernetes, тому k3d швидко стартує та добре підходить для локальних PoC.

### Переваги

- дуже швидкий старт;
- легкий і ресурсоефективний;
- зручний для PoC;
- підтримує multi-node;
- добре підходить для локальних демо;
- просте створення та видалення кластерів.

### Недоліки

- використовує k3s, а не повний upstream Kubernetes;
- деякі компоненти можуть відрізнятися від production Kubernetes;
- для складних enterprise-сценаріїв може знадобитись додаткова перевірка сумісності.

## Docker licensing risk та Podman

Docker Desktop має ліцензійні обмеження для комерційного використання у великих компаніях. Для стартапу це може бути не критично на ранньому етапі, але ризик потрібно враховувати.

Podman може бути альтернативою Docker, оскільки є open-source та не потребує Docker Desktop. Minikube має хорошу підтримку Podman. Kind і k3d також можуть працювати з Podman у певних сценаріях, але Docker-сумісність у них зазвичай простіша та стабільніша.

Для PoC можна використовувати Docker або Podman, але для довгострокового використання варто перевірити ліцензійні вимоги Docker Desktop.

## Демонстрація minikube

Рекомендований інструмент для PoC — minikube, оскільки він простий для початківців, має хорошу документацію, підтримує addons і вже доступний у багатьох навчальних або локальних середовищах.

### Створення кластера

```bash
minikube start --driver=docker
```

### Перевірка кластера

```bash
kubectl get nodes
kubectl cluster-info
```

Очікуваний результат:

```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   ...
```

### Розгортання Hello World

```bash
kubectl create deployment hello-world --image=nginx
kubectl expose deployment hello-world --type=NodePort --port=80
```

### Перевірка deployment та service

```bash
kubectl get deployments
kubectl get pods
kubectl get svc
```

### Отримання URL сервісу

```bash
minikube service hello-world --url
```

Або можна використати port-forward:

```bash
kubectl port-forward service/hello-world 8080:80
```

Після цього перевірити застосунок:

```bash
curl http://localhost:8080
```

Очікуваний результат — HTML-сторінка nginx.

### Видалення тестового застосунку

```bash
kubectl delete service hello-world
kubectl delete deployment hello-world
```

### Зупинка кластера

```bash
minikube stop
```

## Висновки

Для PoC стартапу AsciiArtify рекомендовано використовувати minikube. Він найкраще підходить для команди без значного DevOps-досвіду, оскільки має простий старт, хорошу документацію, підтримку addons і дозволяє швидко перевірити базові Kubernetes-сценарії.

Kind краще підходить для CI/CD та автоматизованого тестування Kubernetes-маніфестів.

k3d є хорошим вибором для швидких локальних демо та легковагових Kubernetes-кластерів, але через використання k3s може мати відмінності від повного production Kubernetes.

З урахуванням простоти, документації та навчальної цінності, minikube є оптимальним вибором для першого PoC AsciiArtify.