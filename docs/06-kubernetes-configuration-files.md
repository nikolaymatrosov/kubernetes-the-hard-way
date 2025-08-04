# Генерируем конфиги для аутентификации в Kubernetes

В этой лабораторной вы
сгенерируете [файлы конфигурации Kubernetes](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
, также известные как `kubeconfigs`. При помощи них клиенты находят сервера и аутентифицируются на серверах Kubernetes
API.

## Важное

**Все команды в этой главе выполняются на jumpbox** - центральной машине управления кластером. 
Это упрощает процесс и обеспечивает централизованное управление конфигурациями.

## Клиентские конфиги для аутентификации

На этом шаге вы сгенерируете файлы kubeconfig для клиентов `controller manager`, `kubelet`, `kube-proxy` и `scheduler`,
а также для пользователя `admin`.

### Kubernetes Public IP Address

Каждый kubeconfig требует сервер Kubernetes API, к которому будет подключаться. Для обеспечения высокой доступности
будем использовать IP-адрес, который назначим внешнему балансировщику нагрузки, который поставим перед серверами API
Kubernetes.

Получим IP адрес `kubernetes-the-hard-way`:
Retrieve the `kubernetes-the-hard-way` static IP address:

```bash
# На jumpbox
cd ~/kubernetes-the-hard-way/certificates

KUBERNETES_PUBLIC_ADDRESS=$(yc vpc address get kubernetes-the-hard-way --format json | jq '.external_ipv4_address.address' -r)
```

### Конфигурационный файл для kubelet

Когда вы генерируете kubeconfig для кублетов, должен быть использован клиентский сертификат соответсвующий имени ноды.
Таким образом вы можете быть уверены, что у кублета получится авторизоваться через
Kubernetes [Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/).

> Следующие команды команды должны выполняться из директории, которая использовалась в передыдущей
> лабораторной [Создание CA и генерация TLS сертификатов](04-certificate-authority.md) lab.

Сгенерируем kubeconfig файл для каждой ноды:

```bash
for instance in node-0 node-1; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

Результат:

```
node-0.kubeconfig
node-1.kubeconfig
```

### Файл конфигурации для kube-proxy

Сгенерируем kubeconfig файл для сервиса `kube-proxy`:

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}
```

Результат:

```
kube-proxy.kubeconfig
```

### Файл конфигурации для kube-controller-manager

Сгенерируем kubeconfig файл для `kube-controller-manager`:

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}
```

Результат:

```
kube-controller-manager.kubeconfig
```

### Файл конфигурации для kube-scheduler

Сгенерируем kubeconfig файл для `kube-scheduler`:

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}
```

Результат:

```
kube-scheduler.kubeconfig
```

### Файл конфигурации для admin

Сгенерируем kubeconfig файл для пользователя `admin`:

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}
```

Результат:

```
admin.kubeconfig
```

## Распределение конфигурационных файлов

Скопируйте конфигурационные файлы на соответствующие машины:

```bash
# Создайте архивы с конфигурациями
tar -czf server-configs.tar.gz \
  kube-controller-manager.kubeconfig kube-scheduler.kubeconfig

tar -czf node-0-configs.tar.gz \
  node-0.kubeconfig kube-proxy.kubeconfig

tar -czf node-1-configs.tar.gz \
  node-1.kubeconfig kube-proxy.kubeconfig

# Скопируйте архивы на машины
scp server-configs.tar.gz yc-user@<server-external-ip>:~/
scp node-0-configs.tar.gz yc-user@<node-0-external-ip>:~/
scp node-1-configs.tar.gz yc-user@<node-1-external-ip>:~/
```

## Проверка конфигураций

Проверьте, что все конфигурационные файлы созданы корректно:

```bash
# Проверьте конфигурацию admin
kubectl --kubeconfig=admin.kubeconfig version

# Проверьте конфигурацию node-0
kubectl --kubeconfig=node-0.kubeconfig version

# Проверьте конфигурацию node-1
kubectl --kubeconfig=node-1.kubeconfig version
```

Дальше: [Генерируем ключи шифрования данных](07-data-encryption-keys.md)
