# Настройка сетевых маршрутов для подов

Поды, назначенные на узел, получают IP-адрес из диапазона Pod CIDR узла. В этот момент поды не могут общаться с другими
подами, работающими на разных узлах, из-за отсутствующих
сетевых маршрутов.

В этой лабораторной работе вы создадите маршрут для каждого рабочего узла, который сопоставляет диапазон Pod CIDR узла с
внутренним IP-адресом узла.

> Существуют [другие способы](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this)
реализации сетевой модели Kubernetes.

## Таблица маршрутизации

В этом разделе вы соберете информацию, необходимую для создания маршрутов в сети VPC `kubernetes-the-hard-way`.

Выведите внутренний IP-адрес и диапазон Pod CIDR для каждого рабочего экземпляра:

```bash
{
  SERVER_IP=$(grep server machines.txt | cut -d " " -f 1)
  NODE_0_IP=$(grep node-0 machines.txt | cut -d " " -f 1)
  NODE_0_SUBNET=$(grep node-0 machines.txt | cut -d " " -f 4)
  NODE_1_IP=$(grep node-1 machines.txt | cut -d " " -f 1)
  NODE_1_SUBNET=$(grep node-1 machines.txt | cut -d " " -f 4)
}
```

```bash
ssh root@server <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF
```

```bash
ssh root@node-0 <<EOF
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF
```

```bash
ssh root@node-1 <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
EOF
```

## Проверка

```bash
ssh root@server ip route
```

```text
default via XXX.XXX.XXX.XXX dev eth0 
10.200.0.0/24 via XXX.XXX.XXX.XXX dev eth0 
10.200.1.0/24 via XXX.XXX.XXX.XXX dev eth0 
XXX.XXX.XXX.0/24 dev eth0 proto kernel scope link src XXX.XXX.XXX.XXX 
```

```bash
ssh root@node-0 ip route
```

```text
default via XXX.XXX.XXX.XXX dev eth0 
10.200.1.0/24 via XXX.XXX.XXX.XXX dev eth0 
XXX.XXX.XXX.0/24 dev eth0 proto kernel scope link src XXX.XXX.XXX.XXX 
```

```bash
ssh root@node-1 ip route
```

```text
default via XXX.XXX.XXX.XXX dev eth0 
10.200.0.0/24 via XXX.XXX.XXX.XXX dev eth0 
XXX.XXX.XXX.0/24 dev eth0 proto kernel scope link src XXX.XXX.XXX.XXX 
```

Далее: [Смоук тест](13-smoke-test.md)
