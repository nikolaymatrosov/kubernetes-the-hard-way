# Генерируем Data Encryption конфиг и ключ

Kubernetes хранит самые разные данные, в том числе состояние кластера, конфигурации приложений и секреты. Kubernetes
поддерживает возможность [зашифровать](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data) такие данные.

В этой лабораторной вы сгенерируете ключ шифрования и [конфиг](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration)
подходящий для шифрования Kubernetes Secrets.

## Ключ шифрования

Сгенерируйте ключ шифрования:

```bash
export ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## Файл конфигурации шифрования

Создайте файл конфигурации шифрования `encryption-config.yaml`:

```bash
envsubst < configs/encryption-config.yaml \
  > encryption-config.yaml
```

Скопируйте файл конфигурации шифрования `encryption-config.yaml` на каждый экземпляр контроллера:

```bash
scp encryption-config.yaml root@server:~/
```

Дальше: [Развертывание кластера etcd](08-bootstrapping-etcd.md)
