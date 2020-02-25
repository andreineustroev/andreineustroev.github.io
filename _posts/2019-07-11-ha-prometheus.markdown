---
layout: post
title:  "HA prometheus"
date:   2020-02-22 22:15:20 +0500
categories: monitoring, prometheus, ha
---

Возникла необходимость сделать отказоустойчивый prometheus, благо что он такое умеет, хоть и немного странно. Но нагуглить какой-то гайд мне сходу не удалось, поэтому пишу свой.

# Вводные
Используем 2 сервера, на каждом только docker + docker-compose без swarm. Большинство гайдов как раз про оркестрацию, мол запускаем несколько prometheus и alertmanager в service-mesh в единственном экземпляре и оно работает, alertmanager же stateless.

Но меня такое не устроило, поэтому приступим.

# Настройка alertmanager

Чтобы переключить alertmanager в кластерный режим нужно указать параметр запуска `--cluster.listen-address=` с любым адресом и портом, например `0.0.0.0:9094`. Параметр `--cluster.advertise-address` так и не понял на что влияет, без него само находится. Так же важно перечислить всех пиров в параметре `--cluster.peer` по параметру на пира.

Естественно 9094 порт нужно открыть, пишут что там tcp и udp но я увидел только tcp.

# Настройка promtheus

Чтобы добавить таких alertmanager ов, нужно в конфиге прометея указать их всех. Они сами между собой договорятся, кто посылает алерты.

```yaml
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - alertmanager1:9093
      - alertmanager2:9093
      - alertmanager3:9093
```
# Iptables

Поскольку у нас не swarm  и сервисы взаимодействуют друг с другом через интернет, нужно ограничить трафик. 
В ходе настройки стало ясно, что сервисы взаимодействуют и сами с собой поэтому нужно изменить имя моста создаваемого docker-compose. Для этого изменим секцию networks:
```yaml
networks:
  default:
    driver: bridge
    driver_opts:
      com.docker.network.enable_ipv6: "false"
      com.docker.network.bridge.name: "monitoring" # need for iptables
 ```
 Создадим цепочку
 ```bash
 iptables -N monitoring
 ```
 Направим весь трафик в неё
 ```bash
 iptables -I DOCKER-USER -j MONITORING
 ```
 И запоним её правилами
 ```bash
iptables -A MONITORING -i monitoring -j ACCEPT # разрешаем подключения с моста
iptables -A MONITORING -s 192.168.30.12/32 -p tcp -m tcp --dport 9094 -j ACCEPT #Разрешаем подключения со второго сервера
iptables -A MONITORING -p tcp -m tcp --dport 9094 -j DROP # Всех остальных дропаем
```
