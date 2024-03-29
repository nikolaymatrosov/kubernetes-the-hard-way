# Нам понадобится

## Yandex Cloud

Изначально туториал был написан для Google Cloud Platform](https://cloud.google.com/), но я хочу переложить его на 
Yandex Cloud, где мы будем разворачивать инфраструктуру для кластера Kubernetes с нуля.

Ожидаемая стоимость запущенного кластера: примерно 50₽ в час (1200₽ в день) 

## Yandex Cloud CLI

### Установите Yandex Cloud CLI

По этой [инструкции](https://cloud.yandex.ru/docs/cli/quickstart) вы сможете установить и настроить `yc` CLI. 

Убедитесь, что установленная версия 0.88.0 или выше:
```bash
yc --version
```

## Установите jq
Он пригодится нам для разбора вывода `yc` CLI.

### MacOS X
```bash
brew instal jq
```

### Linux
```bash
apt install jq
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

Дальше: [Установка инструментов](02-client-tools.md)
