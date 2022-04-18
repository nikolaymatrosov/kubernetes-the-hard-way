# Генерируем конфиги для аутентификации в Kubernetes

В этой лабораторной вы
сгенерируете [файлы конфигурации Kubernetes](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
, также известные как `kubeconfigs`. При помощи них клиенты находят сервера и аутентифицируются на серверах Kubernetes
API.

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
for instance in worker-0 worker-1 worker-2; do
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
worker-0.kubeconfig
worker-1.kubeconfig
worker-2.kubeconfig
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

Сгенерируем kubeconfig файл для сервиса `kube-controller-manager`:

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

Сгенерируем kubeconfig файл для сервиса `kube-scheduler`:

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

### The admin Kubernetes Configuration File

Сгенерируем kubeconfig файл для пользователя `admin`:

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
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

##       

## Разложим файлы конфигурации


Скопируем нужные файлы kubeconfig для `kubelet` и `kube-proxy` на каждый воркер:
Copy the appropriate `kubelet` and `kube-proxy` kubeconfig files to each worker instance:


```bash
for instance in worker-0 worker-1 worker-2; do
  EXTERNAL_IP=$(yc compute instance get ${instance} --format json | jq '.network_interfaces[0].primary_v4_address.one_to_one_nat.address' -r)
  for filename in ${instance}.kubeconfig kube-proxy.kubeconfig; do
    scp $filename yc-user@$EXTERNAL_IP:~/
  done
done
```

Скопируем нужные файлы kubeconfig для `kube-controller-manager` и `kube-scheduler` на каждую машину контроллера:
Copy the appropriate `kube-controller-manager` and `kube-scheduler` kubeconfig files to each controller instance:

```bash
for instance in controller-0 controller-1 controller-2; do
  EXTERNAL_IP=$(yc compute instance get ${instance} --format json | jq '.network_interfaces[0].primary_v4_address.one_to_one_nat.address' -r)
  for filename in admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig; do
    scp $filename yc-user@$EXTERNAL_IP:~/
  done
done
```

Дальше: [Генерируем Data Encryption конфиг и ключ](06-data-encryption-keys.md)
