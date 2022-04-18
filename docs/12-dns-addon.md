# Разворачиваем DNS Cluster Add-on

В этой лабораторной вы развернете [DNS add-on](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/).
Он обеспечивает обнаружение сервисов, бегущих в контейнерах внутри Kubernetes, через DNS. Основано на [CoreDNS](https://coredns.io/). 

## Аддон

Разверните `coredns`:

```bash
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns-1.8.yaml
```

> output

```
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
```
На хостах выполните следующие 3 команды:

```bash
sudo mkdir /etc/coredns
cat <<EOF | sudo tee /etc/coredns/Corefile
.:53 {
  forward . dns://10.240.0.2
  log
  errors
  cache {
    serve_stale 1h
  }
}
EOF
```

```bash
cat <<EOF | sudo tee /etc/systemd/system/coredns.service
[Unit]
Description=CoreDNS DNS server
Documentation=https://coredns.io
After=network.target

[Service]
PermissionsStartOnly=true
LimitNOFILE=1048576
LimitNPROC=512
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
NoNewPrivileges=true
User=coredns
WorkingDirectory=~
ExecStart=/usr/local/bin/coredns -conf=/etc/coredns/Corefile
ExecReload=/bin/kill -SIGUSR1 $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

```bash
sudo curl -O -L https://github.com/coredns/coredns/releases/download/v1.8.4/coredns_1.8.4_linux_amd64.tgz
sudo tar -zxf coredns_1.8.4_linux_amd64.tgz --directory /usr/local/bin
sudo useradd -d /var/lib/coredns -m coredns
sudo systemctl disable systemd-resolved.service
sudo systemctl stop systemd-resolved
sudo sed -i 's/127.0.0.53/127.0.0.1/' /etc/resolv.conf
sudo systemctl enable coredns
sudo systemctl start coredns
```

Выведите список подов созданных `kube-dns`:

```bash
kubectl get pods -l k8s-app=kube-dns -n kube-system
```

> output

```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-8494f9c688-hh7r2   1/1     Running   0          10s
coredns-8494f9c688-zqrj2   1/1     Running   0          10s
```

## Проверка

Создайте `busybox`:

```bash
kubectl run busybox --image=busybox:1.28 --command -- sleep 3600
```

Выведите список подов созданных при развертывании`busybox`:

```bash
kubectl get pods -l run=busybox
```

> output

```
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          3s
```

Получите полное название пода `busybox`:

```bash
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

Выполните DNS lookup для сервиса `kubernetes` внутри пода `busybox`:

```bash
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

> output

```
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```
Дальше: [Комплексная проверка кластера](13-smoke-test.md)
