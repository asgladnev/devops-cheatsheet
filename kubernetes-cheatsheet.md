# ☸️ Kubernetes Cheatsheet — Полная шпаргалка

> **Версия:** K8s 1.28+  
> **Соглашение:** `<value>` — заменить своим значением, `[flag]` — необязательный флаг

---

## 📌 Содержание

1. [kubectl — базовая настройка](#1-kubectl--базовая-настройка)
2. [Контексты и Namespaces](#2-контексты-и-namespaces)
3. [Pods](#3-pods)
4. [Deployments](#4-deployments)
5. [ReplicaSets](#5-replicasets)
6. [Services](#6-services)
7. [ConfigMaps и Secrets](#7-configmaps-и-secrets)
8. [Volumes и PersistentVolumes](#8-volumes-и-persistentvolumes)
9. [StatefulSets](#9-statefulsets)
10. [DaemonSets](#10-daemonsets)
11. [Jobs и CronJobs](#11-jobs-и-cronjobs)
12. [Ingress](#12-ingress)
13. [RBAC](#13-rbac)
14. [Horizontal Pod Autoscaler (HPA)](#14-horizontal-pod-autoscaler-hpa)
15. [Resource Limits и Requests](#15-resource-limits-и-requests)
16. [Node Management](#16-node-management)
17. [Helm](#17-helm)
18. [Отладка и Troubleshooting](#18-отладка-и-troubleshooting)
19. [Полезные флаги и трюки](#19-полезные-флаги-и-трюки)
20. [Манифесты — шаблоны](#20-манифесты--шаблоны)

---

## 1. kubectl — базовая настройка

```bash
# Проверить версию kubectl и кластера
kubectl version --short

# Проверить доступность кластера
kubectl cluster-info

# Проверить конфигурацию
kubectl config view

# Путь к kubeconfig по умолчанию
~/.kube/config

# Использовать другой kubeconfig
export KUBECONFIG=/path/to/kubeconfig
kubectl --kubeconfig=/path/to/kubeconfig get pods

# Автодополнение (bash)
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

# Автодополнение (zsh)
source <(kubectl completion zsh)
echo "[[ $commands[kubectl] ]] && source <(kubectl completion zsh)" >> ~/.zshrc

# Алиас k (очень удобно)
alias k=kubectl
complete -F __start_kubectl k
```

---

## 2. Контексты и Namespaces

### Контексты

```bash
# Список всех контекстов
kubectl config get-contexts

# Текущий контекст
kubectl config current-context

# Переключить контекст
kubectl config use-context <context-name>

# Переименовать контекст
kubectl config rename-context <old-name> <new-name>

# Удалить контекст
kubectl config delete-context <context-name>

# Создать контекст
kubectl config set-context <name> \
  --cluster=<cluster> \
  --user=<user> \
  --namespace=<namespace>
```

### Namespaces

```bash
# Список namespaces
kubectl get namespaces
kubectl get ns

# Создать namespace
kubectl create namespace <name>

# Удалить namespace (осторожно! удалит всё внутри)
kubectl delete namespace <name>

# Установить namespace по умолчанию для контекста
kubectl config set-context --current --namespace=<namespace>

# Запустить команду в конкретном namespace
kubectl get pods -n <namespace>
kubectl get pods --namespace=<namespace>

# Получить ресурсы во всех namespace
kubectl get pods --all-namespaces
kubectl get pods -A
```

---

## 3. Pods

### Просмотр

```bash
# Список всех pods в текущем namespace
kubectl get pods
kubectl get po

# Список pods с дополнительной информацией (IP, Node)
kubectl get pods -o wide

# Список pods во всех namespaces
kubectl get pods -A

# Список pods с метками (labels)
kubectl get pods --show-labels

# Список pods с фильтрацией по label
kubectl get pods -l app=myapp
kubectl get pods -l 'env in (prod, staging)'
kubectl get pods -l app=myapp,tier=frontend

# Детальная информация о pod
kubectl describe pod <pod-name>
kubectl describe pod <pod-name> -n <namespace>

# Вывод в JSON / YAML
kubectl get pod <pod-name> -o json
kubectl get pod <pod-name> -o yaml

# Вывод определённых полей (jsonpath)
kubectl get pod <pod-name> -o jsonpath='{.status.podIP}'
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Наблюдать за pods в реальном времени
kubectl get pods -w
kubectl get pods --watch
```

### Логи

```bash
# Логи pod
kubectl logs <pod-name>

# Логи конкретного контейнера в pod
kubectl logs <pod-name> -c <container-name>

# Следить за логами в реальном времени
kubectl logs -f <pod-name>
kubectl logs -f <pod-name> -c <container-name>

# Последние N строк
kubectl logs <pod-name> --tail=100

# Логи за последние X минут/часов
kubectl logs <pod-name> --since=1h
kubectl logs <pod-name> --since=30m

# Логи упавшего (предыдущего) контейнера
kubectl logs <pod-name> -p
kubectl logs <pod-name> --previous
```

### Запуск и управление

```bash
# Запустить pod из образа (быстрый тест)
kubectl run <pod-name> --image=<image>
kubectl run nginx --image=nginx

# Запустить pod и войти в него сразу
kubectl run -it <pod-name> --image=<image> --restart=Never -- sh

# Запустить временный pod для отладки (удалится после выхода)
kubectl run tmp --image=busybox --restart=Never --rm -it -- sh
kubectl run tmp --image=curlimages/curl --restart=Never --rm -it -- sh

# Удалить pod
kubectl delete pod <pod-name>

# Принудительно удалить (без grace period)
kubectl delete pod <pod-name> --force --grace-period=0

# Удалить несколько pods
kubectl delete pods <pod1> <pod2>

# Удалить pods по label
kubectl delete pods -l app=myapp

# Выполнить команду в pod
kubectl exec <pod-name> -- <command>
kubectl exec <pod-name> -- ls /app

# Интерактивная оболочка в pod
kubectl exec -it <pod-name> -- sh
kubectl exec -it <pod-name> -- bash
kubectl exec -it <pod-name> -c <container-name> -- bash

# Копировать файлы из/в pod
kubectl cp <pod-name>:/path/to/file ./local-file
kubectl cp ./local-file <pod-name>:/path/to/file
kubectl cp <pod-name>:/path/to/dir ./local-dir -c <container>
```

### Port forwarding

```bash
# Пробросить порт локально к pod
kubectl port-forward pod/<pod-name> <local-port>:<pod-port>
kubectl port-forward pod/nginx 8080:80

# Пробросить к сервису
kubectl port-forward svc/<service-name> <local-port>:<service-port>

# Пробросить к deployment
kubectl port-forward deployment/<name> <local-port>:<port>

# Слушать на всех интерфейсах (не только localhost)
kubectl port-forward pod/<pod-name> --address 0.0.0.0 8080:80
```

---

## 4. Deployments

### Просмотр

```bash
# Список deployments
kubectl get deployments
kubectl get deploy

# Детальная информация
kubectl describe deployment <name>

# YAML текущего deployment
kubectl get deployment <name> -o yaml
```

### Создание и обновление

```bash
# Создать deployment
kubectl create deployment <name> --image=<image>
kubectl create deployment nginx --image=nginx:1.25

# Создать deployment с несколькими репликами
kubectl create deployment nginx --image=nginx --replicas=3

# Применить манифест из файла
kubectl apply -f deployment.yaml
kubectl apply -f ./k8s/  # применить все файлы в папке
kubectl apply -f https://url/to/manifest.yaml

# Обновить образ
kubectl set image deployment/<name> <container>=<new-image>
kubectl set image deployment/nginx nginx=nginx:1.26

# Обновить env-переменную
kubectl set env deployment/<name> ENV_KEY=value

# Масштабировать deployment
kubectl scale deployment/<name> --replicas=5
kubectl scale deployment/nginx --replicas=0   # остановить все pods

# Редактировать deployment в редакторе
kubectl edit deployment <name>

# Перезапустить все pods в deployment (rolling restart)
kubectl rollout restart deployment/<name>
```

### Rollout (откат и история)

```bash
# Статус rollout
kubectl rollout status deployment/<name>

# История rollout
kubectl rollout history deployment/<name>

# Детали конкретной ревизии
kubectl rollout history deployment/<name> --revision=2

# Откатиться на предыдущую версию
kubectl rollout undo deployment/<name>

# Откатиться на конкретную ревизию
kubectl rollout undo deployment/<name> --to-revision=2

# Приостановить rollout
kubectl rollout pause deployment/<name>

# Возобновить rollout
kubectl rollout resume deployment/<name>
```

---

## 5. ReplicaSets

```bash
# Список ReplicaSets
kubectl get replicasets
kubectl get rs

# Детальная информация
kubectl describe rs <name>

# Масштабировать ReplicaSet напрямую (лучше через Deployment)
kubectl scale rs/<name> --replicas=3
```

---

## 6. Services

### Просмотр

```bash
# Список сервисов
kubectl get services
kubectl get svc

# Детальная информация
kubectl describe svc <name>

# Endpoints (реальные IP pods за сервисом)
kubectl get endpoints <service-name>
kubectl get ep
```

### Создание

```bash
# Expose deployment как ClusterIP сервис
kubectl expose deployment <name> --port=80 --target-port=8080

# Expose как NodePort
kubectl expose deployment <name> --type=NodePort --port=80

# Expose как LoadBalancer
kubectl expose deployment <name> --type=LoadBalancer --port=80

# Создать сервис из файла
kubectl apply -f service.yaml

# Удалить сервис
kubectl delete svc <name>
```

### Типы сервисов

| Тип | Описание |
|-----|----------|
| `ClusterIP` | Только внутри кластера (по умолчанию) |
| `NodePort` | Доступен по IP ноды + порт (30000-32767) |
| `LoadBalancer` | Создаёт внешний LB (облако) |
| `ExternalName` | Alias для внешнего DNS-имени |
| `Headless` | `clusterIP: None` — DNS для каждого pod |

---

## 7. ConfigMaps и Secrets

### ConfigMaps

```bash
# Создать из литерала
kubectl create configmap <name> --from-literal=KEY=value
kubectl create configmap app-config \
  --from-literal=DB_HOST=postgres \
  --from-literal=DB_PORT=5432

# Создать из файла
kubectl create configmap <name> --from-file=config.properties
kubectl create configmap <name> --from-file=./configs/

# Создать из env-файла
kubectl create configmap <name> --from-env-file=.env

# Просмотр
kubectl get configmaps
kubectl get cm
kubectl describe cm <name>
kubectl get cm <name> -o yaml

# Удалить
kubectl delete cm <name>
```

### Secrets

```bash
# Создать secret (generic)
kubectl create secret generic <name> \
  --from-literal=username=admin \
  --from-literal=password=s3cr3t

# Создать из файла
kubectl create secret generic <name> --from-file=./secret.txt

# Создать Docker Registry secret
kubectl create secret docker-registry <name> \
  --docker-server=<registry> \
  --docker-username=<user> \
  --docker-password=<password> \
  --docker-email=<email>

# TLS secret
kubectl create secret tls <name> \
  --cert=tls.crt \
  --key=tls.key

# Просмотр (значения в base64)
kubectl get secrets
kubectl describe secret <name>
kubectl get secret <name> -o yaml

# Декодировать значение secret
kubectl get secret <name> -o jsonpath='{.data.password}' | base64 --decode
kubectl get secret <name> -o go-template='{{.data.password | base64decode}}'

# Удалить
kubectl delete secret <name>
```

> ⚠️ **Best practice:** Никогда не храни секреты в git. Используй External Secrets Operator, Vault, или Sealed Secrets.

---

## 8. Volumes и PersistentVolumes

### PersistentVolume (PV) и PersistentVolumeClaim (PVC)

```bash
# Список PV
kubectl get persistentvolumes
kubectl get pv

# Список PVC
kubectl get persistentvolumeclaims
kubectl get pvc
kubectl get pvc -n <namespace>

# Детали
kubectl describe pv <name>
kubectl describe pvc <name>

# Удалить PVC (осторожно! может удалить данные в зависимости от Reclaim Policy)
kubectl delete pvc <name>
```

### StorageClasses

```bash
# Список StorageClasses
kubectl get storageclasses
kubectl get sc

# Описание
kubectl describe sc <name>
```

### Статусы PV

| Статус | Описание |
|--------|----------|
| `Available` | Свободен, не привязан к PVC |
| `Bound` | Привязан к PVC |
| `Released` | PVC удалён, PV ещё хранит данные |
| `Failed` | Ошибка автоматического освобождения |

---

## 9. StatefulSets

```bash
# Список StatefulSets
kubectl get statefulsets
kubectl get sts

# Детали
kubectl describe sts <name>

# Масштабировать StatefulSet
kubectl scale sts/<name> --replicas=3

# Обновить образ
kubectl set image sts/<name> <container>=<image>

# Статус rollout
kubectl rollout status sts/<name>

# Откат
kubectl rollout undo sts/<name>

# Удалить (PVC сохранятся!)
kubectl delete sts <name>

# Удалить вместе с PVC
kubectl delete sts <name> && kubectl delete pvc -l app=<name>
```

---

## 10. DaemonSets

```bash
# Список DaemonSets
kubectl get daemonsets
kubectl get ds

# Детали
kubectl describe ds <name>

# Обновить образ
kubectl set image ds/<name> <container>=<image>

# Статус rollout
kubectl rollout status ds/<name>

# Откат
kubectl rollout undo ds/<name>
```

---

## 11. Jobs и CronJobs

### Jobs

```bash
# Создать job
kubectl create job <name> --image=<image>
kubectl create job hello --image=busybox -- echo "Hello"

# Список jobs
kubectl get jobs

# Детали
kubectl describe job <name>

# Логи job (найти pod по label)
kubectl logs -l job-name=<job-name>

# Удалить job (и связанные pods)
kubectl delete job <name>
```

### CronJobs

```bash
# Создать CronJob
kubectl create cronjob <name> \
  --image=<image> \
  --schedule="*/5 * * * *" \
  -- <command>

# Список CronJobs
kubectl get cronjobs
kubectl get cj

# Детали
kubectl describe cj <name>

# Запустить CronJob вручную (создать Job из CronJob)
kubectl create job <job-name> --from=cronjob/<cronjob-name>

# Приостановить CronJob
kubectl patch cj <name> -p '{"spec":{"suspend":true}}'

# Возобновить CronJob
kubectl patch cj <name> -p '{"spec":{"suspend":false}}'

# Удалить
kubectl delete cj <name>
```

### Cron-синтаксис

```
┌───────── minute (0 - 59)
│ ┌───────── hour (0 - 23)
│ │ ┌───────── day of month (1 - 31)
│ │ │ ┌───────── month (1 - 12)
│ │ │ │ ┌───────── day of week (0 - 6, Sun=0)
│ │ │ │ │
* * * * *

Примеры:
0 * * * *     — каждый час
*/5 * * * *   — каждые 5 минут
0 9 * * 1-5   — в 9:00 в рабочие дни
0 0 1 * *     — в полночь 1-го числа каждого месяца
```

---

## 12. Ingress

```bash
# Список Ingress
kubectl get ingress
kubectl get ing

# Детали
kubectl describe ing <name>

# Применить манифест
kubectl apply -f ingress.yaml

# Удалить
kubectl delete ing <name>
```

---

## 13. RBAC

### ServiceAccounts

```bash
# Список ServiceAccounts
kubectl get serviceaccounts
kubectl get sa

# Создать SA
kubectl create serviceaccount <name>

# Детали
kubectl describe sa <name>
```

### Roles и ClusterRoles

```bash
# Список Roles (namespace-level)
kubectl get roles -n <namespace>

# Список ClusterRoles (cluster-level)
kubectl get clusterroles

# Детали
kubectl describe role <name> -n <namespace>
kubectl describe clusterrole <name>
```

### RoleBindings и ClusterRoleBindings

```bash
# Список RoleBindings
kubectl get rolebindings -n <namespace>

# Список ClusterRoleBindings
kubectl get clusterrolebindings

# Создать RoleBinding
kubectl create rolebinding <name> \
  --role=<role-name> \
  --user=<user> \
  -n <namespace>

# Создать ClusterRoleBinding
kubectl create clusterrolebinding <name> \
  --clusterrole=<role-name> \
  --user=<user>

# Привязать к ServiceAccount
kubectl create rolebinding <name> \
  --role=<role-name> \
  --serviceaccount=<namespace>:<sa-name> \
  -n <namespace>

# Проверить права
kubectl auth can-i <verb> <resource>
kubectl auth can-i get pods
kubectl auth can-i create deployments --namespace=prod

# Проверить права от имени другого пользователя
kubectl auth can-i get pods --as=<user>
kubectl auth can-i get pods --as=system:serviceaccount:<ns>:<sa>

# Список всех прав текущего пользователя
kubectl auth can-i --list
kubectl auth can-i --list --namespace=prod
```

---

## 14. Horizontal Pod Autoscaler (HPA)

```bash
# Создать HPA
kubectl autoscale deployment <name> \
  --min=2 \
  --max=10 \
  --cpu-percent=70

# Список HPA
kubectl get hpa

# Детали
kubectl describe hpa <name>

# Удалить HPA
kubectl delete hpa <name>
```

> ⚠️ Для работы HPA необходим **metrics-server** в кластере.

---

## 15. Resource Limits и Requests

### Просмотр потребления ресурсов

```bash
# Потребление ресурсов pods (требует metrics-server)
kubectl top pods
kubectl top pods -n <namespace>
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory

# Потребление ресурсов нод
kubectl top nodes

# Ресурсы pod из describe
kubectl describe pod <name> | grep -A 5 "Limits\|Requests"
```

### Best practices для Limits/Requests

```yaml
# Всегда задавай requests и limits!
resources:
  requests:
    memory: "64Mi"    # Гарантированная память
    cpu: "250m"       # 0.25 CPU ядра
  limits:
    memory: "128Mi"   # Максимальная память
    cpu: "500m"       # 0.5 CPU ядра

# 1000m = 1 CPU core
# Для памяти: Ki, Mi, Gi
```

### LimitRange (ограничения по умолчанию для namespace)

```bash
kubectl get limitrange -n <namespace>
kubectl describe limitrange -n <namespace>
```

### ResourceQuota (квоты для namespace)

```bash
kubectl get resourcequota -n <namespace>
kubectl describe resourcequota -n <namespace>
```

---

## 16. Node Management

```bash
# Список нод
kubectl get nodes
kubectl get nodes -o wide

# Детали ноды
kubectl describe node <node-name>

# Labels нод
kubectl get nodes --show-labels

# Добавить label на ноду
kubectl label node <node-name> <key>=<value>
kubectl label node worker-1 disktype=ssd

# Удалить label
kubectl label node <node-name> <key>-

# Пометить ноду как unschedulable (новые pods не будут планироваться)
kubectl cordon <node-name>

# Снять ограничение
kubectl uncordon <node-name>

# Эвакуировать все pods с ноды (перед обслуживанием)
kubectl drain <node-name>
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force

# Taints — запрет планирования без tolerations
kubectl taint nodes <node-name> <key>=<value>:<effect>
kubectl taint nodes worker-1 dedicated=gpu:NoSchedule

# Удалить taint
kubectl taint nodes <node-name> <key>:<effect>-
kubectl taint nodes worker-1 dedicated:NoSchedule-
```

### Эффекты Taint

| Effect | Описание |
|--------|----------|
| `NoSchedule` | Не планировать новые pods без toleration |
| `PreferNoSchedule` | Стараться не планировать (мягкое ограничение) |
| `NoExecute` | Эвакуировать существующие pods + не планировать новые |

---

## 17. Helm

```bash
# Версия
helm version

# Добавить репозиторий
helm repo add <name> <url>
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami

# Обновить репозитории
helm repo update

# Список репозиториев
helm repo list

# Поиск чарта
helm search repo <keyword>
helm search hub <keyword>  # поиск в Artifact Hub

# Показать значения чарта (values.yaml)
helm show values <repo>/<chart>
helm show chart <repo>/<chart>
helm show all <repo>/<chart>

# Установить чарт
helm install <release-name> <repo>/<chart>
helm install my-nginx bitnami/nginx

# Установить с кастомными values
helm install <release-name> <repo>/<chart> \
  --values values.yaml \
  --set key=value \
  --set-string key=value

# Установить в конкретный namespace
helm install <release-name> <repo>/<chart> \
  -n <namespace> \
  --create-namespace

# Предварительный просмотр (dry run)
helm install <name> <chart> --dry-run --debug

# Обновить release
helm upgrade <release-name> <repo>/<chart>
helm upgrade <release-name> <repo>/<chart> \
  --values values.yaml \
  --set image.tag=2.0

# Установить или обновить (install or upgrade)
helm upgrade --install <release-name> <repo>/<chart>

# Список установленных releases
helm list
helm list -A  # все namespaces
helm list -n <namespace>

# Статус release
helm status <release-name>

# История release
helm history <release-name>

# Откатить release
helm rollback <release-name>          # предыдущая версия
helm rollback <release-name> <revision>

# Удалить release
helm uninstall <release-name>
helm uninstall <release-name> -n <namespace>

# Получить сгенерированные манифесты
helm get manifest <release-name>

# Получить values установленного release
helm get values <release-name>

# Lint чарта (проверка синтаксиса)
helm lint ./my-chart

# Template — рендер без установки
helm template <name> ./my-chart --values values.yaml

# Упаковать чарт
helm package ./my-chart
```

---

## 18. Отладка и Troubleshooting

### Диагностика pods

```bash
# Pod не стартует — смотрим events
kubectl describe pod <pod-name>

# Частые причины в Events:
# - ImagePullBackOff / ErrImagePull → проблема с образом или registry
# - CrashLoopBackOff              → контейнер падает при старте
# - OOMKilled                     → нехватка памяти (увеличь limits)
# - Pending                       → нет ресурсов / нет подходящей ноды
# - CreateContainerConfigError    → ошибка в ConfigMap или Secret

# Смотрим логи упавшего контейнера
kubectl logs <pod-name> --previous

# Войти в pod для отладки
kubectl exec -it <pod-name> -- sh

# Запустить временный отладочный pod в том же namespace
kubectl run debug --image=busybox --restart=Never --rm -it -- sh

# Отладочный pod с сетевыми утилитами
kubectl run debug --image=nicolaka/netshoot --restart=Never --rm -it -- bash

# Проверить DNS в pod
kubectl exec -it <pod-name> -- nslookup <service-name>
kubectl exec -it <pod-name> -- nslookup kubernetes.default

# Проверить доступность сервиса
kubectl exec -it <pod-name> -- curl http://<service-name>:<port>
```

### Диагностика сети

```bash
# Проверить endpoints сервиса
kubectl get endpoints <service-name>

# Нет endpoints → labels на pods не совпадают с selector сервиса
kubectl get pods --show-labels
kubectl describe svc <service-name>  # смотрим Selector

# Проверить NetworkPolicy
kubectl get networkpolicy -n <namespace>
kubectl describe networkpolicy <name>

# Проверить Ingress
kubectl describe ing <name>
kubectl get ing -A
```

### Диагностика нод

```bash
# Почему pod не планируется на ноду
kubectl describe pod <pod-name>  # смотрим Events → FailedScheduling

# Посмотреть состояние нод
kubectl get nodes
kubectl describe node <node-name>  # смотрим Conditions, Taints, Capacity

# Events по всему namespace
kubectl get events -n <namespace>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Events только Warning
kubectl get events -n <namespace> --field-selector type=Warning

# Events по всему кластеру
kubectl get events -A
```

### Диагностика ресурсов

```bash
# Сколько ресурсов потребляют pods
kubectl top pods -n <namespace>
kubectl top pods -A --sort-by=memory

# Quota namespace
kubectl describe resourcequota -n <namespace>

# LimitRange
kubectl describe limitrange -n <namespace>

# Allocatable vs Capacity ноды
kubectl describe node <name> | grep -A 5 "Allocatable\|Capacity"
```

### Проверка конфигурации

```bash
# Валидировать yaml без применения
kubectl apply -f manifest.yaml --dry-run=client
kubectl apply -f manifest.yaml --dry-run=server  # более строгая проверка

# Diff — что изменится при apply
kubectl diff -f manifest.yaml

# Проверить API-ресурсы (какие CRD доступны)
kubectl api-resources
kubectl api-resources --namespaced=true
kubectl api-resources --api-group=apps

# Проверить версии API
kubectl api-versions
```

---

## 19. Полезные флаги и трюки

### Форматирование вывода

```bash
# Форматы вывода
-o yaml          # YAML
-o json          # JSON
-o wide          # дополнительные колонки
-o name          # только имена ресурсов
-o jsonpath='{}' # jsonpath-выражение

# Кастомные колонки
kubectl get pods -o custom-columns=\
'NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName'

# Сортировка
kubectl get pods --sort-by='.metadata.creationTimestamp'
kubectl get pods --sort-by='.status.podIP'
```

### Работа с манифестами

```bash
# Применить несколько файлов
kubectl apply -f file1.yaml -f file2.yaml

# Применить всё в директории
kubectl apply -f ./k8s/

# Применить рекурсивно
kubectl apply -R -f ./k8s/

# Удалить по манифесту
kubectl delete -f manifest.yaml

# Получить yaml существующего ресурса (для редактирования)
kubectl get deployment <name> -o yaml > deployment.yaml
```

### Patch — быстрое изменение

```bash
# Patch JSON
kubectl patch deployment <name> \
  -p '{"spec":{"replicas":3}}'

# Patch стратегически
kubectl patch deployment <name> \
  --type=strategic \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","image":"new:tag"}]}}}}'

# Patch конкретного поля
kubectl patch deployment <name> \
  --type=json \
  -p '[{"op":"replace","path":"/spec/replicas","value":5}]'
```

### Аннотации и Labels

```bash
# Добавить label
kubectl label pod <name> env=prod
kubectl label deployment <name> version=v2

# Удалить label
kubectl label pod <name> env-

# Добавить аннотацию
kubectl annotate pod <name> description="My pod"

# Удалить аннотацию
kubectl annotate pod <name> description-
```

### Полезные однострочники

```bash
# Удалить все pods в состоянии Evicted
kubectl get pods -A | grep Evicted | awk '{print $2 " -n " $1}' | xargs -I {} kubectl delete pod {}

# Удалить все Completed jobs
kubectl delete jobs --field-selector status.successful=1 -n <namespace>

# Перезапустить все deployments в namespace
kubectl rollout restart deployment -n <namespace>

# Список образов всех pods
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.image}{"\n"}{end}{end}'

# Следить за событиями в реальном времени
kubectl get events -w --sort-by='.lastTimestamp'

# Масштабировать все deployments до 0 (заморозить namespace)
kubectl scale deployment --all --replicas=0 -n <namespace>

# Экспортировать все манифесты namespace
kubectl get all -n <namespace> -o yaml > namespace-backup.yaml
```

---

## 20. Манифесты — шаблоны

### Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: default
  labels:
    app: my-app
    version: v1
spec:
  containers:
    - name: app
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
      env:
        - name: ENV_KEY
          value: "env_value"
        - name: SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: secret-key
        - name: CONFIG_KEY
          valueFrom:
            configMapKeyRef:
              name: my-configmap
              key: config-key
      readinessProbe:
        httpGet:
          path: /healthz
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
      livenessProbe:
        httpGet:
          path: /healthz
          port: 80
        initialDelaySeconds: 15
        periodSeconds: 20
  restartPolicy: Always
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  namespace: default
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: my-sa
      containers:
        - name: app
          image: my-image:1.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: my-app
```

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: default
spec:
  selector:
    app: my-app
  type: ClusterIP          # ClusterIP | NodePort | LoadBalancer
  ports:
    - name: http
      protocol: TCP
      port: 80             # порт сервиса
      targetPort: 8080     # порт контейнера
  # nodePort: 30080        # только для NodePort (30000-32767)
```

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-configmap
  namespace: default
data:
  APP_ENV: production
  LOG_LEVEL: info
  config.properties: |
    server.port=8080
    server.timeout=30
```

### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
type: Opaque
data:
  # значения должны быть в base64: echo -n 'value' | base64
  username: YWRtaW4=
  password: czNjcjN0
```

### PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce       # RWO | ROX | RWX | RWOP
  storageClassName: standard
  resources:
    requests:
      storage: 10Gi
```

### Ingress (nginx)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
```

### HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

### CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: my-cronjob
  namespace: default
spec:
  schedule: "0 2 * * *"   # каждый день в 2:00
  concurrencyPolicy: Forbid  # Allow | Forbid | Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: job
              image: busybox
              command: ["/bin/sh", "-c", "echo hello"]
              resources:
                requests:
                  cpu: "100m"
                  memory: "64Mi"
                limits:
                  cpu: "200m"
                  memory: "128Mi"
```

---

## 💡 Best Practices — Краткая сводка

| Практика | Рекомендация |
|----------|-------------|
| **Namespaces** | Разделяй среды: dev, staging, prod |
| **Labels** | Всегда добавляй `app`, `version`, `env` |
| **Resources** | Всегда задавай requests и limits |
| **Probes** | Используй readiness и liveness probes |
| **Secrets** | Не храни в git, используй Vault/Sealed Secrets |
| **Images** | Не используй тег `latest` в prod |
| **RBAC** | Принцип минимальных прав (least privilege) |
| **PodDisruptionBudget** | Защита от одновременного удаления всех pods |
| **topologySpreadConstraints** | Распределяй pods по нодам/зонам |
| **Replicas** | Минимум 2 реплики для prod |
| **Strategy** | RollingUpdate с `maxUnavailable: 0` для zero-downtime |
| **HPA** | Всегда настраивай автомасштабирование |
| **NetworkPolicy** | Ограничивай сетевой трафик между pods |

---

*Шпаргалка актуальна для Kubernetes 1.28+*
