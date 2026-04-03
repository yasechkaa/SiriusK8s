# Kubernetes Cheatsheet — Краткий справочник

## kubectl — основные команды

### Информация о кластере
```bash
kubectl cluster-info                    # Адрес API Server
kubectl get nodes -o wide               # Список нод с деталями
kubectl top nodes                       # Потребление ресурсов нод
kubectl top pods -A                     # Потребление ресурсов подов
kubectl api-resources                   # Все типы объектов в кластере
```

### Pods
```bash
kubectl get pods -A                     # Все поды во всех namespace
kubectl get pods -n <ns> -o wide        # Поды в namespace с IP и нодой
kubectl describe pod <name> -n <ns>     # Детали пода (events!)
kubectl logs <pod> -n <ns> -f           # Логи пода (follow)
kubectl logs <pod> -c <container>       # Логи конкретного контейнера
kubectl logs <pod> --previous           # Логи предыдущего инстанса
kubectl exec -it <pod> -- bash          # Shell в контейнер
kubectl port-forward <pod> 8080:80      # Проброс порта
kubectl delete pod <pod> -n <ns>        # Удалить под (будет пересоздан)
```

### Deployments
```bash
kubectl get deploy -n <ns>              # Список деплойментов
kubectl scale deploy <name> --replicas=3  # Масштабирование
kubectl rollout status deploy <name>    # Статус обновления
kubectl rollout history deploy <name>   # История ревизий
kubectl rollout undo deploy <name>      # Откат на предыдущую версию
kubectl set image deploy/<name> <container>=<image>:<tag>  # Обновить образ
```

### Services
```bash
kubectl get svc -A                      # Все сервисы
kubectl describe svc <name> -n <ns>     # Детали сервиса (endpoints!)
kubectl get endpoints <svc> -n <ns>     # IP подов за сервисом
```

### ConfigMaps и Secrets
```bash
kubectl create configmap <name> --from-file=config.yaml
kubectl create secret generic <name> --from-literal=password=secret
kubectl get secret <name> -o jsonpath='{.data.password}' | base64 -d
```

### Troubleshooting
```bash
kubectl get events -A --sort-by='.lastTimestamp'   # Последние события
kubectl describe node <node>            # Проблемы ноды
kubectl get pods --field-selector=status.phase=Failed  # Упавшие поды
kubectl run debug --image=busybox -it --rm -- sh   # Debug-контейнер
kubectl auth can-i create pods --as=system:serviceaccount:default:mysa
```

## Типичные проблемы и решения

| Статус | Причина | Диагностика |
|--------|---------|-------------|
| `Pending` | Нет ресурсов / нет подходящей ноды | `kubectl describe pod` → Events |
| `CrashLoopBackOff` | Приложение падает | `kubectl logs` + `kubectl describe` |
| `ImagePullBackOff` | Неверный образ / нет доступа к registry | Проверить image name, imagePullSecrets |
| `OOMKilled` | Превышен memory limit | Увеличить limits или оптимизировать приложение |
| `Evicted` | Нода под давлением (disk/memory) | `kubectl describe node` → Conditions |
| `CreateContainerConfigError` | Нет ConfigMap/Secret | Проверить существование объектов |

## YAML-шаблоны

### Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Service + Ingress
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-svc
            port:
              number: 80
```

### NetworkPolicy (deny all, allow specific)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: myns
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web
  namespace: myns
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    ports:
    - port: 8080
```

## Docker — полезные команды

### Основные операции
```bash
docker ps                      # Контейнеры (только запущенные)
docker ps -a                   # ВСЕ контейнеры (в том числе остановленные)
docker images                  # Локальные образы
docker pull nginx:alpine       # Скачать образ
docker run -it ubuntu bash     # Запуск контейнера в shell
docker exec -it <container> bash         # Интерпретатор внутри уже работающего контейнера
docker logs <container>                  # Логи контейнера
docker stop <container>                  # Остановить контейнер
docker start <container>                 # Запустить остановленный контейнер
docker rm <container>                    # Удалить контейнер
docker rmi <image>                       # Удалить образ
docker build -t myimage:tag .            # Собрать образ из Dockerfile
docker system prune -a                   # Удалить все неиспользуемое (ОСТОРОЖНО!)
```

### Инспекция и копирование файлов
```bash
docker inspect <container>               # Детальная информация о контейнере
docker diff <container>                  # Изменения файловой системы контейнера
docker cp <container>:/path/to/file .    # Скачать файл из контейнера
docker cp ./local.txt <container>:/root  # Поместить файл в контейнер
```

### Безопасный запуск
```bash
docker run \
  --read-only \
  --no-new-privileges \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --user 1000:1000 \
  --memory 512m --cpus 0.5 \
  --security-opt seccomp=default \
  your-image:tag
```

## Полезные alias

```bash
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias ke='kubectl exec -it'
alias kaf='kubectl apply -f'
alias kdf='kubectl delete -f'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgn='kubectl get nodes'
alias dps='docker ps'
alias dpa='docker ps -a'
alias dim='docker images'
alias drun='docker run -it'
alias dexec='docker exec -it'
alias dlogs='docker logs'
```
