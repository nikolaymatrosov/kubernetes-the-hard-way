# Создание виртуальных машин

На этом шаге вы создадите виртуальные машины для развертывания кластера Kubernetes с использованием упрощенной архитектуры.

## Упрощенная архитектура

В современной версии туториала мы используем упрощенную архитектуру для лучшего понимания:

- **Jumpbox**: Центральная машина для управления кластером (уже создана)
- **Server**: Один control plane узел (вместо трех для упрощения)
- **Worker nodes**: Два рабочих узла для запуска подов

Это делает туториал более доступным для изучения, сохраняя при этом все ключевые концепции.

## Требования к ресурсам

### Рекомендуемые типы машин для Yandex Cloud

| Машина | Тип | vCPU | RAM | Диск | Назначение |
|--------|-----|------|-----|------|------------|
| jumpbox | standard-v3 | 1 | 2GB | 20GB | Управление кластером |
| server | standard-v3 | 2 | 4GB | 50GB | Control plane |
| node-0 | standard-v3 | 2 | 4GB | 50GB | Worker node |
| node-1 | standard-v3 | 2 | 4GB | 50GB | Worker node |

> **Примечание**: Для production окружений рекомендуется использовать минимум 3 control plane узла для высокой доступности.

## Создание Control Plane узла

Создайте виртуальную машину для control plane:

```bash
yc compute instance create \
  --name server \
  --zone ru-central1-a \
  --network-interface subnet-name=kubernetes,security-group-ids=kubernetes-the-hard-way-allow-internal,kubernetes-the-hard-way-allow-external \
  --memory 4 \
  --cores 2 \
  --core-fraction 20 \
  --disk-type network-ssd \
  --disk-size 50 \
  --create-boot-disk type=network-ssd,image-folder-id=standard-images,image-family=debian-12 \
  --ssh-key ~/.ssh/id_rsa.pub \
  --platform-id standard-v3
```

## Создание Worker узлов

Создайте первый worker узел:

```bash
yc compute instance create \
  --name node-0 \
  --zone ru-central1-a \
  --network-interface subnet-name=kubernetes,security-group-ids=kubernetes-the-hard-way-allow-internal \
  --memory 4 \
  --cores 2 \
  --core-fraction 20 \
  --disk-type network-ssd \
  --disk-size 50 \
  --create-boot-disk type=network-ssd,image-folder-id=standard-images,image-family=debian-12 \
  --ssh-key ~/.ssh/id_rsa.pub \
  --platform-id standard-v3
```

Создайте второй worker узел:

```bash
yc compute instance create \
  --name node-1 \
  --zone ru-central1-a \
  --network-interface subnet-name=kubernetes,security-group-ids=kubernetes-the-hard-way-allow-internal \
  --memory 4 \
  --cores 2 \
  --core-fraction 20 \
  --disk-type network-ssd \
  --disk-size 50 \
  --create-boot-disk type=network-ssd,image-folder-id=standard-images,image-family=debian-12 \
  --ssh-key ~/.ssh/id_rsa.pub \
  --platform-id standard-v3
```

## Проверка создания машин

Выведите список всех виртуальных машин:

```bash
yc compute instance list
```

> output

```
+----------------------+--------+---------------+---------+---------------+-------------+
|          ID          |  NAME  |    ZONE ID    | STATUS  |  EXTERNAL IP  | INTERNAL IP |
+----------------------+--------+---------------+---------+---------------+-------------+
| fhm****************f | jumpbox| ru-central1-a | RUNNING | 51.250.**.*** | 10.240.0.5  |
| abc****************d | server | ru-central1-a | RUNNING | 51.250.**.*** | 10.240.0.10 |
| def****************g | node-0 | ru-central1-a | RUNNING |               | 10.240.0.20 |
| ghi****************j | node-1 | ru-central1-a | RUNNING |               | 10.240.0.21 |
+----------------------+--------+---------------+---------+---------------+-------------+
```

## Настройка внутренних IP адресов

Для удобства работы создайте переменные с внутренними IP адресами:

```bash
# Получите внутренние IP адреса
SERVER_INTERNAL_IP=$(yc compute instance get server --format json | jq '.network_interfaces[0].primary_v4_address.address' -r)
NODE_0_INTERNAL_IP=$(yc compute instance get node-0 --format json | jq '.network_interfaces[0].primary_v4_address.address' -r)
NODE_1_INTERNAL_IP=$(yc compute instance get node-1 --format json | jq '.network_interfaces[0].primary_v4_address.address' -r)

echo "Server Internal IP: $SERVER_INTERNAL_IP"
echo "Node-0 Internal IP: $NODE_0_INTERNAL_IP"
echo "Node-1 Internal IP: $NODE_1_INTERNAL_IP"
```

## Проверка сетевой связности

Проверьте, что все машины могут общаться между собой:

```bash
# Подключитесь к jumpbox
ssh yc-user@$(yc compute instance get jumpbox --format json | jq '.network_interfaces[0].primary_v4_address.one_to_one_nat.address' -r)

# Проверьте доступность server
ping -c 3 $SERVER_INTERNAL_IP

# Проверьте доступность worker узлов
ping -c 3 $NODE_0_INTERNAL_IP
ping -c 3 $NODE_1_INTERNAL_IP
```

## Подготовка к следующему шагу

Теперь у вас есть:
- ✅ Jumpbox для централизованного управления
- ✅ Control plane узел (server)
- ✅ Два worker узла (node-0, node-1)
- ✅ Настроенная сеть с группами безопасности
- ✅ Внутренние IP адреса для всех машин

В следующем шаге мы настроим инфраструктуру открытых ключей и создадим TLS сертификаты для безопасного общения между компонентами кластера.

Дальше: [Настройка Certificate Authority](05-certificate-authority.md) 