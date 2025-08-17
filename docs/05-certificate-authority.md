# Создание CA и генерация TLS сертификатов

На этом шаге мы создадим [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure), используя openssl для настройки Certificate Authority и генерации TLS сертификатов для следующих компонентов: kube-apiserver, kube-controller-manager, kube-scheduler, kubelet и kube-proxy. Команды в этом разделе должны выполняться на `jumpbox`.

## Certificate Authority

В этом разделе мы создадим Certificate Authority, который может быть использован для генерации дополнительных TLS сертификатов для других компонентов Kubernetes. Настройка CA и генерация сертификатов с помощью `openssl` может занять много времени, особенно если делаете это впервые. Для упрощения этого руководства я включил файл конфигурации openssl `ca.conf`, который определяет все детали, необходимые для генерации сертификатов для каждого компонента Kubernetes.

Потратьте время на изучение файла конфигурации `ca.conf`:

```bash
cat ca.conf
```

Вам не нужно понимать все в файле `ca.conf` для завершения этого руководства, но вы должны рассматривать его как отправную точку для изучения `openssl` и конфигурации, которая входит в управление сертификатами на высоком уровне.

Каждый certificate authority начинается с приватного ключа и корневого сертификата. В этом разделе мы создадим самоподписанный certificate authority, и хотя этого достаточно для данного руководства, это не следует считать тем, что вы бы делали в реальной производственной среде.

Сгенерируйте файл конфигурации CA, сертификат и приватный ключ:

```bash
{
  openssl genrsa -out ca.key 4096
  openssl req -x509 -new -sha512 -noenc \
    -key ca.key -days 3653 \
    -config ca.conf \
    -out ca.crt
}
```
Проверим результат выполнения вышеуказанной команды:

```bash
ls -1 ca.crt ca.key
```

Результат:

```txt
ca.crt
ca.key
```

## Создание клиентских и серверных сертификатов

В этом разделе вы сгенерируете клиентские и серверные сертификаты для каждого компонента Kubernetes и клиентский сертификат для пользователя `admin` в Kubernetes.

Сгенерируйте сертификаты и приватные ключи:

```bash
certs=(
  "admin" "node-0" "node-1"
  "kube-proxy" "kube-scheduler"
  "kube-controller-manager"
  "kube-api-server"
  "service-accounts"
)

for i in ${certs[*]}; do
  openssl genrsa -out "${i}.key" 4096

  openssl req -new -key "${i}.key" -sha256 \
    -config "ca.conf" -section ${i} \
    -out "${i}.csr"

  openssl x509 -req -days 3653 -in "${i}.csr" \
    -copy_extensions copyall \
    -sha256 -CA "ca.crt" \
    -CAkey "ca.key" \
    -CAcreateserial \
    -out "${i}.crt"
done
```

Результат выполнения вышеуказанной команды сгенерирует приватный ключ, запрос на сертификат и подписанный SSL сертификат для каждого из компонентов Kubernetes. Вы можете перечислить сгенерированные файлы следующей командой:

```bash
ls -1 *.crt *.key *.csr
```
Результат:
```text
admin.crt
admin.csr
admin.key
ca.crt
ca.key
kube-api-server.crt
kube-api-server.csr
kube-api-server.key
kube-controller-manager.crt
kube-controller-manager.csr
kube-controller-manager.key
kube-proxy.crt
kube-proxy.csr
kube-proxy.key
kube-scheduler.crt
kube-scheduler.csr
kube-scheduler.key
node-0.crt
node-0.csr
node-0.key
node-1.crt
node-1.csr
node-1.key
service-accounts.crt
service-accounts.csr
service-accounts.key
```


## Распределение клиентских и серверных сертификатов

В этом разделе вы скопируете различные сертификаты на каждую машину по пути, где каждый компонент Kubernetes будет искать свою пару сертификатов. В реальной среде эти сертификаты должны рассматриваться как набор чувствительных секретов, поскольку они используются компонентами Kubernetes как учетные данные для аутентификации друг с другом.

Скопируйте соответствующие сертификаты и приватные ключи на машины `node-0` и `node-1`:

```bash
for host in node-0 node-1; do
  ssh root@${host} mkdir -p /var/lib/kubelet/

  scp ca.crt root@${host}:/var/lib/kubelet/

  scp ${host}.crt \
    root@${host}:/var/lib/kubelet/kubelet.crt

  scp ${host}.key \
    root@${host}:/var/lib/kubelet/kubelet.key
done
```

Скопируйте соответствующие сертификаты и приватные ключи на машину `server`:

```bash
scp \
  ca.key ca.crt \
  kube-api-server.key kube-api-server.crt \
  service-accounts.key service-accounts.crt \
  root@server:~/
```

> Клиентские сертификаты `kube-proxy`, `kube-controller-manager`, `kube-scheduler` и `kubelet` будут использоваться для генерации файлов конфигурации клиентской аутентификации в следующем разделе.

Дальше: [Генерация файлов конфигурации Kubernetes для аутентификации](06-kubernetes-configuration-files.md)

