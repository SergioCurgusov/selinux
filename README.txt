1)Запустить nginx на нестандартном порту 3-мя разными способами


nginx не стартует

[root@selinux yum.repos.d]# systemctl status nginx    
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Fri 2024-07-05 20:23:17 UTC; 3min 50s ago
  Process: 2851 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 2850 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)

Jul 05 20:23:17 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jul 05 20:23:17 selinux nginx[2851]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jul 05 20:23:17 selinux nginx[2851]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Jul 05 20:23:17 selinux nginx[2851]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jul 05 20:23:17 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Jul 05 20:23:17 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Jul 05 20:23:17 selinux systemd[1]: Unit nginx.service entered failed state.
Jul 05 20:23:17 selinux systemd[1]: nginx.service failed.

Проверяем, работает ли файрволл.

[root@selinux yum.repos.d]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)

Правильно ли настроена конфигурация.

[root@selinux yum.repos.d]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

Включен ли selinux

[root@selinux yum.repos.d]# getenforce
Enforcing

а) Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool

В логах selinux ищем срабатывание на порт 4881

[root@selinux yum.repos.d]# cat /var/log/audit/audit.log | grep 4881
type=AVC msg=audit(1720210997.155:811): avc:  denied  { name_bind } for  pid=2851 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

Ищем по времени:
grep $(cat /var/log/audit/audit.log | grep 4881 | awk '{print $2}' | sed 's/msg=audit(//g' | sed 's/)://g' | head -1) /var/log/audit/audit.log | audit2why

[root@selinux vagrant]# grep $(cat /var/log/audit/audit.log | grep 4881 | awk '{print $2}' | sed 's/msg=audit(//g' | sed 's/)://g' | head -1) /var/log/audit/audit.log | audit2why@selinux vagrant]# grep $(cat /v
type=AVC msg=audit(1720267904.298:811): avc:  denied  { name_bind } for  pid=2847 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1

Т.е. утилита audit2why показывает, почему nginx блокируется и что с этим сделать. Делаем это:

setsebool -P nis_enabled 1

[root@selinux vagrant]# systemctl restart nginx
[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2024-07-06 12:48:03 UTC; 14s ago
  Process: 3083 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3081 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3080 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3085 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3085 nginx: master process /usr/sbin/nginx
           └─3087 nginx: worker process

Jul 06 12:48:03 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jul 06 12:48:03 selinux nginx[3081]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jul 06 12:48:03 selinux nginx[3081]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jul 06 12:48:03 selinux systemd[1]: Started The nginx HTTP and reverse proxy server

nginx запущен.

Проверяем из браузера:

lynx http://127.0.0.1:4881

Проверяем статус nis_enabled

[root@selinux vagrant]# getsebool -a | grep nis_enabled
nis_enabled --> on

Вернём запрет обратно и перейдём к следующему способу:
setsebool -P nis_enabled 0

б) Теперь разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип:

semanage port -l | grep http

[root@selinux vagrant]# semanage port -l | grep http 
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989

Добавим порт в тип http_port_t:

semanage port -a -t http_port_t -p tcp 4881

Проверка:

[root@selinux vagrant]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989

Перезапускаем nginx:

systemctl restart nginx

Проверка:
systemctl status nginx

[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2024-07-06 14:10:41 UTC; 11s ago
  Process: 22216 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22214 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 22213 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22218 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22218 nginx: master process /usr/sbin/nginx
           └─22220 nginx: worker process

Jul 06 14:10:41 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jul 06 14:10:41 selinux nginx[22214]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jul 06 14:10:41 selinux nginx[22214]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jul 06 14:10:41 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
Hint: Some lines were ellipsized, use -l to show in full.

Удаляем порт из исключений и перейдём к 3-ему способу:
semanage port -d -t http_port_t -p tcp 4881

в)Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux:

Посмотрим логи SELinux, которые относятся к nginx:
grep nginx /var/log/audit/audit.log

grep nginx /var/log/audit/audit.log | audit2allow -M nginx

[root@selinux vagrant]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp

Audit2allow сформировал модуль, и сообщил нам команду, с помощью которой можно применить данный модуль:

semodule -i nginx.pp

[root@selinux vagrant]# systemctl start nginx
[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2024-07-06 18:35:38 UTC; 8s ago
  Process: 22344 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22342 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 22341 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22346 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22346 nginx: master process /usr/sbin/nginx
           └─22348 nginx: worker process

Jul 06 18:35:38 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jul 06 18:35:38 selinux nginx[22342]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jul 06 18:35:38 selinux nginx[22342]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jul 06 18:35:38 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
Hint: Some lines were ellipsized, use -l to show in full.

nginx работает.

2) Обеспечение работоспособности приложения при включенном SELinux

Ремарки я сделал в README.md

vagrant up
vagrant status

sergio@sergio-Z87P-D3:/media/VM/selinux/2/otus-linux-adm/selinux_dns_problems$ vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.

Хорошо, что теперь в принципе запустилось. Теперь можно выполнять задачу.

Подключаемся к клиенту:

vagrant ssh client

sergio@sergio-Z87P-D3:/media/VM/selinux/2/otus-linux-adm/selinux_dns_problems$ vagrant ssh client
Last login: Sat Jul  6 20:23:23 2024 from 10.0.2.2
###############################
### Welcome to the DNS lab! ###
###############################

- Use this client to test the enviroment
- with dig or nslookup. Ex:
    dig @192.168.50.10 ns01.dns.lab

- nsupdate is available in the ddns.lab zone. Ex:
    nsupdate -k /etc/named.zonetransfer.key
    server 192.168.50.10
    zone ddns.lab 
    update add www.ddns.lab. 60 A 192.168.50.15
    send

- rndc is also available to manage the servers
    rndc -c ~/rndc.conf reload

###############################
### Enjoy! ####################
###############################

Попробуем внести изменения в зону:

[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit

sudo su

Смотрим логи:

cat /var/log/audit/audit.log | audit2why

audit2why ни о чём не сигнализирует, т.е. на клиенте отсутствуют ошибки.

Подключаемся на сервер в другой вкладке терминала:

vagrant ssh ns01

sergio@sergio-Z87P-D3:/media/VM/selinux/2/otus-linux-adm/selinux_dns_problems$ vagrant ssh ns01
Last login: Sat Jul  6 20:21:33 2024 from 10.0.2.2

sudo su

Анализируем логи с помощью audit2why:

cat /var/log/audit/audit.log | audit2why

[root@ns01 vagrant]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1720298495.218:2007): avc:  denied  { create } for  pid=5298 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

[root@ns01 vagrant]# 

audit2why всё написала: вместо типа named_t используется тип etc_t. Исправляем:

ls -laZ /etc/named

[root@ns01 vagrant]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
[root@ns01 vagrant]# 

Здесь мы также видим, что контекст безопасности неправильный. Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге. Посмотреть в каком каталоги должны лежать, файлы, чтобы на них распространялись правильные политики SELinux можно с помощью команды:

sudo semanage fcontext -l | grep named

[root@ns01 vagrant]# sudo semanage fcontext -l | grep named
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
[root@ns01 vagrant]#

Изменим тип контекста безопасности для каталога /etc/named:

chcon -R -t named_zone_t /etc/named

[root@ns01 vagrant]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
[root@ns01 vagrant]# 

Переходим на клиент и вносим изменения:

[root@client vagrant]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
[root@client vagrant]# 

Видим, что ошибок нет.

[root@client vagrant]# dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.16 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 56883
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; AUTHORITY SECTION:
ddns.lab.		600	IN	SOA	ns01.dns.lab. root.dns.lab. 2711201407 3600 600 86400 600

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Sat Jul 06 20:59:53 UTC 2024
;; MSG SIZE  rcvd: 91

[root@client vagrant]# 

Изменения применились.

Теперь ещё раз исправляем playbook.yml