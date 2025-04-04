# OTUS.Lesson33.OSPF
Домашнее задание
1. Развернуть 3 виртуальные машины
2. Объединить их разными vlan
- настроить OSPF между машинами на базе Quagga;
- изобразить ассиметричный роутинг;
- сделать один из линков "дорогим", но что бы при этом роутинг был симметричным.

Тестовый стенд 3 (2CPU 4Gb RAM 6Gb HDD OS Debian 12)

1. Разворачиваем 3 виртуальные машины
    туть скрин

2) Настройка OSPF между машинами на базе Quagga

Запустим playbook ospf_PART1.yml
После выполнения плейбука смотрим ospf маршруты на R1
```
R1# sh ip r ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/1] is directly connected, ens19, weight 1, 00:06:17
O>* 10.0.11.0/30 [110/2] via 10.0.10.2, ens19, weight 1, 00:05:37
  *                      via 10.0.12.2, ens20, weight 1, 00:05:37
O   10.0.12.0/30 [110/1] is directly connected, ens20, weight 1, 00:05:42
O   192.168.10.0/24 [110/1] is directly connected, ens21, weight 1, 00:06:17
O>* 192.168.20.0/24 [110/2] via 10.0.10.2, ens19, weight 1, 00:05:37
O>* 192.168.30.0/24 [110/2] via 10.0.12.2, ens20, weight 1, 00:05:42
```
делаем трейс 
```
zhurkin@R1:~$ traceroute 192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
 1  192.168.30.1 (192.168.30.1)  0.600 ms  0.532 ms  0.473 ms
```
На всех остальных роутерах тоже можем проверить связанность, все сетки доступны

3) Настройка ассиметричного роутинга
Изменим стоимость маршрута от R1(ens20) до R3
запустим плейбук ospf_PART2.yml
После выполнения проверим маршруты и трейс до R3
```
R1# sh ip r ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/1] is directly connected, ens19, weight 1, 00:06:39
O>* 10.0.11.0/30 [110/2] via 10.0.10.2, ens19, weight 1, 00:06:04
O   10.0.12.0/30 [110/3] via 10.0.10.2, ens19, weight 1, 00:05:59
O   192.168.10.0/24 [110/1] is directly connected, ens21, weight 1, 00:06:39
O>* 192.168.20.0/24 [110/2] via 10.0.10.2, ens19, weight 1, 00:06:04
O>* 192.168.30.0/24 [110/3] via 10.0.10.2, ens19, weight 1, 00:05:59
```

zhurkin@R1:~$ traceroute 192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
 1  10.0.10.2 (10.0.10.2)  0.644 ms  0.542 ms  0.443 ms
 2  192.168.30.1 (192.168.30.1)  1.079 ms  1.010 ms  0.921 ms

Посмотрим tcpdump'ом пакеты на R2 и R3

как видно на R2
```
zhurkin@R2:~$ sudo tcpdump -ni any icmp 
sudo: unable to resolve host R2: Неизвестное имя или служба
tcpdump: data link type LINUX_SLL2
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
15:29:50.148889 ens19 In  IP 192.168.10.1 > 192.168.30.1: ICMP echo request, id 41202, seq 1, length 64
15:29:50.148929 ens20 Out IP 192.168.10.1 > 192.168.30.1: ICMP echo request, id 41202, seq 1, length 64
```

как видно на R3
```
root@R3:~# tcpdump -ni any icmp
tcpdump: data link type LINUX_SLL2
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
15:29:50.146364 ens19 In  IP 192.168.10.1 > 192.168.30.1: ICMP echo request, id 41202, seq 1, length 64
15:29:50.146482 ens20 Out IP 192.168.30.1 > 192.168.10.1: ICMP echo reply, id 41202, seq 1, length 64
```

4) Настройка симметичного роутинга

Изменим стоимость маршрута от R3(ens20) до R1
запустим плейбук ospf_PART3.yml
Сразу проверим маршруты на R3
```
R3# sh ip r ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O>* 10.0.10.0/30 [110/2] via 10.0.11.2, ens19, weight 1, 00:00:02
  *                      via 10.0.12.1, ens20, weight 1, 00:00:02
O   10.0.11.0/30 [110/1] is directly connected, ens19, weight 1, 00:00:42
O   10.0.12.0/30 [110/1] is directly connected, ens20, weight 1, 00:00:42
O>* 192.168.10.0/24 [110/2] via 10.0.12.1, ens20, weight 1, 00:00:07
O>* 192.168.20.0/24 [110/2] via 10.0.11.2, ens19, weight 1, 00:00:02
O   192.168.30.0/24 [110/1] is directly connected, ens21, weight 1, 00:00:42
```

После выполнения проверим маршруты на R3 и трейс от R1 до R3

```
R3# sh ip r ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O>* 10.0.10.0/30 [110/2] via 10.0.11.2, ens19, weight 1, 00:01:56
O   10.0.11.0/30 [110/1] is directly connected, ens19, weight 1, 00:02:31
O   10.0.12.0/30 [110/1000] is directly connected, ens20, weight 1, 00:02:30
O>* 192.168.10.0/24 [110/3] via 10.0.11.2, ens19, weight 1, 00:01:51
O>* 192.168.20.0/24 [110/2] via 10.0.11.2, ens19, weight 1, 00:01:56
O   192.168.30.0/24 [110/1] is directly connected, ens21, weight 1, 00:02:30
```
На R3 теперь один маршрут в сеть 192.168.10.0/24
```
root@R1:~# traceroute  192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
 1  10.0.10.2 (10.0.10.2)  0.540 ms  0.483 ms  0.455 ms
 2  192.168.30.1 (192.168.30.1)  1.248 ms  1.210 ms  1.181 ms
```
как видно на R2
```
15:33:23.028955 ens19 In  IP 192.168.10.1 > 192.168.30.1: ICMP echo request, id 43055, seq 1, length 64
15:33:23.029017 ens20 Out IP 192.168.10.1 > 192.168.30.1: ICMP echo request, id 43055, seq 1, length 64
15:33:23.029602 ens20 In  IP 192.168.30.1 > 192.168.10.1: ICMP echo reply, id 43055, seq 1, length 64
15:33:23.029649 ens19 Out IP 192.168.30.1 > 192.168.10.1: ICMP echo reply, id 43055, seq 1, length 64
```

как видно на R3
```
15:33:23.026978 ens19 In  IP 192.168.10.1 > 192.168.30.1: ICMP echo request, id 43055, seq 1, length 64
15:33:23.027101 ens19 Out IP 192.168.30.1 > 192.168.10.1: ICMP echo reply, id 43055, seq 1, length 64
```
