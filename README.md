# SELinux

1. Запустить nginx на нестандартном порту 3-мя разными способами:
 
- переключатели setsebool;
 
- добавление нестандартного порта в имеющийся тип;

- формирование и установка модуля SELinux.

2. Обеспечить работоспособность приложения при включенном selinux.

- развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;

- выяснить причину неработоспособности механизма обновления зоны (см. README);

- предложить решение (или решения) для данной проблемы;

- выбрать одно из решений для реализации, предварительно обосновав выбор;

- реализовать выбранное решение и продемонстрировать его работоспособность.


---

### 1. Запустить nginx на нестандартном порту 3-мя разными способами

Результатом выполнения команды vagrant up станет созданная виртуальная машина с установленным nginx, который работает на порту TCP 4881.
Порт TCP 4881 уже проброшен до хоста. SELinux включен.   

<summary> Во время развёртывания стенда попытка запустить nginx завершится с ошибкой: </summary>
  
```
    selinux: Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
    selinux: ● nginx.service - The nginx HTTP and reverse proxy server
    selinux:    Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
    selinux:    Active: failed (Result: exit-code) since Tue 2024-11-19 16:41:28 UTC; 14ms ago
    selinux:   Process: 13765 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
    selinux:   Process: 13764 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    selinux:
    selinux: Nov 19 16:41:28 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
    selinux: Nov 19 16:41:28 selinux nginx[13765]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    selinux: Nov 19 16:41:28 selinux nginx[13765]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
    selinux: Nov 19 16:41:28 selinux nginx[13765]: nginx: configuration file /etc/nginx/nginx.conf test failed
    selinux: Nov 19 16:41:28 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
    selinux: Nov 19 16:41:28 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
    selinux: Nov 19 16:41:28 selinux systemd[1]: Unit nginx.service entered failed state.
```

  - Примечание: Данная ошибка появляется из-за того, что SELinux блокирует работу nginx на нестандартном порту.


**Посмотрим статус сервиса Nginx**
```
systemctl status nginx.service
```


<summary> результат выполнения команды: </summary>

```
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Tue 2024-11-19 16:41:28 UTC; 28min ago
  Process: 13765 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 13764 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)

Nov 19 16:41:28 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Nov 19 16:41:28 selinux nginx[13765]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Nov 19 16:41:28 selinux nginx[13765]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Nov 19 16:41:28 selinux nginx[13765]: nginx: configuration file /etc/nginx/nginx.conf test failed
Nov 19 16:41:28 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Nov 19 16:41:28 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Nov 19 16:41:28 selinux systemd[1]: Unit nginx.service entered failed state.
Nov 19 16:41:28 selinux systemd[1]: nginx.service failed.
```

**Проверим, что в ОС отключен файервол:**
```
systemctl status firewalld
```

<summary> результат выполнения команды: </summary>

```
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```

**Проверим, что конфигурация nginx корректна:**
```
nginx -t
```

<summary> результат выполнения команды: </summary>

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

**Проверим режим работы SELinux:**
```
getenforce
```

<summary> результат выполнения команды: </summary>

```
Enforcing
```

  - Примечание: Режим **Enforcing** означает, что SELinux будет блокировать запрещенную активность.   
   
**Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool:**   
   
Находим в логах (**/var/log/audit/audit.log**) информацию о блокировании порта **4881** (порт из статуса сервиса nginx):
```
cat /var/log/audit/audit.log | grep 4881
```

<summary> результат выполнения команды: </summary>

```
type=AVC msg=audit(1732034488.921:1114): avc:  denied  { name_bind } for  pid=13765 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```

Возьмем время из результата предыдущей команды (в которое был записан этот лог) и, с помощью утилиты audit2why смотрим:

```
grep 1732034488.921:1114 /var/log/audit/audit.log | audit2why
```

<summary> результат выполнения команды: </summary>

```
type=AVC msg=audit(1732034488.921:1114): avc:  denied  { name_bind } for  pid=13765 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
```

  - Примечание: Утилита **audit2why** покажет почему трафик блокируется. Исходя из вывода утилиты, мы видим, что нам нужно поменять параметр **nis_enabled**   

Включим параметр **nis_enabled** и перезапустим nginx:

```
setsebool -P nis_enabled 1
```
```
systemctl restart nginx
```
```
systemctl status nginx
```

<summary> результат выполнения команд: </summary>

```
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2024-11-19 17:27:16 UTC; 5s ago
  Process: 32730 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 32728 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 32727 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 32732 (nginx)
   CGroup: /system.slice/nginx.service
           ├─32732 nginx: master process /usr/sbin/nginx
           └─32734 nginx: worker process

Nov 19 17:27:16 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Nov 19 17:27:16 selinux nginx[32728]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Nov 19 17:27:16 selinux nginx[32728]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Nov 19 17:27:16 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```


#### Видим, что Nginx теперь работает   

  - Примечание: Проверить статус параметра можно с помощью команды:

<summary> getsebool -a | grep nis_enabled </summary>

```
nis_enabled --> on
```

**Вернём запрет работы nginx на порту 4881 обратно. Для этого отключим nis_enabled:**
```
setsebool -P nis_enabled 0
```

**Пробуем запустить Nginx (он снова не запустится):**

```
systemctl restart nginx.service; systemctl status nginx.service
```

<summary> результат выполнения команд: </summary>

```
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Tue 2024-11-19 17:37:21 UTC; 13ms ago
  Process: 32730 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 339 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 336 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 32732 (code=exited, status=0/SUCCESS)

Nov 19 17:37:21 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Nov 19 17:37:21 selinux nginx[339]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Nov 19 17:37:21 selinux nginx[339]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Nov 19 17:37:21 selinux nginx[339]: nginx: configuration file /etc/nginx/nginx.conf test failed
Nov 19 17:37:21 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Nov 19 17:37:21 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Nov 19 17:37:21 selinux systemd[1]: Unit nginx.service entered failed state.
Nov 19 17:37:21 selinux systemd[1]: nginx.service failed.
```


**Теперь разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип:**   

**Выполним поиск имеющегося типа, для http трафика:**
```
semanage port -l | grep http
```

<summary> результат выполнения команд: </summary>

```
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

Добавим порт в тип **http_port_t** и проверим, что порт добавился:
```
semanage port -a -t http_port_t -p tcp 4881
```
```
semanage port -l | grep  http_port_t
```

<summary> результат выполнения команд: </summary>

```
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```

**Попробуем запустить Nginx и проверим статус службы:**

```
systemctl restart nginx.service; systemctl status nginx.service
```

<summary> результат выполнения команды: </summary>

```
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2024-11-19 17:45:00 UTC; 12ms ago
  Process: 366 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 364 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 363 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 369 (nginx)
   CGroup: /system.slice/nginx.service
           ├─369 nginx: master process /usr/sbin/nginx
           └─373 nginx: master process /usr/sbin/nginx

Nov 19 17:45:00 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Nov 19 17:45:00 selinux nginx[364]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Nov 19 17:45:00 selinux nginx[364]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Nov 19 17:45:00 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

#### Видим, что Nginx снова работает 
   
**Удалим нестандартный порт (4881) из имеющегося типа и проверим, что порт удалился:**

```
semanage port -d -t http_port_t -p tcp 4881
```
```
semanage port -l | grep  http_port_t
```

<summary> результат выполнения команд: </summary>

```
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```

**Пробуем запустить Nginx (он снова не запустится):**

```
systemctl restart nginx.service; systemctl status nginx.service
```

<summary> результат выполнения команд: </summary>

```
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Tue 2024-11-19 17:51:34 UTC; 13ms ago
  Process: 366 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 392 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 391 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 369 (code=exited, status=0/SUCCESS)

Nov 19 17:51:34 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Nov 19 17:51:34 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Nov 19 17:51:34 selinux nginx[392]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Nov 19 17:51:34 selinux nginx[392]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Nov 19 17:51:34 selinux nginx[392]: nginx: configuration file /etc/nginx/nginx.conf test failed
Nov 19 17:51:34 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Nov 19 17:51:34 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Nov 19 17:51:34 selinux systemd[1]: Unit nginx.service entered failed state.
Nov 19 17:51:34 selinux systemd[1]: nginx.service failed.
```

**Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux:**  

Попробуем снова запустить nginx:

```
systemctl restart nginx.service; systemctl status nginx.service
```

<summary> результат выполнения команды: </summary>

```
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```

Nginx не запуститься, так как SELinux продолжает его блокировать.   
**Посмотрим логи SELinux, которые относятся к nginx:**

```
systemctl restart nginx.service; systemctl status nginx.service
```

<summary> результат выполнения команды: </summary>

```
type=SOFTWARE_UPDATE msg=audit(1732034488.421:1112): pid=13691 uid=0 auid=1000 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='sw="nginx-filesystem-1:1.20.1-10.el7.noarch" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=SOFTWARE_UPDATE msg=audit(1732034488.656:1113): pid=13691 uid=0 auid=1000 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='sw="nginx-1:1.20.1-10.el7.x86_64" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=AVC msg=audit(1732034488.921:1114): avc:  denied  { name_bind } for  pid=13765 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1732034488.921:1114): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=55957336c8b8 a2=10 a3=7fff68147590 items=0 ppid=1 pid=13765 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1732034488.921:1115): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=SERVICE_START msg=audit(1732037236.975:1165): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=SERVICE_STOP msg=audit(1732037805.155:1170): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=AVC msg=audit(1732037805.186:1171): avc:  denied  { name_bind } for  pid=32756 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1732037805.186:1171): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=55a4d751e8b8 a2=10 a3=7ffee020fc40 items=0 ppid=1 pid=32756 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1732037805.186:1172): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=AVC msg=audit(1732037841.651:1173): avc:  denied  { name_bind } for  pid=339 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1732037841.651:1173): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=5573d69808b8 a2=10 a3=7fff6370c2d0 items=0 ppid=1 pid=339 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1732037841.651:1174): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=SERVICE_START msg=audit(1732038300.470:1178): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=SERVICE_STOP msg=audit(1732038694.206:1182): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=AVC msg=audit(1732038694.238:1183): avc:  denied  { name_bind } for  pid=392 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1732038694.238:1183): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=561a5fb338b8 a2=10 a3=7ffe0d8da3a0 items=0 ppid=1 pid=392 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1732038694.248:1184): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=AVC msg=audit(1732038865.736:1185): avc:  denied  { name_bind } for  pid=406 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1732038865.736:1185): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=562ec69518b8 a2=10 a3=7ffea0dc5a20 items=0 ppid=1 pid=406 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1732038865.736:1186): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
```

Воспользуемся утилитой **audit2allow** для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту:
```
grep nginx /var/log/audit/audit.log | audit2allow -M nginx
```

<summary> результат выполнения команд: </summary>

```
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```

**Audit2allow** сформировал модуль и сообщил нам команду, с помощью которой можно применить данный модуль: **semodule -i nginx.pp**   

**Выполним предложенную команду:**
```
semodule -i nginx.pp
```

**Попробуем запустить Nginx и проверим статус службы:**

```
systemctl restart nginx.service; systemctl status nginx.service
```

<summary> результат выполнения команды: </summary>

```
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2024-11-19 18:07:22 UTC; 12ms ago
  Process: 451 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 449 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 448 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 453 (nginx)
   CGroup: /system.slice/nginx.service
           ├─453 nginx: master process /usr/sbin/nginx
           └─456 nginx: master process /usr/sbin/nginx

Nov 19 18:07:22 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Nov 19 18:07:22 selinux nginx[449]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Nov 19 18:07:22 selinux nginx[449]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Nov 19 18:07:22 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```


#### Видим, что Nginx снова работает 

Доп.информация:
  - При использовании модуля изменения сохранятся после перезагрузки.
  - Просмотр всех установленных модулей: `semodule -l`
  - Для удаления модуля воспользуемся командой: `semodule -r nginx`
---
### 2. Обеспечение работоспособности приложения при включенном SELinux

Выходим из предыдущего домашнего задания 2 раза набрав `exit`  
Зайдите в папку с проектом ДЗ 2 (в нашем примере **/opt/otus/SELinux/dns**)
```
cd /opt/otus/SELinux/dns
```
**Запустите проект из папки:**
```
vagrant up
```
Результатом выполнения команды vagrant up станет созданные 2 виртуальных машины: **ns01** и **client**.   
   
Посмотреть имена виртуальных машин можно командой:

```
vagrant status
```

<summary> результат выполнения команды: </summary>

```
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.

```

    
**Зайдите в виртуальную машину (box) с именем: client**

**Попробуем внести изменения в зону:**
```
nsupdate -k /etc/named.zonetransfer.key
```
```
server 192.168.50.10
```
```
zone ddns.lab
```
```
update add www.ddns.lab. 60 A 192.168.50.15
```
```
send
```

<summary> результат выполнения команд: </summary>

```
update failed: SERVFAIL
```

Выйдем из редактора зон:
```
quit
```

Изменения внести не получилось. Давайте посмотрим логи SELinux, чтобы понять в чём может быть проблема.   
Для этого воспользуемся утилитой **audit2why**:
```
cat /var/log/audit/audit.log | audit2why
```
Видим, что на клиенте отсутствуют ошибки.   

Не закрывая сессию на клиенте, подключимся к серверу ns01 и проверим логи SELinux:   
Для этого, подключимся к машине с Vagrant в соседнем окне и выполним команду:
```
cd /opt/otus/SELinux/dns && vagrant ssh ns01
```
  - Дальнейшие действия выполняются от пользователя root. Переходим в root пользователя:
```
sudo -i
```
**Проверим логи SELinux:**
```
cat /var/log/audit/audit.log | audit2why
```

<summary> результат выполнения команды: </summary>

```
type=AVC msg=audit(1732088140.214:2333): avc:  denied  { create } for  pid=16132 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1732088371.059:2334): avc:  denied  { create } for  pid=16132 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
```


В логах мы видим, что ошибка в контексте безопасности. Вместо типа **named_t** используется тип **etc_t**   
   
Проверим данную проблему в каталоге **/etc/named**:
```
ls -laZ /etc/named
```

<summary> результат выполнения команды: </summary>

```
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```

  - Примечание: Тут мы также видим, что контекст безопасности неправильный. Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге.

**Посмотреть в каком каталоги должны лежать, файлы, чтобы на них распространялись правильные политики SELinux можно с помощью команды:**
```
semanage fcontext -l | grep named
```

<summary> результат выполнения команды: </summary>

```
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0
/etc/unbound(/.*)?                                 all files          system_u:object_r:named_conf_t:s0
/var/run/bind(/.*)?                                all files          system_u:object_r:named_var_run_t:s0
/var/log/named.*                                   regular file       system_u:object_r:named_log_t:s0
/var/run/named(/.*)?                               all files          system_u:object_r:named_var_run_t:s0
/var/named/data(/.*)?                              all files          system_u:object_r:named_cache_t:s0
/dev/xen/tapctrl.*                                 named pipe         system_u:object_r:xenctl_t:s0
/var/run/unbound(/.*)?                             all files          system_u:object_r:named_var_run_t:s0
/var/lib/softhsm(/.*)?                             all files          system_u:object_r:named_cache_t:s0
/var/lib/unbound(/.*)?                             all files          system_u:object_r:named_cache_t:s0
/var/named/slaves(/.*)?                            all files          system_u:object_r:named_cache_t:s0
/var/named/chroot(/.*)?                            all files          system_u:object_r:named_conf_t:s0
/etc/named\.rfc1912.zones                          regular file       system_u:object_r:named_conf_t:s0
/var/named/dynamic(/.*)?                           all files          system_u:object_r:named_cache_t:s0
/var/named/chroot/etc(/.*)?                        all files          system_u:object_r:etc_t:s0
/var/named/chroot/lib(/.*)?                        all files          system_u:object_r:lib_t:s0
/var/named/chroot/proc(/.*)?                       all files          <<None>>
/var/named/chroot/var/tmp(/.*)?                    all files          system_u:object_r:named_cache_t:s0
/var/named/chroot/usr/lib(/.*)?                    all files          system_u:object_r:lib_t:s0
/var/named/chroot/etc/pki(/.*)?                    all files          system_u:object_r:cert_t:s0
/var/named/chroot/run/named.*                      all files          system_u:object_r:named_var_run_t:s0
/var/named/chroot/var/named(/.*)?                  all files          system_u:object_r:named_zone_t:s0
/usr/lib/systemd/system/named.*                    regular file       system_u:object_r:named_unit_file_t:s0
/var/named/chroot/var/run/dbus(/.*)?               all files          system_u:object_r:system_dbusd_var_run_t:s0
/usr/lib/systemd/system/unbound.*                  regular file       system_u:object_r:named_unit_file_t:s0
/var/named/chroot/var/log/named.*                  regular file       system_u:object_r:named_log_t:s0
/var/named/chroot/var/run/named.*                  all files          system_u:object_r:named_var_run_t:s0
/var/named/chroot/var/named/data(/.*)?             all files          system_u:object_r:named_cache_t:s0
/usr/lib/systemd/system/named-sdb.*                regular file       system_u:object_r:named_unit_file_t:s0
/var/named/chroot/var/named/slaves(/.*)?           all files          system_u:object_r:named_cache_t:s0
/var/named/chroot/etc/named\.rfc1912.zones         regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/var/named/dynamic(/.*)?          all files          system_u:object_r:named_cache_t:s0
/var/run/ndc                                       socket             system_u:object_r:named_var_run_t:s0
/dev/gpmdata                                       named pipe         system_u:object_r:gpmctl_t:s0
/dev/initctl                                       named pipe         system_u:object_r:initctl_t:s0
/dev/xconsole                                      named pipe         system_u:object_r:xconsole_device_t:s0
/usr/sbin/named                                    regular file       system_u:object_r:named_exec_t:s0
/etc/named\.conf                                   regular file       system_u:object_r:named_conf_t:s0
/usr/sbin/lwresd                                   regular file       system_u:object_r:named_exec_t:s0
/var/run/initctl                                   named pipe         system_u:object_r:initctl_t:s0
/usr/sbin/unbound                                  regular file       system_u:object_r:named_exec_t:s0
/usr/sbin/named-sdb                                regular file       system_u:object_r:named_exec_t:s0
/var/named/named\.ca                               regular file       system_u:object_r:named_conf_t:s0
/etc/named\.root\.hints                            regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/dev                              directory          system_u:object_r:device_t:s0
/etc/rc\.d/init\.d/named                           regular file       system_u:object_r:named_initrc_exec_t:s0
/usr/sbin/named-pkcs11                             regular file       system_u:object_r:named_exec_t:s0
/etc/rc\.d/init\.d/unbound                         regular file       system_u:object_r:named_initrc_exec_t:s0
/usr/sbin/unbound-anchor                           regular file       system_u:object_r:named_exec_t:s0
/usr/sbin/named-checkconf                          regular file       system_u:object_r:named_checkconf_exec_t:s0
/usr/sbin/unbound-control                          regular file       system_u:object_r:named_exec_t:s0
/var/named/chroot_sdb/dev                          directory          system_u:object_r:device_t:s0
/var/named/chroot/var/log                          directory          system_u:object_r:var_log_t:s0
/var/named/chroot/dev/log                          socket             system_u:object_r:devlog_t:s0
/etc/rc\.d/init\.d/named-sdb                       regular file       system_u:object_r:named_initrc_exec_t:s0
/var/named/chroot/dev/null                         character device   system_u:object_r:null_device_t:s0
/var/named/chroot/dev/zero                         character device   system_u:object_r:zero_device_t:s0
/usr/sbin/unbound-checkconf                        regular file       system_u:object_r:named_exec_t:s0
/var/named/chroot/dev/random                       character device   system_u:object_r:random_device_t:s0
/var/run/systemd/initctl/fifo                      named pipe         system_u:object_r:initctl_t:s0
/var/named/chroot/etc/rndc\.key                    regular file       system_u:object_r:dnssec_t:s0
/usr/share/munin/plugins/named                     regular file       system_u:object_r:services_munin_plugin_exec_t:s0
/var/named/chroot_sdb/dev/null                     character device   system_u:object_r:null_device_t:s0
/var/named/chroot_sdb/dev/zero                     character device   system_u:object_r:zero_device_t:s0
/var/named/chroot/etc/localtime                    regular file       system_u:object_r:locale_t:s0
/var/named/chroot/etc/named\.conf                  regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot_sdb/dev/random                   character device   system_u:object_r:random_device_t:s0
/etc/named\.caching-nameserver\.conf               regular file       system_u:object_r:named_conf_t:s0
/usr/lib/systemd/systemd-hostnamed                 regular file       system_u:object_r:systemd_hostnamed_exec_t:s0
/var/named/chroot/var/named/named\.ca              regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/etc/named\.root\.hints           regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/etc/named\.caching-nameserver\.conf regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/lib64 = /usr/lib
/var/named/chroot/usr/lib64 = /usr/lib
```

  - Примечание: из всего вывода команды мы смотрим на данную строку:   
`/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0`

Изменим тип контекста безопасности для каталога **/etc/named**:
```
chcon -R -t named_zone_t /etc/named
```
   
Посмотрим на папку /etc/named:
```
ls -laZ /etc/named
```

<summary> результат выполнения команды: </summary>

```
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```


**Перейдем на клиента (client) и попробуем снова внести изменения:**
```
nsupdate -k /etc/named.zonetransfer.key
```
```
server 192.168.50.10
```
```
zone ddns.lab
```
```
update add www.ddns.lab. 60 A 192.168.50.15
```
```
send
```
```
quit
```
Теперь мы не наблюдаем ошибку.   

**Проверим применились ли изменения:**

```
dig www.ddns.lab
```

<summary> результат выполнения команды: </summary>

```
; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.16 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 19558
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.               3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Wed Nov 20 08:18:05 UTC 2024
;; MSG SIZE  rcvd: 96
```


Видим, что изменения применились.   

**Перезагрузим оба хоста `init 6`, зайдем на них и ещё раз сделаем запрос с помощью dig:**
```
dig @192.168.50.10 www.ddns.lab
```

<summary> результат выполнения команды: </summary>

```
; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.16 <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 30098
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.               3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Wed Nov 20 08:26:58 UTC 2024
;; MSG SIZE  rcvd: 96
```

**Видим, что настройки сохранились.**   

  - Доп информация:
Для того, чтобы вернуть правила обратно, можно ввести команду на сервере (ns01):
```
restorecon -v -R /etc/named
```

<summary> результат выполнения команды: </summary>

```
restorecon reset /etc/named context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.dns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.dns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic context unconfined_u:object_r:named_zone_t:s0->unconfined_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1.jnl context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.newdns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.50.168.192.rev context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
```

