# Развертывание кластера etcd

Компоненты Kubernetes не содержат состояния и хранят состояние кластера в [etcd](https://github.com/etcd-io/etcd). В этой лабораторной вы развернете кластер etcd на одной ноде.

## Предварительные требования

Скопируйте бинарные файлы `etcd` и файлы systemd unit на машину `server`:

```bash
scp \
  downloads/controller/etcd \
  downloads/client/etcdctl \
  units/etcd.service \
  root@server:~/
```

Команды из этой лабораторной должны выполняться на машине `server`. Подключитесь к м��шине `server` используя команду `ssh`. Пример:

```bash
ssh root@server
```

## Развертывание кластера etcd

### Установка бинарных файлов etcd

Извлеките и установите сервер `etcd` и утилиту командной строки `etcdctl`:

```bash
{
  mv etcd etcdctl /usr/local/bin/
}
```

### Конфигурация сервера etcd

```bash
{
  mkdir -p /etc/etcd /var/lib/etcd
  chmod 700 /var/lib/etcd
  cp ca.crt kube-api-server.key kube-api-server.crt \
    /etc/etcd/
}
```

Каждый член etcd должен иметь уникальное имя в рамках кластера etcd. Установите имя etcd в соответствии с именем хоста текущего экземпляра вычислений:

Создайте файл systemd unit `etcd.service`:

```bash
mv etcd.service /etc/systemd/system/
```

### Запуск сервера etcd

```bash
{
  systemctl daemon-reload
  systemctl enable etcd
  systemctl start etcd
}
```

## Проверка

Выведите список членов кластера etcd:

```bash
etcdctl member list
```

```text
6702b0a34e2cfd39, started, controller, http://127.0.0.1:2380, http://127.0.0.1:2379, false
```

Дальше: [Развертывание панели управления Kubernetes](09-bootstrapping-kubernetes-controllers.md)
