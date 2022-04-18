# Развертывание Kubernetes Control Plane

В этой лабораторной работе вы развернете Kubernetes Control Plane на трех инстансах и настроите ее
для обеспечения высокой доступности. Вы также создадите внешний балансировщик нагрузки, который даст удаленным клиентам
доступ к API Kubernetes.
На каждом узле будут установлены следующие компоненты: сервер API Kubernetes, Scheduler и Controller Manager.

## Важное

Команды из этой лабораторной должный выполняться на каждом инстансе контроллера: `controller-0`, `controller-1`
и `controller-2`. Зайдите на каждый инстанс.

### Параллельное выполнение команд через tmux

[tmux](https://github.com/tmux/tmux/wiki) может быть использован для параллельного выполнения команд на нескольких
инстансах. Смотри [Параллельное выполнение команд в tmux](01-prerequisites.md)

## Развертывание Kubernetes Control Plane

Создайте директорию для конфигов Kubernetes:

```bash
sudo mkdir -p /etc/kubernetes/config
```

### Скачайте и установите Kubernetes Controller

Скачайте официальный релиз Kubernetes:

```bash
K8S_VERSION=v1.21.0
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/linux/amd64/kubectl"
```

Установите Kubernetes:

```bash
{
  chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
  sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
}
```

### Настройте сервер Kubernetes API

```bash
{
  sudo mkdir -p /var/lib/kubernetes/

  sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem \
    encryption-config.yaml /var/lib/kubernetes/
}
```

Внутренний IP-адрес будет использован другими членима кластера для общения с API сервером. Получить внутренний адрес
можно следующим образом:

```bash
INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)
```

Так как мы не настраивали yc на инстансах, а то следующий код нужно выполнить локально, а затем на серверах установить
переменную окружения.

```bash
yc vpc address get kubernetes-the-hard-way --format json | jq '.external_ipv4_address.address' -r
```

```bash
KUBERNETES_PUBLIC_ADDRESS=
```

Создайте `kube-apiserver.service` файл описания юнита в systemd:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --runtime-config='api/all=true' \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-account-issuer=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Сконфигурируйте Kubernetes Controller Manager

Переместите `kube-controller-manager` kubeconfig на место:

```bash
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

Создайте `kube-controller-manager.service` файл описания юнита в systemd:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Сконфигурируйте Kubernetes Scheduler

Переместите `kube-scheduler` kubeconfig на место:

```bash
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

Создайте файл конфигурации `kube-scheduler.yaml`:

```bash
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

Создайте `kube-scheduler.service` файл описания юнита в systemd:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Запустите Controller Services

```bash
{
  sudo systemctl daemon-reload
  sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
}
```

> У серверов Kubernetes API может уйти до 10 секунд на то чтобы стартовать полностью.

### Включите проверки состояния по HTTP

[Network Load Balancer](https://cloud.yandex.ru/docs/network-load-balancer/concepts/) который будет использоваться для
распределения трафика между тремя серверами API позволит каждому серверу API терминировать TLS и проверять
сертификаты клиентов. Балансировщик сетевой нагрузки поддерживает только проверки работоспособности по HTTP, что
означает, что HTTPS эндпоинт API сервера, не может быть использован.
В качестве обходного пути веб-сервер nginx можно использовать для проверки работоспособности прокси-сервера HTTP. В этом
разделе будет установлен и настроен nginx для приема проверок работоспособности HTTP на порту `80` и прокси-соединения с
сервером API на `https://127.0.0.1:6443/healthz`.

> Эндпоинт `/healthz` API сервера по умолчанию не требует аутентификации.

Установи базовый веб-сервер чтобы обрабатывать HTTP проверки состояния:

```bash
sudo apt-get update
sudo apt-get install -y nginx
```

```bash
cat > kubernetes.default.svc.cluster.local <<EOF
server {
  listen      80;
  listen      [::]:80;
  server_name _;

  location /healthz {
     proxy_pass                    https://127.0.0.1:6443/healthz;
     proxy_ssl_trusted_certificate /var/lib/kubernetes/ca.pem;
     add_header Host kubernetes.default.svc.cluster.local;
  }
}
EOF
```

```bash
{
  sudo rm /etc/nginx/sites-enabled/default
  sudo mv kubernetes.default.svc.cluster.local \
    /etc/nginx/sites-available/kubernetes.default.svc.cluster.local

  sudo ln -s /etc/nginx/sites-available/kubernetes.default.svc.cluster.local /etc/nginx/sites-enabled/
}
```

```bash
sudo systemctl restart nginx
```

```bash
sudo systemctl enable nginx
```

### Проверка

```bash
kubectl cluster-info --kubeconfig admin.kubeconfig
```

> output

```
Kubernetes control plane is running at https://127.0.0.1:6443
```

Протестируйте nginx HTTP прокси для проверок:

```bash
curl  -i http://127.0.0.1/healthz
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Sun, 02 May 2021 04:19:29 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 2
Connection: keep-alive
Cache-Control: no-cache, private
X-Content-Type-Options: nosniff
X-Kubernetes-Pf-Flowschema-Uid: c43f32eb-e038-457f-9474-571d43e5c325
X-Kubernetes-Pf-Prioritylevel-Uid: 8ba5908f-5569-4330-80fd-c643e7512366

ok
```

> Не забудьте все команды выше нужно выполнять на всех трех инстансах контроллера: `controller-0`, `controller-1`
> и `controller-2`

## RBAC для авторизации Kubelet

В этой секции вы сконфигурируете RBAC пермишены для того, чтобы разрешить API серверу Kubernetes получать доступ к
Kublet API на каждом воркере. Доступ к Kubelet API нужен для получения метрик, логов и исполнения команд в подах.

> В этом туториале мы устанавливаем флаг Kubelet `--authorization-mode` в занчение `Webhook`. В этом режиме используется
> [SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access) для проверки авторизации в
> API

Команды в этой секции применятся к кластеру целиком, поэтому их нужно запускать только с одной из машин контроллера.

Создайте [ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole) `system:kube-apiserver-to-kubelet`
с правами на доступ к Kubelet API и выполнения самых общих задач связанных с управлением подами:

```bash
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

Kubernetes API сервер аутентифицируется в кублете как пользователь `kubernetes`, используя клиентский сертификат,
который задается при помощи флага `--kubelet-client-certificate`.

Привяжите `system:kube-apiserver-to-kubelet` ClusterRole к пользователю `kubernetes`:

```bash
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

## Балансировщик для Kubernetes Frontend

В этой секции вы создадите внешний балансировщик для Kubernetes API.
Статический IP-адрес `kubernetes-the-hard-way` будет прикреплен к нему.

> Следующие команды нужно выполнить с машины, где установлен YC CLI.

### Создание Network Load Balancer

```bash
{
  KUBERNETES_PUBLIC_ADDRESS=$(yc vpc address get kubernetes-the-hard-way --format json | jq '.external_ipv4_address.address' -r)
  
  yc load-balancer target-group create --name=kubernetes
  TARGET_GROUP_ID=$(yc load-balancer target-group get --name=kubernetes --format json | jq '.id' -r)
  
  for i in 0 1 2; do
    yc load-balancer target-group add --name=kubernetes --target subnet-name=kubernetes,address=10.240.0.1${i}
  done
  
  yc load-balancer network-load-balancer create \
  --region-id ru-central1 \
  --name kubernetes-lb \
  --listener name=test-listener,external-ip-version=ipv4,port=6443,target-port=6443,external-address=$KUBERNETES_PUBLIC_ADDRESS \
  --target-group target-group-id=${TARGET_GROUP_ID},healthcheck-name=health,healthcheck-http-port=80,healthcheck-http-path=/healthz
  
}
```

### Проверка

Получите IP-адресс `kubernetes-the-hard-way` :

```bash
KUBERNETES_PUBLIC_ADDRESS=$(yc vpc address get kubernetes-the-hard-way --format json | jq '.external_ipv4_address.address' -r)
```

Сделайте HTTP запрос к Kubernetes, чтобы узнать его версию:

```bash
curl --cacert ca.pem "https://${KUBERNETES_PUBLIC_ADDRESS}:6443/version"
```

> output

```
{
  "major": "1",
  "minor": "21",
  "gitVersion": "v1.21.0",
  "gitCommit": "cb303e613a121a29364f75cc67d3d580833a7479",
  "gitTreeState": "clean",
  "buildDate": "2021-04-08T16:25:06Z",
  "goVersion": "go1.16.1",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

Дальше: [Развертывание воркеров Kubernetes](09-bootstrapping-kubernetes-workers.md)
