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
Скопируйте `encryption-config.yaml` на каждый контроллер:

```bash
for instance in controller-0 controller-1 controller-2; do
  EXTERNAL_IP=$(yc compute instance get ${instance} --format json | jq '.network_interfaces[0].primary_v4_address.one_to_one_nat.address' -r)
  for filename in encryption-config.yaml; do
    scp $filename yc-user@$EXTERNAL_IP:~/
  done
done
```

Дальше: [Развертывание кластера etcd](07-bootstrapping-etcd.md)
