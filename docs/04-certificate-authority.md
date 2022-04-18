# Создание CA и генерация TLS сертификатов

На этом шаге мы создадим [PKI](https://en.wikipedia.org/wiki/Public_key_infrastructure), используя CloudFlare's PKI
toolkit. Затем используем его чтобы настроить Certificate Authority и сгенерировать TLS сертификаты для ряда
компонентов: `etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `kubelet` и `kube-proxy`.

## Certificate Authority

Начнем с того, что создадим Certificate Authority, который может позже быть использован для выпуска дополнительных TLS
сертификатов.

Генерируем файл конфигурации CA, сертификат и приватный(закрытый) ключ:

```bash
{

cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}
```

В результате мы получим следующие файлы:

```
ca.csr
ca.pem
ca-config.json
ca-csr.json
ca-key.pem
```

## Клиентские и серверные сертификаты

Теперь вы сгенерируете клиентские и серверные сертификаты для каждого из компонентов Kubernetes и клиентский сертификат
для пользователя `admin` в Kubernetes.

### Клиентский сертификат для админа

Сгенерируйте для пользователя `admin` сертификат и приватный ключ:

```bash
{

cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin

}
```

Результат:

```
admin.csr
admin.pem
admin-csr.json
admin-key.pem
```

### Клиентский сертификат для Kubelet

Kubernetes использует [специальный режим авторизации](https://kubernetes.io/docs/admin/authorization/node/), который
называется `Node Authorizer`. Он используется дя авторизации в Kubernetes API запросов сделанных конкретно
[Kubelets](https://kubernetes.io/docs/concepts/overview/components/#kubelet). Чтобы быть авторизованным Node
Authorizer кублеты должны использовать данные для авторизации, которые идентифицируют их как членов
группы `system:nodes`, с именем `system:node:<nodeName>`. На этом шаге вы создадите по сертификату для каждого
воркера Kubernetes, такому, что он будет удовлетворять описанным выше условиям от Node Authorizer.

Сгенерируйте сертификат и приватный ключ для каждого воркера Kubernetes:

```bash
for instance in worker-0 worker-1 worker-2; do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

EXTERNAL_IP=$(yc compute instance get ${instance} --format json | jq '.network_interfaces[0].primary_v4_address.one_to_one_nat.address' -r)

INTERNAL_IP=$(yc compute instance get ${instance} --format json | jq '.network_interfaces[0].primary_v4_address.address' -r)

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done
```

Результаты:

```
worker-0-key.pem
worker-0.pem
worker-1-key.pem
worker-1.pem
worker-2-key.pem
worker-2.pem
```

### Сертификат для клиента Controller Manager

Сгенерируйте клиентский сертификат и приватный ключ для `kube-controller-manager`:

```bash
{

cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}
```

Результат:

```
kube-controller-manager-key.pem
kube-controller-manager.pem
```

### Сертификат для клиента Kube Proxy

Сгенерируйте клиентский сертификат и приватный ключ для `kube-proxy`:

```bash
{

cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}
```

Результат:

```
kube-proxy-key.pem
kube-proxy.pem
```

### Клиентский сертификат для Scheduler

Сгенерируйте клиентский сертификат и приватный ключ для `kube-scheduler`:

```bash
{

cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}
```

Результат:

```
kube-scheduler-key.pem
kube-scheduler.pem
```

### Сертификат для сервера Kubernetes API

Статический IP-адрес `kubernetes-the-hard-way`, который мы создали ранее, будет включен в список SAN(Subject Alternative
Names)
в сертификате для сервера Kubernetes API.
Это обеспечит возможность проверки сертификата удаленными клиентами.

Сгенерируйте сертификат и приватный ключ для сервера Kubernetes API.

```bash
{

KUBERNETES_PUBLIC_ADDRESS=$(yc vpc address get kubernetes-the-hard-way --format json | jq '.external_ipv4_address.address' -r)

KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

}
```

> Серверу Kubernetes API автоматически присваивается имя `kubernetes` во внутреннем DNS, которое будет указывать на
> первый
> IP-адрес (`10.32.0.1`) из диапазона (`10.32.0.0/24`), который будет зарезервирован для внутренних сервисов кластера.
> В ходе
> шага [control plane bootstrapping](08-bootstrapping-kubernetes-controllers.md#configure-the-kubernetes-api-server)

Результат:

```
kubernetes-key.pem
kubernetes.pem
```

## Пара ключей для сервисных аккаунтов

Kubernetes Controller Manager использует пару ключей, для того чтобы генерировать и подписывать токены сервисных
аккаунтов, как описано в [этом разделе документации](https://kubernetes.io/docs/admin/service-accounts-admin/).

Сгенерируйте сертификат и приватный ключ `service-account`:

```bash
{

cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

}
```

Результат:

```
service-account-key.pem
service-account.pem
```

## Distribute the Client and Server Certificates

## Разложите клиентские и серверные сертификаты на ВМ

Скопируйте сертификаты и приватные ключи на соответсвующие им инстансы:

```bash
for instance in worker-0 worker-1 worker-2; do
  EXTERNAL_IP=$(yc compute instance get ${instance} --format json | jq '.network_interfaces[0].primary_v4_address.one_to_one_nat.address' -r)
  for filename in ca.pem ${instance}.pem ${instance}-key.pem; do
    scp $filename yc-user@$EXTERNAL_IP:~/
  done
done
```

Скопируйте неоюходимые сертификаты и приватные ключи на соответсвующие им инстансы контороллера Kubernetes:

```bash
for instance in controller-0 controller-1 controller-2; do
  EXTERNAL_IP=$(yc compute instance get ${instance} --format json | jq '.network_interfaces[0].primary_v4_address.one_to_one_nat.address' -r)
  for filename in ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem service-account-key.pem service-account.pem; do
    scp $filename yc-user@$EXTERNAL_IP:~/
  done
done
```

> Клиентские сертификаты `kube-proxy`, `kube-controller-manager`, `kube-scheduler` и `kubelet` понадобятся нам в
> следующей лабораторной, для того чтобы сгенерировать конфигурационные файлы для аутентификации.

Дальше: [Генерируем конфиги для аутентификации в Kubernetes](05-kubernetes-configuration-files.md)
