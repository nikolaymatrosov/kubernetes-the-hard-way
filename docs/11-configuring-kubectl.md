# Настройка kubectl для удаленного доступа

В этой лабораторной работе вы сгенерируете файл kubeconfig для утилиты командной строки `kubectl` на основе учетных
данных пользователя `admin`.

> Выполняйте команды в этой лабораторной работе на машине `jumpbox`.

## Конфигурационный файл администратора Kubernetes

Каждый kubeconfig требует API-сервер Kubernetes для подключения.

Вы должны быть в состоянии пинговать `server.kubernetes.local` на основе DNS-записи `/etc/hosts` из предыдущей
лабораторной работы.

```bash
curl --cacert ca.crt \
  https://server.kubernetes.local:6443/version
```

```text
{
  "major": "1",
  "minor": "32",
  "gitVersion": "v1.32.3",
  "gitCommit": "32cc146f75aad04beaaa245a7157eb35063a9f99",
  "gitTreeState": "clean",
  "buildDate": "2025-03-11T19:52:21Z",
  "goVersion": "go1.23.6",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

Сгенерируйте файл kubeconfig, подходящий для аутентификации в качестве пользователя `admin`:

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443

  kubectl config set-credentials admin \
    --client-certificate=admin.crt \
    --client-key=admin.key

  kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin

  kubectl config use-context kubernetes-the-hard-way
}
```

Результат выполнения приведенной выше команды должен создать файл kubeconfig в местоположении по умолчанию
`~/.kube/config`, используемом утилитой командной строки `kubectl`. Это также означает, что вы можете запускать команду
`kubectl` без указания конфигурации.

## Проверка

Проверьте версию удаленного кластера Kubernetes:

```bash
kubectl version
```

```text
Client Version: v1.32.3
Kustomize Version: v5.5.0
Server Version: v1.32.3
```

Выведите список узлов в удаленном кластере Kubernetes:

```bash
kubectl get nodes
```

```
NAME     STATUS   ROLES    AGE    VERSION
node-0   Ready    <none>   10m   v1.32.3
node-1   Ready    <none>   10m   v1.32.3
```

Далее: [Предоставление маршрутов сети подов](12-pod-network-routes.md)
