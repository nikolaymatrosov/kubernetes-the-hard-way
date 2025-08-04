# Генерируем Data Encryption конфиг и ключ

Kubernetes хранит самые разные данные, в том числе состояние кластера, конфигурации приложений и секреты. Kubernetes
поддерживает возможность [зашифровать](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data) такие данные.

В этой лабораторной вы
сгенерируете [конфиг](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration)
подходящий для шифрования Kubernetes Secrets.

## Ключ шифрования

Сгенерируйте ключ шифрования.

```bash
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## Файл конфигурации шифрования

Создайте `encryption-config.yaml`.

```bash
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```
Скопируйте `encryption-config.yaml` на server:

```bash
# Получите внешний IP server
SERVER_EXTERNAL_IP=$(yc compute instance get server --format json | jq '.network_interfaces[0].primary_v4_address.one_to_one_nat.address' -r)

# Скопируйте файл на server
scp encryption-config.yaml yc-user@$SERVER_EXTERNAL_IP:~/
```

Дальше: [Развертывание кластера etcd](08-bootstrapping-etcd.md)
