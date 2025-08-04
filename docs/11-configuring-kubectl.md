# Настраиваем удаленный доступ kubectl

В этой лабораторной вы сгенерируете kubeconfig для утилиты `kubectl`, основанный на данных для авторизации
пользователя `admin`.

> Выполняйте команды из директории где у вас лежат сертификаты для `admin` на jumpbox.

## Важное

**Все команды в этой главе выполняются на jumpbox** - центральной машине управления кластером. 
Это упрощает процесс и обеспечивает централизованное управление доступом к кластеру.

## Конфигурационный файл

Каждый kubeconfig требует API сервер Kubernetes к которому будет подключаться. Для большей доступности мы используем IP
балансера.

Сгенерируйте kubeconfig подходящий, чтобы авторизоваться от имени пользователя `admin`:

```bash
# На jumpbox
cd ~/kubernetes-the-hard-way/certificates

{
  KUBERNETES_PUBLIC_ADDRESS=$(yc vpc address get kubernetes-the-hard-way --format json | jq '.external_ipv4_address.address' -r)

  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem

  kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin

  kubectl config use-context kubernetes-the-hard-way
}
```

## Проверка

Проверьте версию удаленного кластера Kubernetes:

```bash
kubectl version
```

> output

```
Client Version: version.Info{Major:"1", Minor:"32", GitVersion:"v1.32.3", GitCommit:"a0c1c0b2d2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b", GitTreeState:"clean", BuildDate:"2024-01-15T16:31:21Z", GoVersion:"go1.21.0", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"32", GitVersion:"v1.32.3", GitCommit:"a0c1c0b2d2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b", GitTreeState:"clean", BuildDate:"2024-01-15T16:25:06Z", GoVersion:"go1.21.0", Compiler:"gc", Platform:"linux/amd64"}
```

Выведите список нод удаленного кластера Kubernetes:

```bash
kubectl get nodes
```

> output

```
NAME     STATUS   ROLES    AGE     VERSION
node-0   Ready    <none>   2m35s   v1.32.3
node-1   Ready    <none>   2m35s   v1.32.3
```

## Дополнительная проверка

Проверьте компоненты кластера:

```bash
# Проверьте namespace kube-system
kubectl get pods -n kube-system

# Проверьте сервисы
kubectl get services -n kube-system

# Проверьте endpoints
kubectl get endpoints -n kube-system
```

## Настройка для других пользователей

Если вы хотите настроить доступ для других пользователей, создайте дополнительные kubeconfig файлы:

```bash
# Пример для пользователя developer
kubectl config set-credentials developer \
  --client-certificate=developer.pem \
  --client-key=developer-key.pem

kubectl config set-context developer-context \
  --cluster=kubernetes-the-hard-way \
  --user=developer

# Сохраните в отдельный файл
kubectl config view --flatten > developer-kubeconfig.yaml
```

Дальше: [Provisioning Pod Network Routes](12-pod-network-routes.md)
