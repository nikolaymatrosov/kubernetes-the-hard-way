# Создание виртуальных машин

На этом шаге вы создадите виртуальные машины для развертывания кластера Kubernetes с использованием упрощенной
архитектуры.

## Упрощенная архитектура

В современной версии туториала мы используем упрощенную архитектуру для лучшего понимания:

- **Jumpbox**: Центральная машина для управления кластером (уже создана)
- **Server**: Один control plane узел (вместо трех для упрощения)
- **Worker nodes**: Два рабочих узла для запуска подов

Это делает туториал более доступным для изучения, сохраняя при этом все ключевые концепции.

## Требования к ресурсам

### Рекомендуемые типы машин для Yandex Cloud

| Машина  | Тип         | vCPU | RAM | Диск | Назначение           |
|---------|-------------|------|-----|------|----------------------|
| jumpbox | standard-v3 | 2    | 2GB | 20GB | Управление кластером |
| server  | standard-v3 | 2    | 4GB | 50GB | Control plane        |
| node-0  | standard-v3 | 2    | 4GB | 50GB | Worker node          |
| node-1  | standard-v3 | 2    | 4GB | 50GB | Worker node          |

> **Примечание**: Для production окружений рекомендуется использовать минимум 3 control plane узла для высокой
> доступности.

## Создание NAT-gateway для внешнего доступа
Создайте NAT-gateway для обеспечения внешнего доступа к вашим машинам:

```bash
yc vpc gateway create \
    --name kubernetes-the-hard-way-nat-gateway \
    --description "NAT gateway for Kubernetes The Hard Way"
```
### Создание Routing Table
Создайте таблицу маршрутизации для NAT-gateway:
```bash
GW_ID=$(yc vpc gateway get kubernetes-the-hard-way-nat-gateway --format json | jq -r '.id')
yc vpc route-table create \
    --name kubernetes-the-hard-way-nat-route-table \
    --network-name kubernetes-the-hard-way \
    --route destination=0.0.0.0/0,gateway-id=$GW_ID \
    --description "Routing table for NAT gateway"
```

### Привязка NAT-gateway к подсети
Привяжите NAT-gateway к подсети `kubernetes`:
```bash
yc vpc subnet update \
    --name kubernetes \
    --route-table-name kubernetes-the-hard-way-nat-route-table 
```

## Создание Control Plane узла

Создайте виртуальную машину для control plane, выполнив следующую команду с локальной машины:

```bash
SG_ID_EX=$(yc vpc security-group get kubernetes-the-hard-way-allow-external --format json | jq '.id' -r)
SG_ID_IN=$(yc vpc security-group get kubernetes-the-hard-way-allow-internal --format json | jq '.id' -r)

yc compute instance create \
  --hostname server \
  --name server \
  --zone ru-central1-a \
  --network-interface ipv4-address=10.240.0.10,nat-ip-version=ipv4,subnet-name=kubernetes,security-group-ids=\[$SG_ID_EX,$SG_ID_IN\] \
  --memory 4 \
  --cores 2 \
  --core-fraction 20 \
  --create-boot-disk size=50,type=network-ssd,image-folder-id=standard-images,image-family=debian-12 \
  --ssh-key ~/.ssh/id_rsa.pub \
  --platform-id standard-v3
```

## Создание Worker узлов

Создайте два worker узла:

```bash
SG_ID_IN=$(yc vpc security-group get kubernetes-the-hard-way-allow-internal --format json | jq '.id' -r)

for i in {0..1}; do
    yc compute instance create \
      --hostname node-$i \
      --name node-$i \
      --zone ru-central1-a \
      --network-interface ipv4-address=10.240.0.2$i,subnet-name=kubernetes,security-group-ids=$SG_ID_IN \
      --memory 4 \
      --cores 2 \
      --core-fraction 20 \
      --create-boot-disk size=50,type=network-ssd,image-folder-id=standard-images,image-family=debian-12 \
      --ssh-key ~/.ssh/id_rsa.pub \
      --platform-id standard-v3
done
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
| fhm****************d | server | ru-central1-a | RUNNING | 51.250.**.*** | 10.240.0.10 |
| fhm****************g | node-0 | ru-central1-a | RUNNING |               | 10.240.0.20 |
| fhm****************j | node-1 | ru-central1-a | RUNNING |               | 10.240.0.21 |
+----------------------+--------+---------------+---------+---------------+-------------+
```

## База данных машин

Этот туториал использует текстовый файл в качестве базы данных машин для хранения различных атрибутов машин, которые
будут использоваться при настройке control plane и worker узлов Kubernetes. Следующая схема представляет записи в базе
данных машин, по одной записи на строку:

```text
IPV4_ADDRESS FQDN HOSTNAME POD_SUBNET
```

Каждый из столбцов соответствует IP-адресу машины `IPV4_ADDRESS`, полному доменному имени `FQDN`, имени хоста `HOSTNAME`
и IP-подсети `POD_SUBNET`. Kubernetes назначает один IP-адрес на `pod`, и `POD_SUBNET` представляет уникальный диапазон
IP-адресов, назначенный каждой машине в кластере для этой цели.

Создайте файл `machines.txt` с внутренними IP-адресами ваших машин на `jumpbox`:

```bash
{
# Получите внутренние IP-адреса машин
SERVER_IP=$(yc compute instance get server --format json | jq -r '.network_interfaces[0].primary_v4_address.address')
NODE0_IP=$(yc compute instance get node-0 --format json | jq -r '.network_interfaces[0].primary_v4_address.address')
NODE1_IP=$(yc compute instance get node-1 --format json | jq -r '.network_interfaces[0].primary_v4_address.address')

# Создайте файл machines.txt
cat > machines.txt << EOF
${SERVER_IP} server.kubernetes.local server
${NODE0_IP} node-0.kubernetes.local node-0 10.200.0.0/24
${NODE1_IP} node-1.kubernetes.local node-1 10.200.1.0/24
EOF
}
```

Проверьте содержимое файла:

```bash
cat machines.txt
```

## Настройка SSH доступа

SSH будет использоваться для настройки машин в кластере. Убедитесь, что у вас есть SSH доступ с правами `root` к каждой
машине, указанной в вашей базе данных машин.

### Включение SSH доступа для root

Если SSH доступ для `root` уже включен на каждой из ваших машин, вы можете пропустить этот раздел.

По умолчанию новая установка `debian` отключает SSH доступ для пользователя `root`. Это делается из соображений безопасности, поскольку пользователь `root` имеет полный административный контроль над Unix-подобными системами. 

Настройте SSH доступ для root на всех машинах из файла `machines.txt`:

```bash
while read IP FQDN HOST SUBNET; do
  echo "Настройка SSH доступа для root на ${HOST} (${IP})..."
  ssh -n yc-user@${IP} -o StrictHostKeyChecking=accept-new "sudo sed -i 's/^#*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config"
  ssh -n yc-user@${IP} "sudo cp /home/yc-user/.ssh/authorized_keys /root/.ssh/authorized_keys"
  ssh -n yc-user@${IP} "sudo chmod 600 /root/.ssh/authorized_keys"
  ssh -n yc-user@${IP} "sudo chown root:root /root/.ssh/authorized_keys"
  ssh -n yc-user@${IP} "sudo systemctl restart sshd"
  echo "SSH доступ для root настроен на ${HOST}"
done < machines.txt
```

### Генерация и распространение SSH ключей

В этом разделе вы сгенерируете и распространите SSH ключевую пару на машины `server`, `node-0` и `node-1`, которая будет
использоваться для выполнения команд на этих машинах в течение всего туториала. Выполните следующие команды с машины
`jumpbox`.

Сгенерируйте новый SSH ключ:

```bash
ssh-keygen
```

```text
Generating public/private rsa key pair.
Enter file in which to save the key (/home/yc-user/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/yc-user/.ssh/id_rsa
Your public key has been saved in /home/yc-user/.ssh/id_rsa.pub
```

Скопируйте SSH публичный ключ на каждую машину:

```bash
while read IP FQDN HOST SUBNET; do
  ssh-copy-id root@${IP}
done < machines.txt
```

После добавления каждого ключа, проверьте что SSH доступ по публичному ключу работает:

```bash
while read IP FQDN HOST SUBNET; do
  ssh -n root@${IP} hostname
done < machines.txt
```

```text
server
node-0
node-1
```

## Настройка имен хостов

В этом разделе вы назначите имена хостов машинам `server`, `node-0` и `node-1`. Имя хоста будет использоваться при
выполнении команд с `jumpbox` на каждую машину. Имя хоста также играет важную роль в кластере. Вместо использования
IP-адресов клиенты Kubernetes будут использовать имя хоста `server` для отправки команд на API сервер Kubernetes. Имена
хостов также используются каждой worker машиной, `node-0` и `node-1`, при регистрации в кластере Kubernetes.

Установите имя хоста на каждой машине, указанной в файле `machines.txt`:

```bash
while read IP FQDN HOST SUBNET; do
    CMD="sed -i 's/^127.0.1.1.*/127.0.1.1\t${FQDN} ${HOST}/' /etc/hosts"
    ssh -n root@${IP} "$CMD"
    ssh -n root@${IP} hostnamectl set-hostname ${HOST}
    ssh -n root@${IP} systemctl restart systemd-hostnamed
done < machines.txt
```

Проверьте что имя хоста установлено на каждой машине:

```bash
while read IP FQDN HOST SUBNET; do
  ssh -n root@${IP} hostname --fqdn
done < machines.txt
```

```text
server.kubernetes.local
node-0.kubernetes.local
node-1.kubernetes.local
```

## Редактирование файла `/etc/hosts`

В этом разделе вы сгенерируете файл `hosts`, который будет добавлен к файлу `/etc/hosts` на `jumpbox` и к файлам
`/etc/hosts` на всех трех участниках кластера, используемых в этом туториале. Это позволит каждой машине быть доступной
по имени хосту, такому как `server`, `node-0` или `node-1`.

Создайте новый файл `hosts` и добавьте заголовок для идентификации добавляемых машин:

```bash
echo "" > hosts
echo "# Kubernetes The Hard Way" >> hosts
```

Сгенерируйте запись хоста для каждой машины в файле `machines.txt` и добавьте её в файл `hosts`:

```bash
while read IP FQDN HOST SUBNET; do
    ENTRY="${IP} ${FQDN} ${HOST}"
    echo $ENTRY >> hosts
done < machines.txt
```

Просмотрите записи хостов в файле `hosts`:

```bash
cat hosts
```

```text

# Kubernetes The Hard Way
10.240.0.10 server.kubernetes.local server
10.240.0.20 node-0.kubernetes.local node-0
10.240.0.21 node-1.kubernetes.local node-1
```

## Добавление записей `/etc/hosts` на Jumpbox

В этом разделе вы добавите DNS записи из файла `hosts` в локальный файл `/etc/hosts` на вашей машине `jumpbox`.

Добавьте DNS записи из `hosts` в `/etc/hosts`:

```bash
sudo sh -c 'cat hosts >> /etc/hosts'
```

Проверьте что файл `/etc/hosts` был обновлен:

```bash
cat /etc/hosts
```

```text
127.0.0.1       localhost
127.0.1.1       jumpbox

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

# Kubernetes The Hard Way
10.240.0.10 server.kubernetes.local server
10.240.0.20 node-0.kubernetes.local node-0
10.240.0.21 node-1.kubernetes.local node-1
```

Теперь вы должны иметь возможность подключаться по SSH к каждой машине, указанной в файле `machines.txt`, используя имя
хоста:

```bash
for host in server node-0 node-1
   do ssh -o StrictHostKeyChecking=accept-new root@${host} hostname
done
```

```text
Warning: Permanently added 'server' (ED25519) to the list of known hosts.
server
Warning: Permanently added 'node-0' (ED25519) to the list of known hosts.
node-0
Warning: Permanently added 'node-1' (ED25519) to the list of known hosts.
node-1
```

## Добавление записей `/etc/hosts` на удаленные машины

В этом разделе вы добавите записи хостов из файла `hosts` в `/etc/hosts` на каждой машине, указанной в текстовом файле
`machines.txt`.

Скопируйте файл `hosts` на каждую машину и добавьте содержимое в `/etc/hosts`:

```bash
while read IP FQDN HOST SUBNET; do
  scp hosts root@${HOST}:~/
  ssh -n \
    root@${HOST} "cat hosts >> /etc/hosts"
done < machines.txt
```

Теперь имена хостов могут использоваться при подключении к машинам с вашей машины `jumpbox` или любой из трех машин в
кластере Kubernetes. Вместо использования IP-адресов вы теперь можете подключаться к машинам, используя имена хостов,
такие как `server`, `node-0` или `node-1`.

Далее: [Создание центра сертификации и генерация TLS сертификатов](05-certificate-authority.md)
