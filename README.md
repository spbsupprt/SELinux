# SELinux

- Запустить nginx на нестандартном порту 3-мя разными способами:
 
- переключатели setsebool;
 
- добавление нестандартного порта в имеющийся тип;

- формирование и установка модуля SELinux.

2. Обеспечить работоспособность приложения при включенном selinux.

- развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;

- выяснить причину неработоспособности механизма обновления зоны (см. README);

- предложить решение (или решения) для данной проблемы;

- выбрать одно из решений для реализации, предварительно обосновав выбор;

- реализовать выбранное решение и продемонстрировать его работоспособность.





Запустить nginx на нестандартном порту 3-мя разными способами

https://github.com/Nickmob/vagrant_selinux 

Результатом выполнения команды vagrant up станет созданная виртуальная машина с установленным nginx, который работает на порту TCP 4881. Порт TCP 4881 уже проброшен до хоста. SELinux включен.


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


Примечание: Данная ошибка появляется из-за того, что SELinux блокирует работу nginx на нестандартном порту.

Проверим, что в ОС отключен файервол:

systemctl status firewalld

результат выполнения команды:

● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
     
Проверим, что конфигурация nginx корректна:

nginx -t

результат выполнения команды:

nginx: the configuration file /etc/nginx/nginx.conf syntax is ok

nginx: configuration file /etc/nginx/nginx.conf test is successful

Проверим режим работы SELinux:

getenforce

результат выполнения команды:

Enforcing

Примечание: Режим Enforcing означает, что SELinux будет блокировать запрещенную активность.
Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool:

Находим в логах (/var/log/audit/audit.log) информацию о блокировании порта 4881 (порт из статуса сервиса nginx):

cat /var/log/audit/audit.log | grep 4881
результат выполнения команды:
Возьмем время из результата предыдущей команды (в которое был записан этот лог) и, с помощью утилиты audit2why смотрим:

grep 1732034488.921:1114 /var/log/audit/audit.log | audit2why
результат выполнения команды:
Примечание: Утилита audit2why покажет почему трафик блокируется. Исходя из вывода утилиты, мы видим, что нам нужно поменять параметр nis_enabled
Включим параметр nis_enabled и перезапустим nginx:

setsebool -P nis_enabled 1
systemctl restart nginx
systemctl status nginx

результат выполнения команд:
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


















