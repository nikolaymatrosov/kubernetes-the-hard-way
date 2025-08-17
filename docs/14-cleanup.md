# Отчистка
В этой лабораторной вы удалите все созданные ресурсы, созданные в этом туториале.

## Виртуальный машины

Удалите все виртуальные машины:

```bash
for instance in jumpbox server node-0 node-1; do
  yc compute instance delete ${instance} --async;
done
```

## Сеть

Удалите балансер и сетевые ресурсы:

```bash
{
  yc vpc security-group delete kubernetes-the-hard-way-allow-external
  yc vpc security-group delete kubernetes-the-hard-way-allow-internal
  yc vpc security-group delete kubernetes-the-hard-way-allow-balancer

  yc load-balancer target-group delete kubernetes
  yc load-balancer network-load-balancer delete kubernetes-lb
  
  yc vpc address delete kubernetes-the-hard-way 
  yc dns zone delete kubernetes
}
```

Удалите подсети и сеть `kubernetes-the-hard-way`:

```bash
{
  yc vpc subnet delete kubernetes
  yc vpc route-table delete kubernetes-route
  yc vpc network delete kubernetes-the-hard-way
}
```

## Дополнительная очистка (опционально)

### Удаление Service Account

Если вы создавали Service Account для Cloud Controller Manager:

```bash
# Удалите Service Account
yc iam service-account delete --name k8s-controller-manager

# Удалите ключ (если сохраняли локально)
rm -f key.json
```

### Очистка локальных файлов

```bash
# Удалите локальные файлы конфигурации
rm -rf ~/kubernetes-the-hard-way/
rm -f ~/.kube/config
```

## Проверка очистки

Убедитесь, что все ресурсы удалены:

```bash
# Проверьте виртуальные машины
yc compute instance list

# Проверьте сети
yc vpc network list

# Проверьте DNS зоны
yc dns zone list

# Проверьте статические IP
yc vpc address list
```

Все команды должны вернуть пустые списки или сообщения об отсутствии ресурсов.
