# Разворачиваем DNS Cluster Add-on

В этой лабораторной вы развернете [DNS add-on](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/).
Он обеспечивает обнаружение сервисов, бегущих в контейнерах внутри Kubernetes, через DNS. Основано на [CoreDNS](https://coredns.io/). 

## Современный подход

В современной версии туториала DNS addon является **опциональным** компонентом. Kubernetes может работать без DNS addon, 
используя переменные окружения для обнаружения сервисов. Однако DNS addon значительно упрощает работу с сервисами.

## Развертывание CoreDNS

### Вариант 1: Использование локального файла (рекомендуется)

Скопируйте файл CoreDNS с jumpbox на server:

```bash
# На jumpbox
scp deployments/coredns-1.11.1.yaml yc-user@<server-external-ip>:~/

# На server
kubectl apply -f coredns-1.11.1.yaml
```

### Вариант 2: Прямое применение из репозитория

```bash
kubectl apply -f https://raw.githubusercontent.com/your-repo/kubernetes-the-hard-way-yc/main/deployments/coredns-1.11.1.yaml
```

> output

```
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
```

## Проверка развертывания

Выведите список подов созданных `kube-dns`:

```bash
kubectl get pods -l k8s-app=kube-dns -n kube-system
```

> output

```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-8494f9c688-hh7r2   1/1     Running   0          10s
coredns-8494f9c688-zqrj2   1/1     Running   0          10s
```

## Альтернатива: Service Discovery без DNS

Если вы предпочитаете не использовать DNS addon, Kubernetes предоставляет переменные окружения для обнаружения сервисов:

```bash
# Создайте тестовый сервис
kubectl run nginx --image=nginx:alpine --port=80
kubectl expose pod nginx --port=80 --target-port=80 --name=nginx-service

# Создайте тестовый под для проверки
kubectl run test-pod --image=busybox:1.28 --command -- sleep 3600

# Проверьте переменные окружения
kubectl exec test-pod -- env | grep NGINX
```

## Проверка DNS (если развернут)

Создайте `busybox`:

```bash
kubectl run busybox --image=busybox:1.28 --command -- sleep 3600
```

Выведите список подов созданных при развертывании`busybox`:

```bash
kubectl get pods -l run=busybox
```

> output

```
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          3s
```

Получите полное название пода `busybox`:

```bash
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

Выполните DNS lookup для сервиса `kubernetes` внутри пода `busybox`:

```bash
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

> output

```
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```

## Современные возможности CoreDNS

Новая версия CoreDNS (1.11.1) включает:

- **Улучшенная производительность**: Оптимизированное кэширование и forward
- **Pod Security Standards**: Совместимость с новыми стандартами безопасности
- **EndpointSlices поддержка**: Современный API для endpoints
- **Метрики Prometheus**: Встроенная поддержка мониторинга
- **Health checks**: Улучшенные проверки здоровья

Дальше: [Комплексная проверка кластера](14-smoke-test.md)
