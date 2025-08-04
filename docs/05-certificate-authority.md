# Создание CA и генерация TLS сертификатов

На этом шаге мы создадим [PKI](https://en.wikipedia.org/wiki/Public_key_infrastructure), используя CloudFlare's PKI
toolkit. Затем используем его чтобы настроить Certificate Authority и сгенерировать TLS сертификаты для ряда
компонентов: `etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `kubelet` и `kube-proxy`.

## Важное

**Все команды в этой главе выполняются на jumpbox** - центральной машине управления кластером. 
Это упрощает процесс и обеспечивает централизованное управление сертификатами.

## Certificate Authority

Начнем с того, что создадим Certificate Authority, который может позже быть использован для выпуска дополнительных TLS
сертификатов.

Генерируем файл конфигурации CA, сертификат и приватный(закрытый) ключ:

```bash
# Подключитесь к jumpbox
ssh yc-user@<jumpbox-external-ip>

# Перейдите в директорию для сертификатов
cd ~/kubernetes-the-hard-way
mkdir certificates
cd certificates

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

### Клиентские сертификаты для kubelet

Сгенерируйте клиентские сертификаты для каждого воркера:

```bash
for instance in node-0 node-1; do
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

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${instance}.kubernetes.internal,10.240.0.2${instance:4:1} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done
```

### Клиентский сертификат для kube-proxy

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

### Серверный сертификат для Kubernetes API

Получите внешний IP-адрес балансировщика:

```bash
KUBERNETES_PUBLIC_ADDRESS=$(yc vpc address get kubernetes-the-hard-way --format json | jq '.external_ipv4_address.address' -r)
```

Сгенерируйте сертификат для Kubernetes API:

```bash
{

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
  -hostname=10.32.0.1,10.240.0.10,10.240.0.20,10.240.0.21,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,kubernetes.default.svc.cluster.local \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

}
```

### Сертификат для Service Account

Сгенерируйте сертификат для Service Account:

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

## Распределение сертификатов

Скопируйте сертификаты на соответствующие машины:

```bash
# Создайте архив с сертификатами для server
tar -czf server-certificates.tar.gz \
  ca.pem ca-key.pem kubernetes.pem kubernetes-key.pem \
  service-account.pem service-account-key.pem

# Создайте архивы с сертификатами для worker nodes
for instance in node-0 node-1; do
  tar -czf ${instance}-certificates.tar.gz \
    ca.pem ${instance}.pem ${instance}-key.pem \
    kube-proxy.pem kube-proxy-key.pem
done

# Скопируйте архивы на машины
scp server-certificates.tar.gz yc-user@<server-external-ip>:~/
scp node-0-certificates.tar.gz yc-user@<node-0-external-ip>:~/
scp node-1-certificates.tar.gz yc-user@<node-1-external-ip>:~/
```

## Проверка сертификатов

Проверьте, что все сертификаты созданы корректно:

```bash
# Проверьте CA сертификат
openssl x509 -in ca.pem -text -noout

# Проверьте Kubernetes API сертификат
openssl x509 -in kubernetes.pem -text -noout

# Проверьте admin сертификат
openssl x509 -in admin.pem -text -noout
```

Дальше: [Генерируем конфиги для аутентификации в Kubernetes](06-kubernetes-configuration-files.md)
