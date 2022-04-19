# Создание виртуальных машин

Kubernetes требуется набор ВМ для размещения control plane Kubernetes и рабочих узлов, на которых находятся в итоге и
будут бежать контейнеры. В этой лабораторной работе вы подготовите вычислительные ресурсы, необходимые для запуска
безопасного и высокодоступного кластера Kubernetes в одной
[зоне доступности](https://cloud.yandex.ru/docs/overview/concepts/geo-scope)

## Сеть

[Сетевая модель](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) Kubernetes
предполагает плоскую сеть, в которой контейнеры и ноды могут общаться между собой. В случае, если вы хотели бы такого
избежать в Kubernetes есть [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
, которые определяют как будут сгруппированы контейнеры и помогут ограничить, какие контейнеры могу общаться друг ст
другом или внешними сетевыми эндпоинтами.

> Настройка сетевых политик выходит за рамки этого туториала.

### Виртуальная облачная сеть

Рассмотрим настройку [Virtual Private Cloud](https://cloud.yandex.ru/docs/vpc/concepts/) для нашего кластера.

Создайте сеть с названием `kubernetes-the-hard-way`:

```bash
yc vpc network create --name=kubernetes-the-hard-way
```

[Подсеть](https://cloud.yandex.ru/docs/vpc/concepts/network#subnet) должна быть создана с достаточным количеством
IP-адресов, чтобы в ней можно было назначить приватный IP-адрес для каждой ноды

Создайте подсеть `kubernetes` в сети `kubernetes-the-hard-way`:

```bash
yc vpc subnet create \
  --name=kubernetes \
  --network-name=kubernetes-the-hard-way \
  --zone=ru-central1-a \
  --range=10.240.0.0/24
```

> Диапазона `10.240.0.0/24` будет достаточно для 254 виртуальных машин.

### Группы безопасности

> Сейчас группы безопасности находятся в превью и вам придется запросить доступ к этой функциональности.

Создайте группу безопасности разрешающую внутреннее общение по всем протоколам:

```bash
yc vpc security-group create --name=kubernetes-the-hard-way-allow-internal \
    --network-name=kubernetes-the-hard-way \
    --rule description=internal_allow,protocol=any,direction=ingress,port=any,predefined=self_security_group \
    --rule description=internal_allow,protocol=any,direction=egress,port=any,predefined=self_security_group
```

Создайте группу безопасности разрешающую внешний входящий и исходящий SSH, ICMP, HTTPS трафик :

```bash
yc vpc security-group create --name=kubernetes-the-hard-way-allow-external \
    --network-name=kubernetes-the-hard-way \
    --rule description=internal_allow,protocol=any,direction=egress,port=any,v4-cidrs=0.0.0.0/0 \
    --rule description=internal_allow,protocol=tcp,direction=ingress,port=22,v4-cidrs=0.0.0.0/0 \
    --rule description=internal_allow,protocol=tcp,direction=ingress,port=6443,v4-cidrs=0.0.0.0/0
```

Создайте группу безопасности разрешающую внешний входящий и исходящий SSH, ICMP, HTTPS трафик :

```bash
yc vpc security-group create --name=kubernetes-the-hard-way-allow-balancer \
    --network-name=kubernetes-the-hard-way \
    --rule description=internal_allow,protocol=tcp,direction=egress,port=80,predefined=loadbalancer_healthchecks \
    --rule description=internal_allow,protocol=tcp,direction=ingress,port=80,predefined=loadbalancer_healthchecks
```

> Kubernetes API Servers будут доступны
> через [Network Load Balancer](https://cloud.yandex.ru/docs/network-load-balancer/concepts/)

Получить список групп безопасности в текущем облаке можно так:

```bash
yc vpc security-group list
```

### Публичный IP адрес для Kubernetes

Зарезервируем внешний статический IP адрес, который потом используем на балансере для доступа к Kubernetes API.

```bash
yc vpc address create --name=kubernetes-the-hard-way --external-ipv4=zone=ru-central1-a
```

Убедитесь, что адрес был создан.

```bash
yc vpc address list
```

> output

```
+----------------------+-------------------------+----------------+----------+-------+
|          ID          |          NAME           |    ADDRESS     | RESERVED | USED  |
+----------------------+-------------------------+----------------+----------+-------+
| e9b***************** | kubernetes-the-hard-way | 51.250.***.*** | true     | false |
+----------------------+-------------------------+----------------+----------+-------+
```
## Создадим внутреннюю DNS зону

```bash
NETWORK_ID=$(yc vpc network get kubernetes-the-hard-way --format json | jq '.id' -r)
yc dns zone create --name=kubernetes --zone=. --private-visibility=true --network-ids=${NETWORK_ID}
```


## Доступ по SSH

Для доступа по SSH будет использоваться пара ключей публичный и приватный. Публичный будет передан при создании ВМ в
команду `yc`. Если у вас нет ключей, то вы можете создать их
по [инструкции](https://cloud.yandex.ru/docs/compute/operations/vm-connect/ssh#creating-ssh-keys)

## Виртуальные машины

В этой лабораторной мы будем использовать ВМ на [Ubuntu Server](https://www.ubuntu.com/server) 20.04, который имеет
хорошую поддержку [containerd container runtime](https://github.com/containerd/containerd). Каждый инстанс будет
развернут с внешним IP, чтобы облегчить процесс настройки Kubernetes.


> Суммарный объем создаваемых дисков превышает стартовую квоту. Вам потребуется запросить
> ее [повышение](https://console.cloud.yandex.ru/cloud?section=quotas

### Kubernetes контроллеры

Создадим три виртуальные машины, на которых разместим Kubernetes control plane.

```bash
for i in 0 1 2; do
  name=controller-${i}
  DNS_ZONE_ID=$(yc dns zone get kubernetes --format json | jq '.id' -r)
  INTERNAL_SG=$(yc vpc security-group get kubernetes-the-hard-way-allow-internal --format json | jq '.id' -r)
  EXTERNAL_SG=$(yc vpc security-group get kubernetes-the-hard-way-allow-external --format json | jq '.id' -r)
  BALANCER_SG=$(yc vpc security-group get kubernetes-the-hard-way-allow-balancer --format json | jq '.id' -r)
  yc compute instance create \
    --name ${name} \
    --zone ru-central1-a \
    --cores 2 \
    --memory 8 \
    --network-interface security-group-ids=\[${INTERNAL_SG},${EXTERNAL_SG},${BALANCER_SG}\],subnet-name=kubernetes,nat-ip-version=ipv4,ipv4-address=10.240.0.1${i},dns-record-spec=\{name=${name}.,dns-zone-id=${DNS_ZONE_ID},ttl=300\} \
    --create-boot-disk type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2004-lts,size=200 \
    --ssh-key ~/.ssh/id_rsa.pub \
    --labels role=controller \
    --hostname ${name} \
    --async
done
```

### Воркеры Kubernetes

Для каждого воркера нужно аллоцировать подсеть для pod'ов из CIDR диапазона доступного кластеру Kubernetes. Они будут
использованы в одном следующих шагов для настройки сетевого взаимодействия контейнеров.
Чтобы передать эту информацию внутрь ВМ, мы воспользуемся метадатой машины, куда передадим нужное значение `pod-cidr`.

> CIDR диапазон кластера Kubernetes задается флагом `--cluster-cidr` контроллер менеджера. В этом туториале мы будем
> использовать CIDR диапазон `10.200.0.0/16`, который поддерживает до 254 подсетей.

Создадим три виртуальные машины, которые будут обслуживать воркер ноды Kubernetes:

```bash
for i in 0 1 2; do
  name=worker-${i}
  DNS_ZONE_ID=$(yc dns zone get kubernetes --format json | jq '.id' -r)
  INTERNAL_SG=$(yc vpc security-group get kubernetes-the-hard-way-allow-internal --format json | jq '.id' -r)
  EXTERNAL_SG=$(yc vpc security-group get kubernetes-the-hard-way-allow-external --format json | jq '.id' -r)
  yc compute instance create \
    --name ${name} \
    --zone ru-central1-a \
    --cores 2 \
    --memory 8 \
    --network-interface security-group-ids=\[${INTERNAL_SG},${EXTERNAL_SG}\],subnet-name=kubernetes,nat-ip-version=ipv4,ipv4-address=10.240.0.2${i},dns-record-spec=\{name=${name}.,dns-zone-id=${DNS_ZONE_ID},ttl=300\} \
    --create-boot-disk type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2004-lts,size=200 \
    --ssh-key ~/.ssh/id_rsa.pub \
    --labels role=worker \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --hostname ${name} \
    --async
done
```

## Исправление DNS

```bash
for instance in controller-0 controller-1 controller-2 worker-0 worker-1 worker-2; do
  EXTERNAL_IP=$(yc compute instance get ${instance} --format json | jq '.network_interfaces[0].primary_v4_address.one_to_one_nat.address' -r)
  ssh yc-user@${EXTERNAL_IP} -t "sudo DEBIAN_FRONTEND=noninteractive apt-get install -y resolvconf"
  ssh yc-user@${EXTERNAL_IP} "cat <<EOF | sudo tee /etc/resolvconf/resolv.conf.d/tail
nameserver 10.240.0.2
EOF"
  yc compute instance restart --name=${instance} --async
done 
```

### Проверка

Выведите список виртуальных машин.
List the compute instances in your default compute zone:

```bash
yc compute instance list
```

> output

```
+----------------------+--------------+---------------+---------+---------------+-------------+
|          ID          |     NAME     |    ZONE ID    | STATUS  |  EXTERNAL IP  | INTERNAL IP |
+----------------------+--------------+---------------+---------+---------------+-------------+
| fhm****************f | controller-1 | ru-central1-a | RUNNING | 51.250.**.*** | 10.240.0.11 |
| fhm****************4 | controller-2 | ru-central1-a | RUNNING | 51.250.**.*** | 10.240.0.12 |
| fhm****************k | controller-0 | ru-central1-a | RUNNING | 51.250.**.*** | 10.240.0.10 |
| fhm****************o | worker-2     | ru-central1-a | RUNNING | 51.250.**.*** | 10.240.0.22 |
| fhm****************p | worker-1     | ru-central1-a | RUNNING | 51.250.**.*** | 10.240.0.21 |
| fhm****************3 | worker-0     | ru-central1-a | RUNNING | 51.250.**.*** | 10.240.0.20 |
+----------------------+--------------+---------------+---------+---------------+-------------+
```

## Проверка доступа по SSH

Давайте проверим есть ли у нас доступ к виртуальной машине `controller-0`.
Для этого скопируем ее внешний ip в команду:

```bash
ssh yc-user@51.250.**.***
```


Дальше: [Создание CA и генерация TLS сертификатов](04-certificate-authority.md)
