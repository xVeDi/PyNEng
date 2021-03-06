## Пример использования Jinja2

В этом примере логика разнесена в 3 разных файла (все файлы находятся в каталоге 1_example):
* router_template.py - шаблон
* routers_info.yml - в этом файле в виде списка словарей (в формате YAML) находится информация о маршрутизаторах, для которых нужно сгенерировать конфигурационный файл
* router_config_generator.py - в этом скрипте импортируется файл с шаблоном и считывается информация из файла в формате YAML, а затем генерируются конфигурационные файлы маршрутизаторов

Файл router_template.py
```python
# -*- coding: utf-8 -*-
from jinja2 import Template

template_r1 = Template('''
hostname {{name}}
!
interface Loopback10
 description MPLS loopback
 ip address 10.10.{{id}}.1 255.255.255.255
 !
interface GigabitEthernet0/0
 description WAN to {{name}} sw1 G0/1
!
interface GigabitEthernet0/0.1{{id}}1
 description MPLS to {{to_name}}
 encapsulation dot1Q 1{{id}}1
 ip address 10.{{id}}.1.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf hello-interval 1
 ip ospf cost 10
!
interface GigabitEthernet0/1
 description LAN {{name}} to sw1 G0/2 !
interface GigabitEthernet0/1.{{IT}}
 description PW IT {{name}} - {{to_name}}
 encapsulation dot1Q {{IT}}
 xconnect 10.10.{{to_id}}.1 {{id}}11 encapsulation mpls
 backup peer 10.10.{{to_id}}.2 {{id}}21
  backup delay 1 1
!
interface GigabitEthernet0/1.{{BS}}
 description PW BS {{name}} - {{to_name}}
 encapsulation dot1Q {{BS}}
 xconnect 10.10.{{to_id}}.1 {{to_id}}{{id}}11 encapsulation mpls
  backup peer 10.10.{{to_id}}.2 {{to_id}}{{id}}21
  backup delay 1 1
!
router ospf 10
 router-id 10.10.{{id}}.1
 auto-cost reference-bandwidth 10000
 network 10.0.0.0 0.255.255.255 area 0
 !
''')

```

Файл routers_info.yml
```yaml
- id: 11
  name: Liverpool
  to_name: LONDON
  IT: 791
  BS: 1550
  to_id: 1

- id: 12
  name: Bristol
  to_name: LONDON
  IT: 793
  BS: 1510
  to_id: 1

- id: 14
  name: Coventry
  to_name: Manchester
  IT: 892
  BS: 1650
  to_id: 2
```

Файл router_config_generator.py
```python
# -*- coding: utf-8 -*-
import yaml
from router_template import template_r1

routers = yaml.load(open('routers_info.yml'))

for router in routers:
    r1_conf = router['name']+'_r1.txt'
    with open(r1_conf,'w') as f:
        f.write(template_r1.render(router))

```

Файл router_config_generator.py:
* импортирует шаблон template_r1
* из файла routers_info.yml список параметров считывается в переменную routers

Затем в цикле перебираются объекты (словари) в списке routers:
* название файла, в который записывается итоговая конфигурация, состоит из поля name в словаре и строки _r1.txt
 * например, Liverpool_r1.txt
* файл с таким именем открывается в режиме для записи
* в файл записывается результат рендеринга шаблона с использованием текущего словаря
* конструкция with сама закрывает файл
* управление возвращается в начало цикла (пока не переберутся все словари)

Запускаем файл router_config_generator.py:
```
$ python router_config_generator.py
```

В результате получатся три конфигурационных файла:

{% codetabs name="Liverpool_r1.txt", type="text" -%}
hostname Liverpool
!
interface Loopback10
 description MPLS loopback
 ip address 10.10.11.1 255.255.255.255
!
interface GigabitEthernet0/0
 description WAN to Liverpool sw1 G0/1
!
interface GigabitEthernet0/0.1111
 description MPLS to LONDON
 encapsulation dot1Q 1111
 ip address 10.11.1.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf hello-interval 1
 ip ospf cost 10
!
interface GigabitEthernet0/1
 description LAN Liverpool to sw1 G0/2
!
interface GigabitEthernet0/1.791
 description PW IT Liverpool - LONDON
 encapsulation dot1Q 791
 xconnect 10.10.1.1 1111 encapsulation mpls
  backup peer 10.10.1.2 1121
  backup delay 1 1
!
interface GigabitEthernet0/1.1550
 description PW BS Liverpool - LONDON
 encapsulation dot1Q 1550
 xconnect 10.10.1.1 11111 encapsulation mpls
  backup peer 10.10.1.2 11121
  backup delay 1 1
!
router ospf 10
 router-id 10.10.11.1
 auto-cost reference-bandwidth 10000
 network 10.0.0.0 0.255.255.255 area 0
!
{%- language name="Bristol_r1.txt", type="py" -%}
hostname Bristol
!
interface Loopback10
 description MPLS loopback
 ip address 10.10.12.1 255.255.255.255
!
interface GigabitEthernet0/0
 description WAN to Bristol sw1 G0/1
!
interface GigabitEthernet0/0.1121
 description MPLS to LONDON
 encapsulation dot1Q 1121
 ip address 10.12.1.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf hello-interval 1
 ip ospf cost 10
!
interface GigabitEthernet0/1
 description LAN Bristol to sw1 G0/2
!
interface GigabitEthernet0/1.793
 description PW IT Bristol - LONDON
 encapsulation dot1Q 793
 xconnect 10.10.1.1 1211 encapsulation mpls
  backup peer 10.10.1.2 1221
  backup delay 1 1
!
interface GigabitEthernet0/1.1510
 description PW BS Bristol - LONDON
 encapsulation dot1Q 1510
 xconnect 10.10.1.1 11211 encapsulation mpls
  backup peer 10.10.1.2 11221
  backup delay 1 1
!
router ospf 10
 router-id 10.10.12.1
 auto-cost reference-bandwidth 10000
 network 10.0.0.0 0.255.255.255 area 0
!
{%- language name="Coventry_r1.txt", type="py" -%}
hostname Coventry
!
interface Loopback10
 description MPLS loopback
 ip address 10.10.14.1 255.255.255.255
!
interface GigabitEthernet0/0
 description WAN to Coventry sw1 G0/1
!
interface GigabitEthernet0/0.1141
 description MPLS to Manchester
 encapsulation dot1Q 1141
 ip address 10.14.1.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf hello-interval 1
 ip ospf cost 10
!
interface GigabitEthernet0/1
 description LAN Coventry to sw1 G0/2
!
interface GigabitEthernet0/1.892
 description PW IT Coventry - Manchester
 encapsulation dot1Q 892
 xconnect 10.10.2.1 1411 encapsulation mpls
  backup peer 10.10.2.2 1421
  backup delay 1 1
!
interface GigabitEthernet0/1.1650
 description PW BS Coventry - Manchester
 encapsulation dot1Q 1650
 xconnect 10.10.2.1 21411 encapsulation mpls
  backup peer 10.10.2.2 21421
  backup delay 1 1
!
router ospf 10
 router-id 10.10.14.1
 auto-cost reference-bandwidth 10000
 network 10.0.0.0 0.255.255.255 area 0
!
{%- endcodetabs %}
