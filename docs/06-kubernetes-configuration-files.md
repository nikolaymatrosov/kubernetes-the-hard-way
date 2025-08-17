# Генерируем конфиги для аутентификации в Kubernetes

В этой лабораторной вы
сгенерируете [файлы конфигурации Kubernetes](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
, также известные как `kubeconfigs`. При их помощи клиенты находят сервера и аутентифицируются на серверах Kubernetes
API.

## Важное

**Все команды в этой главе выполняются на jumpbox** - центральной машине управления кластером. 
Это упрощает процесс и обеспечивает централизованное управление конфигурациями.

## Клиентские конфиги для аутентификации

На этом шаге вы сгенерируете файлы kubeconfig для клиентов `kubelet` и пользователя `admin`.

### Конфигурационный файл для kubelet

Когда вы генерируете kubeconfig для кублетов, должен быть использован клиентский сертификат соответствующий имени ноды.
Таким образом вы можете быть уверены, что у кублета получится авторизоваться через
Kubernetes [Node Authorizer](https://kubernetes.io/docs/reference/access-authn-authz/node/).

> Следующие команды должны выполняться из директории, которая использовалась в предыдущей
> лабораторной [Создание CA и генерация TLS сертификатов](05-certificate-authority.md).

Сгенерируем kubeconfig файл для рабочих нод `node-0` и `node-1`:

```bash
for host in node-0 node-1; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=${host}.kubeconfig

  kubectl config set-credentials system:node:${host} \
    --client-certificate=${host}.crt \
    --client-key=${host}.key \
    --embed-certs=true \
    --kubeconfig=${host}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${host} \
    --kubeconfig=${host}.kubeconfig

  kubectl config use-context default \
    --kubeconfig=${host}.kubeconfig
done
```
> Output
```text
Cluster "kubernetes-the-hard-way" set.
User "system:node:node-0" set.
Context "default" created.
Switched to context "default".
Cluster "kubernetes-the-hard-way" set.
User "system:node:node-1" set.
Context "default" created.
Switched to context "default".
```
Проверим, что kubeconfig файлы были созданы:

```bash
ls -1 node-*.kubeconfig
```

Результат:

```text
node-0.kubeconfig
node-1.kubeconfig
```

### Файл конфигурации для kube-proxy

Сгенерируем kubeconfig файл для сервиса `kube-proxy`:

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.crt \
    --client-key=kube-proxy.key \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-proxy.kubeconfig
}
```

Результат:

```text
kube-proxy.kubeconfig
```

### Файл конфигурации для kube-controller-manager

Сгенерируем kubeconfig файл для `kube-controller-manager`:

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.crt \
    --client-key=kube-controller-manager.key \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-controller-manager.kubeconfig
}
```

Результат:

```text
kube-controller-manager.kubeconfig
```

### Файл конфигурации для kube-scheduler

Сгенерируем kubeconfig файл для `kube-scheduler`:

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.crt \
    --client-key=kube-scheduler.key \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-scheduler.kubeconfig
}
```

Результат:

```text
kube-scheduler.kubeconfig
```

### Файл конфигурации для admin

Сгенерируем kubeconfig файл для пользователя `admin`:

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.crt \
    --client-key=admin.key \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default \
    --kubeconfig=admin.kubeconfig
}
```

Результат:

```text
admin.kubeconfig
```

## Распространение файлов конфигурации Kubernetes

Скопируем kubeconfig файлы `kubelet` и `kube-proxy` на машины `node-0` и `node-1`:

```bash
for host in node-0 node-1; do
  ssh root@${host} "mkdir -p /var/lib/{kube-proxy,kubelet}"

  scp kube-proxy.kubeconfig \
    root@${host}:/var/lib/kube-proxy/kubeconfig

  scp ${host}.kubeconfig \
    root@${host}:/var/lib/kubelet/kubeconfig
done
```

Скопируем kubeconfig файлы `kube-controller-manager` и `kube-scheduler` на машину `server`:

```bash
scp admin.kubeconfig \
  kube-controller-manager.kubeconfig \
  kube-scheduler.kubeconfig \
  root@server:~/
```

Далее: [Генерация ключа и конфига шифрования данных](07-data-encryption-keys.md)
