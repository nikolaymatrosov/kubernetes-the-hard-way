# Развертывание кластера etcd

Компоненты Kubernetes не содержат состояния(stateless) и хранят его в [etcd](https://github.com/etcd-io/etcd).
В этой лабораторной вы развернете кластер etcd из трех нод и сконфигурируете его высокую доступность и удаленный доступ.

## Важное

Команды из этой лабораторной должный выполняться на каждом инстансе контроллера: `controller-0`, `controller-1`
и `controller-2`. Зайдите на каждый инстанс.

### Параллельное выполнение команд через tmux

[tmux](https://github.com/tmux/tmux/wiki) может быть использован для параллельного выполнения команд на нескольких
инстансах. Смотри [Параллельное выполнение команд в tmux](01-prerequisites.md)

## Развертывание члена кластера etcd

### Скачайте и установите etcd

Скачайте официальный релиз [etcd](https://github.com/etcd-io/etcd) с GitHub:

```bash
ETCD_VER=v3.5.3
wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz"
```

Распакуйте и установите `etcd` сервер и утилиту командной строки `etcdctl`: 

```bash
{
  tar -xvf etcd-${ETCD_VER}-linux-amd64.tar.gz
  sudo mv etcd-${ETCD_VER}-linux-amd64/etcd* /usr/local/bin/
}
```

### Сконфигурируйте сервер etcd

```bash
{
  sudo mkdir -p /etc/etcd /var/lib/etcd
  sudo chmod 700 /var/lib/etcd
  sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
}
```

Внутренний IP-адрес будет использоваться для того чтобы обрабатывать клиенские запросы и общаться с соседями по кластеру etcd.
Получить внутренний IP-адрес текущего инстанса можно из метадаты: 

```bash
INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)
```

Каждый член etcd должен иметь имя уникальное в рамках etcd кластера. Установите имя в etcd совпадающее с именем хоста. 

Создайте `etcd.service` файл описания юнита в systemd:

```bash
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${HOSTNAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://10.240.0.10:2380,controller-1=https://10.240.0.11:2380,controller-2=https://10.240.0.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Запустите сервер etcd

```bash
{
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
}
```

> Не забудьте, что все команды выше нужно выполнить на всех хостах `controller-0`, `controller-1` и `controller-2`.

## Проверка

Выведем список членов кластера etcd:

```bash
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

> вывод

```
3a57933972cb5131, started, controller-2, https://10.240.0.12:2380, https://10.240.0.12:2379, false
f98dc20bce6225a0, started, controller-0, https://10.240.0.10:2380, https://10.240.0.10:2379, false
ffed16798470cab5, started, controller-1, https://10.240.0.11:2380, https://10.240.0.11:2379, false
```

Дальше: [Развертывание Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)
