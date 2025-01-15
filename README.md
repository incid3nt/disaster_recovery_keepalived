# Домашнее задание к занятию "Disaster recovery и Keepalived" - Еноктаев Олег




---

### Задание 1

- Дана схема для Cisco Packet Tracer, рассматриваемая в лекции.
- На данной схеме уже настроено отслеживание интерфейсов маршрутизаторов Gi0/1 (для нулевой группы)
- Необходимо аналогично настроить отслеживание состояния интерфейсов Gi0/0 (для первой группы).
- Для проверки корректности настройки, разорвите один из кабелей между одним из маршрутизаторов и Switch0 и запустите ping между PC0 и Server0.
- На проверку отправьте получившуюся схему в формате pkt и скриншот, где виден процесс настройки маршрутизатора.

1.
![cisco](https://github.com/incid3nt/disaster_recovery_keepalived/blob/main/img/PacketTracer_3vxH9ggbzG.png)

Код
```
Router 5 Gi0/0
ip address 192.168.0.2/24
standy version 2
standby 0 ip 192.168.0.1
standby priority 105
standby preempt
standby 0 track GigabitEthernet0/1
```

```
Router 5 Gi0/1
ip address 192.168.1.2/24
standy version 2
standby 1 ip 192.168.1.1
standby priority 50
```

```
Router 4 Gi0/0
ip address 192.168.0.3/24
standy version 2
standby 0 ip 192.168.0.1
standby preempt
standby 0 track GigabitEthernet0/1
```

```
Router 4 Gi0/1
ip address 192.168.1.3/24
standy version 2
standby 1 ip 192.168.1.1
standby 1 preempt
```
Посмотрим приоритеты маршрутизатора:
Router 5
```
Router#show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gig0/0      0    105 P Active   local           192.168.0.3     192.168.0.1    
Gig0/1      1    50    Standby  192.168.1.3     local           192.168.1.1    
```
Router 4
```
Router#show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gig0/0      0    100 P Standby  192.168.0.2     local           192.168.0.1    
Gig0/1      1    100 P Active   local           192.168.1.2     192.168.1.1   
```
---

### Задание 2

- Запустите две виртуальные машины Linux, установите и настройте сервис Keepalived как в лекции, используя пример конфигурационного файла.
- Настройте любой веб-сервер (например, nginx или simple python server) на двух виртуальных машинах
- Напишите Bash-скрипт, который будет проверять доступность порта данного веб-сервера и существование файла index.html в root-директории данного веб-сервера.
- Настройте Keepalived так, чтобы он запускал данный скрипт каждые 3 секунды и переносил виртуальный IP на другой сервер, если bash-скрипт завершался с кодом, отличным от нуля (то есть порт веб-сервера был недоступен или отсутствовал index.html). Используйте для этого секцию vrrp_script
- На проверку отправьте получившейся bash-скрипт и конфигурационный файл keepalived, а также скриншот с демонстрацией переезда плавающего ip на другой сервер в случае недоступности порта или файла index.html
установим nginx
```
ansible-playbook /etc/ansible/playbooks/nginx.yml --ask-become-pass
```
содержимое nginx
```
cat nginx.yml
- hosts: nginx
  become: yes
  become_method: sudo
  tasks:

    - name: update
      apt: update_cache=yes

    - name: Install nginx
      apt: name=nginx state=latest
```

```
cat keepalived.yml
- hosts: nginx
  become: yes
  become_method: sudo
  tasks:

    - name: update
      apt: update_cache=yes

    - name: Install keepalived
      apt: name=keepalived state=latest
```
```
oleg@DESKTOP-6TMQOI1:/etc/ansible/playbooks$ ansible-playbook /etc/ansible/playbooks/keepalived.yml --ask-become-pass
BECOME password:

PLAY [nginx] ********************************************************************************************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************************************************************
ok: [nginx2]
ok: [nginx1]

TASK [update] *******************************************************************************************************************************************************************************
ok: [nginx2]
ok: [nginx1]

TASK [Install keepalived] *******************************************************************************************************************************************************************
ok: [nginx1]
ok: [nginx2]

PLAY RECAP **********************************************************************************************************************************************************************************
nginx1                     : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
nginx2                     : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

```
ansible all -m shell -a "cat /etc/keepalived/keepalived.conf"
nginx2 | CHANGED | rc=0 >>
vrrp_instance VI_1 {
        state BACKUP
        interface ens18
        virtual_router_id 100
        priority 200
        advert_int 1

        virtual_ipaddress {
              192.168.1.100/24
        }

}
nginx1 | CHANGED | rc=0 >>
vrrp_instance VI_1 {
        state MASTER
        interface ens18
        virtual_router_id 100
        priority 255
        advert_int 1

        virtual_ipaddress {
              192.168.1.100/24
        }

}
```

```
ansible all -m shell -a "systemctl status keepalived" --ask-become-pass
BECOME password:
nginx2 | CHANGED | rc=0 >>
● keepalived.service - Keepalive Daemon (LVS and VRRP)
     Loaded: loaded (/lib/systemd/system/keepalived.service; enabled; preset: enabled)
     Active: active (running) since Thu 2025-01-16 00:02:26 MSK; 34s ago
       Docs: man:keepalived(8)
             man:keepalived.conf(5)
             man:genhash(1)
             https://keepalived.org
   Main PID: 3532 (keepalived)
      Tasks: 2 (limit: 2302)
     Memory: 4.5M
        CPU: 9ms
     CGroup: /system.slice/keepalived.service
             ├─3532 /usr/sbin/keepalived --dont-fork
             └─3533 /usr/sbin/keepalived --dont-fork
nginx1 | CHANGED | rc=0 >>
● keepalived.service - Keepalive Daemon (LVS and VRRP)
     Loaded: loaded (/lib/systemd/system/keepalived.service; enabled; preset: enabled)
     Active: active (running) since Thu 2025-01-16 00:02:40 MSK; 19s ago
       Docs: man:keepalived(8)
             man:keepalived.conf(5)
             man:genhash(1)
             https://keepalived.org
   Main PID: 3512 (keepalived)
      Tasks: 2 (limit: 2302)
     Memory: 4.5M
        CPU: 8ms
     CGroup: /system.slice/keepalived.service
             ├─3512 /usr/sbin/keepalived --dont-fork
             └─3513 /usr/sbin/keepalived --dont-fork
```
```
ansible all -m shell -a "sudo systemctl status keepalived | grep STATE"
nginx2 | CHANGED | rc=0 >>
янв 16 00:33:26 nginx2 Keepalived_vrrp[4160]: (VI_1) Entering BACKUP STATE (init)
nginx1 | CHANGED | rc=0 >>
янв 16 00:32:41 nginx1 Keepalived_vrrp[4174]: (VI_1) Entering MASTER STATE
```
гасим keepalived на nginx1
```
sudo systemctl stop keepalived
```
проверяем статус:
```
ansible all -m shell -a "sudo systemctl status keepalived | grep STATE
"
nginx1 | CHANGED | rc=0 >>
янв 16 00:46:23 nginx1 Keepalived_vrrp[4313]: (VI_1) Entering MASTER STATE
nginx2 | CHANGED | rc=0 >>
янв 16 00:33:26 nginx2 Keepalived_vrrp[4160]: (VI_1) Entering BACKUP STATE (init)
янв 16 00:44:36 nginx2 Keepalived_vrrp[4160]: (VI_1) Entering MASTER STATE
янв 16 00:45:07 nginx2 Keepalived_vrrp[4160]: (VI_1) Entering BACKUP STATE
янв 16 00:46:03 nginx2 Keepalived_vrrp[4160]: (VI_1) Entering MASTER STATE
янв 16 00:46:23 nginx2 Keepalived_vrrp[4160]: (VI_1) Entering BACKUP STATE
янв 16 00:47:44 nginx2 Keepalived_vrrp[4160]: (VI_1) Entering MASTER STATE
```
запустим обратно
```
sudo systemctl start keepalived
```
смотрим:
```
 ansible all -m shell -a "sudo systemctl status keepalived | grep STATE
"
nginx2 | CHANGED | rc=0 >>
янв 16 00:44:36 nginx2 Keepalived_vrrp[4160]: (VI_1) Entering MASTER STATE
янв 16 00:45:07 nginx2 Keepalived_vrrp[4160]: (VI_1) Entering BACKUP STATE
янв 16 00:46:03 nginx2 Keepalived_vrrp[4160]: (VI_1) Entering MASTER STATE
янв 16 00:46:23 nginx2 Keepalived_vrrp[4160]: (VI_1) Entering BACKUP STATE
янв 16 00:47:44 nginx2 Keepalived_vrrp[4160]: (VI_1) Entering MASTER STATE
янв 16 00:50:19 nginx2 Keepalived_vrrp[4160]: (VI_1) Entering BACKUP STATE
nginx1 | CHANGED | rc=0 >>
янв 16 00:50:19 nginx1 Keepalived_vrrp[4363]: (VI_1) Entering MASTER STATE
```
### Задание 3

- Изучите дополнительно возможность Keepalived, которая называется vrrp_track_file
- Напишите bash-скрипт, который будет менять приоритет внутри файла в зависимости от нагрузки на виртуальную машину (можно разместить данный скрипт в cron и запускать каждую минуту). - - - Рассчитывать приоритет можно, например, на основании Load average.
- Настройте Keepalived на отслеживание данного файла.
- Нагрузите одну из виртуальных машин, которая находится в состоянии MASTER и имеет активный виртуальный IP и проверьте, чтобы через некоторое время она перешла в состояние SLAVE из-за высокой нагрузки и виртуальный IP переехал на другой, менее нагруженный сервер.
- Попробуйте выполнить настройку keepalived на третьем сервере и скорректировать при необходимости формулу так, чтобы плавающий ip адрес всегда был прикреплен к серверу, имеющему наименьшую нагрузку.
- На проверку отправьте получившийся bash-скрипт и конфигурационный файл keepalived, а также скриншоты логов keepalived с серверов при разных нагрузках

