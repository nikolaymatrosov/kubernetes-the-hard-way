# Отчистка
В этой лабораторной вы удалите все созданные ресурсы, созданные в этом туториале.

## Виртуальный машины

Удалите все виртуальные машины:

```bash
for instance in jumpbox server node-0 node-1; do
  yc compute instance delete ${instance} --async;
done
```

Дождитесь завершения удаления. Иначе вы не сможете удалить сетевые ресурсы.

## Сеть

Удалите сетевые ресурсы:

```bash
{
  yc vpc security-group delete kubernetes-the-hard-way-allow-external
  yc vpc security-group delete kubernetes-the-hard-way-allow-internal
  yc vpc subnet update --name kubernetes --disassociate-route-table
      yc vpc route-table delete kubernetes-the-hard-way-nat-route-table

    yc vpc gateway delete kubernetes-the-hard-way-nat-gateway
}
```

Удалите подсети и сеть `kubernetes-the-hard-way`:

```bash
{
  yc vpc subnet delete kubernetes
  yc vpc network delete kubernetes-the-hard-way
}
```

## Дополнительная очистка (опционально)

### Удаление Service Account

Если вы создавали Service Account для Cloud Controller Manager:

```bash
# Удалите Service Account
yc iam service-account delete --name kubernetes-the-hard-way
```

### Очистка локальных файлов

```bash
# Удалите локальные файлы конфигурации
rm -f ~/.kube/config
```

## Проверка очистки

Убедитесь, что все ресурсы удалены:

```bash
# Проверьте виртуальные машины
yc compute instance list

# Проверьте сети
yc vpc network list

# Проверьте статические IP
yc vpc address list
```

Все команды должны вернуть пустые списки или сообщения об отсутствии ресурсов.
