# Настраиваем сетевые маршруты для подов

Поды назначенные на ноды получают IP-адрес из Pod CIDR диапазона ноды. В этот момент поды не могут общаться с подами
бегущими на других нодах, т.к. отсутствуют [сетевые маршруты](https://cloud.yandex.ru/docs/vpc/concepts/static-routes).

В этой лабораторной вы создадите маршруты для каждого воркера, который создает соответствие CIDR диапазона подов и
внутреннего IP-адреса ноды.

> Существуют
> и [другие способы](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this)
> добиться
> того же эффекта в сетевой модели Kubernetes.

## Важное

**Все команды в этой главе выполняются на jumpbox** - центральной машине управления кластером. 
Это упрощает процесс и обеспечивает централизованное управление сетевыми маршрутами.

## Статические маршруты

На этом шаге вы соберете всю информацию необходимую для того чтобы создать статические маршруты в
сети `kubernetes-the-hard-way`.

Выведите внутренний адрес и CIDR диапазона додов для каждого воркера:

```bash
# На jumpbox
for instance in node-0 node-1; do
  yc compute instance get ${instance} --full --format json | jq '.network_interfaces[0].primary_v4_address.address+"\t"+.metadata."pod-cidr"' -r
done 
```

> output

```
10.240.0.20     10.200.0.0/24
10.240.0.21     10.200.1.0/24
```

## Маршруты

Создаете сетевые маршруты для каждого инстанса:

```bash
routes=()
for i in 0 1; do
  routes+=(--route destination=10.200.${i}.0/24,next-hop=10.240.0.2${i})
done
yc vpc route-table create --name=kubernetes-route --network-name=kubernetes-the-hard-way ${routes[@]}
```

Посмотрите созданные машруты:

```bash
yc vpc route-table get kubernetes-route
```

> output

```
id: enp8***
folder_id: b1g***
created_at: "2022-04-18T10:19:52Z"
name: kubernetes-route
network_id: enp***
static_routes:
- destination_prefix: 10.200.0.0/24
  next_hop_address: 10.240.0.20
- destination_prefix: 10.200.1.0/24
  next_hop_address: 10.240.0.21
```

Соединить подсеть и таблицу маршрутиризации:

```bash
yc vpc subnet update kubernetes --route-table-name kubernetes-route
```

## Проверка маршрутов

Проверьте, что маршруты созданы корректно:

```bash
# Проверьте таблицу маршрутов
yc vpc route-table get kubernetes-route --format json | jq '.static_routes'

# Проверьте подсеть
yc vpc subnet get kubernetes --format json | jq '.route_table_id'
```

## Альтернатива: Больше worker nodes (для продакшена)

Для продакшен окружения с большим количеством worker nodes:

```bash
# Для 3 worker nodes
routes=()
for i in 0 1 2; do
  routes+=(--route destination=10.200.${i}.0/24,next-hop=10.240.0.2${i})
done
yc vpc route-table create --name=kubernetes-route --network-name=kubernetes-the-hard-way ${routes[@]}
```

Дальше: [Deploying the DNS Cluster Add-on](13-dns-addon.md)
