# Развертывание Kubernetes Control Plane

В этой лабораторной вы развернете панель управления Kubernetes. Следующие компоненты будут установлены на машине
`server`: Kubernetes API Server, Scheduler и Controller Manager.

## Предварительные требования

Подключитесь к `jumpbox` и скопируйте бинарные файлы Kubernetes и файлы systemd unit на машину `server`:

```bash
scp \
  downloads/controller/kube-apiserver \
  downloads/controller/kube-controller-manager \
  downloads/controller/kube-scheduler \
  downloads/client/kubectl \
  units/kube-apiserver.service \
  units/kube-controller-manager.service \
  units/kube-scheduler.service \
  configs/kube-scheduler.yaml \
  configs/kube-apiserver-to-kubelet.yaml \
  root@server:~/
```

Команды из этой лабораторной должны выполняться на машине `server`. Подключитесь к машине `server` используя команду
`ssh`. Пример:

```bash
ssh root@server
```

## Подготовка Kubernetes Control Plane

Создайте директорию конфигурации Kubernetes:

```bash
mkdir -p /etc/kubernetes/config
```

### Установка бинарных файлов Kubernetes Controller

Установите бинарные файлы Kubernetes:

```bash
{
  mv kube-apiserver \
    kube-controller-manager \
    kube-scheduler kubectl \
    /usr/local/bin/
}
```

### Конфигурация Kubernetes API Server

```bash
{
  mkdir -p /var/lib/kubernetes/

  mv ca.crt ca.key \
    kube-api-server.key kube-api-server.crt \
    service-accounts.key service-accounts.crt \
    encryption-config.yaml \
    /var/lib/kubernetes/
}
```

Создайте файл systemd unit `kube-apiserver.service`:

```bash
mv kube-apiserver.service \
  /etc/systemd/system/kube-apiserver.service
```

### Конфигурация Kubernetes Controller Manager

Переместите kubeconfig `kube-controller-manager` в нужное место:

```bash
mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

Создайте файл systemd unit `kube-controller-manager.service`:

```bash
mv kube-controller-manager.service /etc/systemd/system/
```

### Конфигурация Kubernetes Scheduler

Переместите kubeconfig `kube-scheduler` в нужное место:

```bash
mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

Создайте конфигурационный файл `kube-scheduler.yaml`:

```bash
mv kube-scheduler.yaml /etc/kubernetes/config/
```

Создайте файл systemd unit `kube-scheduler.service`:

```bash
mv kube-scheduler.service /etc/systemd/system/
```

### Запуск служб Controller

```bash
{
  systemctl daemon-reload

  systemctl enable kube-apiserver \
    kube-controller-manager kube-scheduler

  systemctl start kube-apiserver \
    kube-controller-manager kube-scheduler
}
```

> Подождите до 10 секунд для полной инициализации Kubernetes API Server.

Вы можете проверить, активен ли какой-либо из компонентов панели управления, используя команду `systemctl`. Например,
чтобы проверить, полностью ли инициализирован и активен `kube-apiserver`, выполните следующую команду:

```bash
systemctl is-active kube-apiserver
```

Для более детальной проверки статуса, которая включает дополнительную информацию о процессе и сообщения журнала,
используйте команду `systemctl status`:

```bash
systemctl status kube-apiserver
```

Если возникают ошибки или вы хотите просмотреть журналы для любого из компонентов панели управления, используйте команду
`journalctl`. Например, чтобы просмотреть журналы для `kube-apiserver`, выполните следующую команду:

```bash
journalctl -u kube-apiserver
```

### Проверка

На этом этапе компоненты панели управления Kubernetes должны быть запущены и работать. Проверьте это с помощью
инструмента командной строки `kubectl`:

```bash
kubectl cluster-info \
  --kubeconfig admin.kubeconfig
```

```text
Kubernetes control plane is running at https://127.0.0.1:6443
```

## RBAC для авторизации Kubelet

В этом разделе вы настроите разрешения RBAC, чтобы позволить Kubernetes API Server получать доступ к Kubelet API на
каждом рабочем узле. Доступ к Kubelet API необходим для получения метрик, журналов и выполнения команд в подах.

> Это руководство устанавливает флаг Kubelet `--authorization-mode` в `Webhook`. Режим Webhook использует
> API [SubjectAccessReview](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#checking-api-access)
> для определения авторизации.

Команды в этом разделе повлияют на весь кластер и должны выполняться только на машине `server`.

```bash
ssh root@server
```

Создайте [ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole)
`system:kube-apiserver-to-kubelet` с разрешениями для доступа к Kubelet API и выполнения наиболее распространенных
задач, связанных с управлением подами:

```bash
kubectl apply -f kube-apiserver-to-kubelet.yaml \
  --kubeconfig admin.kubeconfig
```

### Проверка

На этом этапе панель управления Kubernetes запущена и работает. Выполните следующие команды с машины `jumpbox`, чтобы
убедиться, что она работает:

Сделайте HTTP-запрос с Jumphost для получения информации о версии Kubernetes:

```bash
curl --cacert ca.crt \
  https://server.kubernetes.local:6443/version
```

```text
{
  "major": "1",
  "minor": "32",
  "gitVersion": "v1.32.3",
  "gitCommit": "32cc146f75aad04beaaa245a7157eb35063a9f99",
  "gitTreeState": "clean",
  "buildDate": "2025-03-11T19:52:21Z",
  "goVersion": "go1.23.6",
  "compiler": "gc",
  "platform": "linux/arm64"
}
```

Дальше: [Развертывание рабочих узлов Kubernetes](10-bootstrapping-kubernetes-workers.md)
