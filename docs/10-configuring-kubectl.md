# Настраиваем удаленный доступ kubectl

В этой лабораторной вы сгенерируете kubeconfig для утилиты `kubectl`, основанный на данных для авторизации
пользователя `admin`.

> Выполняйте команды из директории где у вас лежат сертификаты для `admin`.

## Конфигурационный файл

Каждый kubeconfig требует API сервер Kubernetes к которому будет подключаться. Для большей доступности мы используем IP
балансера.

Сгенерируйте kubeconfig подходящий, чтобы авторизоваться от имени пользователя `admin`:

```bash
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
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:31:21Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:25:06Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
```

Выведите список нод удаленного кластера Kubernetes:

```bash
kubectl get nodes
```

> output

```
NAME       STATUS   ROLES    AGE     VERSION
worker-0   Ready    <none>   2m35s   v1.21.0
worker-1   Ready    <none>   2m35s   v1.21.0
worker-2   Ready    <none>   2m35s   v1.21.0
```

Дальше: [Provisioning Pod Network Routes](11-pod-network-routes.md)
