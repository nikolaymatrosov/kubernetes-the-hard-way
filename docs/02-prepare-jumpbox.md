# Подготовка Jumpbox и инфраструктуры

На этом шаге вы создадите базовую инфраструктуру и jumpbox - центральную машину для управления кластером Kubernetes.

## Упрощенная архитектура

В современной версии туториала мы используем упрощенную архитектуру:
- **Jumpbox**: Центральная машина для управления кластером
- **Server**: Один control plane узел (вместо трех для упрощения)
- **Worker nodes**: Два рабочих узла для запуска подов

Это делает туториал более доступным для изучения, сохраняя при этом все ключевые концепции.

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

## Создание Jumpbox

### Jumpbox

Создайте jumpbox - центральную машину для управления кластером:

```bash
yc compute instance create \
  --name jumpbox \
  --zone ru-central1-a \
  --network-interface subnet-name=kubernetes,security-group-ids=kubernetes-the-hard-way-allow-external \
  --memory 2 \
  --cores 1 \
  --core-fraction 5 \
  --disk-type network-ssd \
  --disk-size 20 \
  --create-boot-disk type=network-ssd,image-folder-id=standard-images,image-family=debian-12 \
  --ssh-key ~/.ssh/id_rsa.pub \
  --platform-id standard-v3
```

### Проверка

Выведите список виртуальных машин:

```bash
yc compute instance list
```

> output

```
+----------------------+--------+---------------+---------+---------------+-------------+
|          ID          |  NAME  |    ZONE ID    | STATUS  |  EXTERNAL IP  | INTERNAL IP |
+----------------------+--------+---------------+---------+---------------+-------------+
| fhm****************f | jumpbox| ru-central1-a | RUNNING | 51.250.**.*** | 10.240.0.5  |
+----------------------+--------+---------------+---------+---------------+-------------+
```

## Проверка доступа по SSH

Давайте проверим есть ли у нас доступ к виртуальной машине `jumpbox`.
Для этого скопируем ее внешний ip в команду:

```bash
# Получите внешний IP jumpbox
JUMPBOX_IP=$(yc compute instance get jumpbox --format json | jq '.network_interfaces[0].primary_v4_address.one_to_one_nat.address' -r)
echo "Jumpbox IP: $JUMPBOX_IP"

# Проверьте доступ
ssh yc-user@$JUMPBOX_IP
```

## Подготовка к следующему шагу

Теперь у вас есть:
- ✅ Сеть `kubernetes-the-hard-way` с подсетью `kubernetes`
- ✅ Группы безопасности для внутреннего и внешнего трафика
- ✅ Статический IP для Kubernetes API
- ✅ DNS зона для внутреннего разрешения имен
- ✅ Jumpbox для централизованного управления

В следующем шаге мы установим инструменты на jumpbox и подготовим его для управления кластером.

Дальше: [Установка инструментов](03-client-tools.md)
