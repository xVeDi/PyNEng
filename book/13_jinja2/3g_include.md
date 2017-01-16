## include

Выражение include позволяет добавить один шаблон в другой.

В итоге, те переменные, которые мы передаем как данные, должны передавать все нужные данные и для основного шаблона, и для того, который мы добавили.

Посмотрим на пример использования include.

Шаблон templates/vlans.txt:
```
{% for vlan, name in vlans.items() %}
vlan {{ vlan }}
 name {{ name }}
{% endfor %}
```

Шаблон templates/ospf.txt:
```
router ospf 1
 auto-cost reference-bandwidth 10000
{% for networks in ospf %}
 network {{ networks.network }} area {{ networks.area }}
{% endfor %}
```

Шаблон templates/bgp.txt:
```
router bgp {{ bgp.local_as }}
{% for ibgp in bgp.ibgp_neighbors %}
 neighbor {{ ibgp }} remote-as {{ bgp.local_as }}
 neighbor {{ ibgp }} update-source {{ bgp.loopback }}
{% endfor %}
{% for ebgp in bgp.ebgp_neighbors %}
 neighbor {{ ebgp }} remote-as {{ bgp.ebgp_neighbors[ebgp] }}
{% endfor %}
```

Теперь создадим шаблон templates/switch.txt, который будет использовать созданные шаблоны ospf и vlans:
```
{% include 'vlans.txt' %}

{% include 'ospf.txt' %}
```

И создадим файл с данными, для генерации конфигурации (data_files/switch.yml):
```json
vlans:
  10: Marketing
  20: Voice
  30: Management
ospf:
  - network: 10.0.1.0 0.0.0.255
    area: 0
  - network: 10.0.2.0 0.0.0.255
    area: 2
  - network: 10.1.1.0 0.0.0.255
    area: 0
```

Результат генерации конфгурации будет таким (мы по-прежнему используем скрипт cfg_gen.py):
```
$ python cfg_gen.py templates/switch.txt data_files/switch.yml
vlan 10
 name Marketing
vlan 20
 name Voice
vlan 30
 name Management

router ospf 1
 auto-cost reference-bandwidth 10000
 network 10.0.1.0 0.0.0.255 area 0
 network 10.0.2.0 0.0.0.255 area 2
 network 10.1.1.0 0.0.0.255 area 0
```

Итоговая конфигурация получилась такой, как-будто строки из шаблонов ospf.txt и vlans.txt, находились в шаблоне switch.txt.


Теперь сделаем шаблон templates/router.txt:
```
{% include 'ospf.txt' %}

{% include 'bgp.txt' %}

logging {{ log_server }}
```

В данном случае, кроме include, добавлена ещё одна строка в шаблон, чтобы показать, что выражения include могут идти вперемешку с обычным шаблоном.

Файл с данными (data_files/router.yml):
```json
ospf:
  - network: 10.0.1.0 0.0.0.255
    area: 0
  - network: 10.0.2.0 0.0.0.255
    area: 2
  - network: 10.1.1.0 0.0.0.255
    area: 0
bgp:
  local_as: 100
  loopback: lo100
  ibgp_neighbors:
    - 10.0.0.2
    - 10.0.0.3
  ebgp_neighbors:
    90.1.1.1: 500
    80.1.1.1: 600
log_server: 10.1.1.1
```


Результат выполнения скрипта будет таким:
```
$ python cfg_gen.py templates/router.txt data_files/router.yml
router ospf 1
 auto-cost reference-bandwidth 10000
 network 10.0.1.0 0.0.0.255 area 0
 network 10.0.2.0 0.0.0.255 area 2
 network 10.1.1.0 0.0.0.255 area 0

router bgp 100
 neighbor 10.0.0.2 remote-as 100
 neighbor 10.0.0.2 update-source lo100
 neighbor 10.0.0.3 remote-as 100
 neighbor 10.0.0.3 update-source lo100
 neighbor 90.1.1.1 remote-as 500
 neighbor 80.1.1.1 remote-as 600

logging 10.1.1.1
```

Таким образом, благодаря include, мы использовали шаблон templates/ospf.txt и в шаблоне templates/switch.txt, и в шаблоне templates/router.txt, вместо того, чтобы повторять одно и то же дважды.
