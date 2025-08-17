# Смоук тест

В этой лабораторной работе вы выполните ряд заданий, чтобы убедиться, что кластер Kubernetes работает корректно.

## Шифрование данных

В этом разделе вы проверите
возможность [шифрование секретных данных в покое](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted).

Создайте обычный секрет:

```bash
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

Выведите hexdump секрета `kubernetes-the-hard-way`, сохраненного в etcd:

```bash
ssh root@server 'apt update && apt install -y bsdextrautils'
ssh root@server \
    'etcdctl get /registry/secrets/default/kubernetes-the-hard-way | hexdump -C'
```

```text
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a 3d b4 d7 00 59 53 87  |:v1:key1:=...YS.|
00000050  ab 7c fd f5 14 9f 90 20  49 0d 26 c8 ab 95 ec 5a  |.|..... I.&....Z|
00000060  f7 91 b2 c8 bf e9 4c 41  c9 af b2 e1 16 7f 42 5d  |......LA......B]|
00000070  89 0d 3f 05 09 0c 38 a0  d2 b0 3c 6d 91 2f 88 fe  |..?...8...<m./..|
00000080  0e 85 d9 be 77 fc 14 96  9a e8 6d a1 74 c0 e6 c6  |....w.....m.t...|
00000090  54 ba 46 b5 25 ff ed c4  0e 71 a7 7a 4e 64 7c c9  |T.F.%....q.zNd|.|
000000a0  22 ef ae 94 e4 d9 9d b0  1b 5a 97 fc 17 80 71 ad  |"........Z....q.|
000000b0  eb cd 41 25 f3 39 b9 4c  d7 75 cf 2a f0 f0 09 cc  |..A%.9.L.u.*....|
000000c0  c4 9a 3d fe 5c 30 69 1e  15 99 5f e8 84 55 de 78  |..=.\0i..._..U.x|
000000d0  f8 06 18 1f a5 cb 0a 2b  08 d2 17 f7 03 a2 41 56  |.......+......AV|
000000e0  ea 73 ec 4a bc a4 b3 a7  0b 30 88 c8 dc 1a 85 f9  |.s.J.....0......|
000000f0  8f 11 e5 83 5b 3c 6e 15  0a 1f 3a fc bf 3d ff e6  |....[<n...:..=..|
00000100  68 2a 86 cf c4 79 3f 41  f6 13 1b 55 a8 f4 94 46  |h*...y?A...U...F|
00000110  5b 61 7d 61 8c 65 23 0f  2d 01 2e c5 4b c5 d4 03  |[a}a.e#.-...K...|
00000120  6f 07 b2 96 ad 79 8c 9b  50 54 0c fa c0 c5 79 19  |o....y..PT....y.|
00000130  f6 48 11 86 26 59 86 de  e5 f9 62 11 42 07 b2 9e  |.H..&Y....b.B...|
00000140  45 c5 3e e2 69 9b c1 55  7e a1 8c 8d a1 71 91 5e  |E.>.i..U~....q.^|
00000150  09 a0 6b 1d 03 f9 5e 40  3b 0a                    |..k...^@;.|
0000015a
```

Ключ etcd должен начинаться с `k8s:enc:aescbc:v1:key1`, что указывает на то, что провайдер `aescbc` использовался для
шифрования данных с ключом шифрования `key1`.

## Развертывания

В этом разделе вы проверите возможность создавать и
управлять [развертываниями](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Создайте развертывание веб-сервера [nginx](https://nginx.org/en/):

```bash
kubectl create deployment nginx \
  --image=nginx:latest
```

Выведите список подов, созданных развертыванием `nginx`:

```bash
kubectl get pods -l app=nginx
```

```bash
NAME                     READY   STATUS    RESTARTS   AGE
nginx-56fcf95486-c8dnx   1/1     Running   0          8s
```

### Проброс портов

В этом разделе вы проверите возможность получать удаленный доступ к приложениям
используя [проброс портов](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

Получите полное имя пода `nginx`:

```bash
POD_NAME=$(kubectl get pods -l app=nginx \
  -o jsonpath="{.items[0].metadata.name}")
```

Пробросьте порт `8080` на вашей локальной машине в порт `80` пода `nginx`:

```bash
kubectl port-forward $POD_NAME 8080:80
```

```text
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

**В новом терминале** выполните HTTP-запрос используя адрес проброса:

```bash
curl --head http://127.0.0.1:8080
```

```text
HTTP/1.1 200 OK
Server: nginx/1.29.1
Date: Sun, 17 Aug 2025 09:29:32 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Wed, 13 Aug 2025 14:33:41 GMT
Connection: keep-alive
ETag: "689ca245-267"
Accept-Ranges: bytes
```

Переключитесь обратно в предыдущий терминал и остановите проброс портов в под `nginx`:

```text
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

### Логи

В этом разделе вы проверите
возможность [получать логи контейнеров](https://kubernetes.io/docs/concepts/cluster-administration/logging/).

Выведите логи пода `nginx`:

```bash
kubectl logs $POD_NAME
```

```text
...
127.0.0.1 - - [06/Apr/2025:17:17:12 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.88.1" "-"
```

### Выполнение команд

В этом разделе вы проверите
возможность [выполнять команды в контейнере](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container).

Выведите версию nginx, выполнив команду `nginx -v` в контейнере `nginx`:

```bash
kubectl exec -ti $POD_NAME -- nginx -v
```

```text
nginx version: nginx/1.29.1
```

## Сервисы

В этом разделе вы проверите возможность выставлять приложения
используя [сервис](https://kubernetes.io/docs/concepts/services-networking/service/).

Выставите развертывание `nginx` используя
сервис [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport):

```bash
kubectl expose deployment nginx \
  --port 80 --type NodePort
```

> Тип сервиса LoadBalancer не может быть использован, поскольку ваш кластер не настроен
> с [интеграцией облачного провайдера](https://kubernetes.io/docs/getting-started-guides/scratch/#cloud-provider).
> Настройка интеграции облачного провайдера выходит за рамки данного руководства.

Получите порт узла, назначенный сервису `nginx`:

```bash
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

Получите имя хоста узла, на котором работает под `nginx`:

```bash
NODE_NAME=$(kubectl get pods \
  -l app=nginx \
  -o jsonpath="{.items[0].spec.nodeName}")
```

Выполните HTTP-запрос используя IP-адрес и порт узла `nginx`:

```bash
curl -I http://${NODE_NAME}:${NODE_PORT}
```

```text
Server: nginx/1.27.4
Date: Sun, 06 Apr 2025 17:18:36 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Wed, 05 Feb 2025 11:06:32 GMT
Connection: keep-alive
ETag: "67a34638-267"
Accept-Ranges: bytes
```

Далее: [Очистка](14-cleanup.md)
