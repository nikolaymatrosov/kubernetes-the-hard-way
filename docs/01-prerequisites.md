# Нам понадобится

## Yandex Cloud

Изначально туториал был написан для Google Cloud Platform](https://cloud.google.com/), но я хочу переложить его на 
Yandex Cloud, где мы будем разворачивать инфраструктуру для кластера Kubernetes с нуля.

Ожидаемая стоимость запущенного кластера: примерно 40-60₽ в час (960-1440₽ в день) 

## Системные требования

### Операционная система
- **Debian 12 (Bookworm)** - рекомендуется для лучшей совместимости с современными компонентами
- **Ubuntu 22.04 LTS** - альтернативный вариант

### Аппаратные требования
- **Jumpbox**: 2 vCPU, 2GB RAM, 20GB disk
- **Control Plane**: 2 vCPU, 4GB RAM, 50GB disk
- **Worker Nodes**: 2 vCPU, 4GB RAM, 50GB disk каждый

## Yandex Cloud CLI

### Установите Yandex Cloud CLI

По этой [инструкции](https://cloud.yandex.ru/docs/cli/quickstart) вы сможете установить и настроить `yc` CLI. 

Убедитесь, что установленная версия 0.120.0 или выше:
```bash
yc --version
```

## Установите jq
Он пригодится нам для разбора вывода `yc` CLI.

### MacOS X
```bash
brew install jq
```

### Linux
```bash
apt install jq
```

## Новые компоненты (Kubernetes v1.32+)

### crictl
Утилита для взаимодействия с CRI-совместимыми контейнерными рантаймами:

```bash
# Linux
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.32.0/crictl-v1.32.0-linux-amd64.tar.gz
tar -xzf crictl-v1.32.0-linux-amd64.tar.gz
sudo mv crictl /usr/local/bin/

# macOS
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.32.0/crictl-v1.32.0-darwin-amd64.tar.gz
tar -xzf crictl-v1.32.0-darwin-amd64.tar.gz
sudo mv crictl /usr/local/bin/
```

### runc
Контейнерный рантайм для запуска контейнеров:

```bash
# Linux
wget https://github.com/opencontainers/runc/releases/download/v1.3.0-rc.1/runc.amd64
sudo mv runc.amd64 /usr/local/bin/runc
sudo chmod +x /usr/local/bin/runc

# macOS
wget https://github.com/opencontainers/runc/releases/download/v1.3.0-rc.1/runc.darwin
sudo mv runc.darwin /usr/local/bin/runc
sudo chmod +x /usr/local/bin/runc
```

### containerd v2
Новая версия containerd с улучшенной производительностью и безопасностью:

```bash
# Linux
wget https://github.com/containerd/containerd/releases/download/v2.1.0/containerd-2.1.0-linux-amd64.tar.gz
tar -xzf containerd-2.1.0-linux-amd64.tar.gz
sudo mv bin/* /usr/local/bin/
```

## Параллельное выполнение команд в tmux

[tmux](https://github.com/tmux/tmux/wiki) можно использовать для одновременного запуска команд на виртуальных машинах. 
Лабораторные работы в этом руководстве могут потребовать запуска одних и тех же команд на нескольких ВМ, в таких 
удобно использовать tmux и разделения окна на несколько панелей с включенной опцией синхронизации, чтобы 
ускорить процесс.

> Использование tmux остается на ваше усмотрение.

![tmux screenshot](images/tmux-screenshot.png)

> Чтобы включить синхронизацию, нажмите `ctrl+b`, а затем `shift+:`. Потом наберите `set synchronize-panes on`. Чтобы 
> отключить: `set synchronize-panes off`.

Дальше: [Установка jumpbox](02-prepare-jumpbox.md)
