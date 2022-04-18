# Отчистка
В этой лабораторной вы удалите все созданные ресурсы, созданные в этом туториале.

## Виртуальный машины

Удалите все виртуальные машины:

```bash
for instance in  controller-0 controller-1 controller-2 worker-0 worker-1 worker-2; do
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
  yc vpc route-table delete kubernetes-route
}
```


Удалите подсети и сеть `kubernetes-the-hard-way`:

```bash
{
  yc vpc subnet delete kubernetes
  yc vpc network delete kubernetes-the-hard-way
}  
```
