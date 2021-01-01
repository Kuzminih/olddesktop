# Задание №2

dig @192.168.50.10 ns01.dns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.3 <<>> @192.168.50.10 ns01.dns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 14393
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;ns01.dns.lab.			IN	A

;; ANSWER SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10

;; AUTHORITY SECTION:
dns.lab.		3600	IN	NS	ns01.dns.lab.

;; Query time: 7 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Sat Dec 19 04:05:11 UTC 2020
;; MSG SIZE  rcvd: 71

nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL #ошибка обновления ddns
> quit

Проверяем логи в name server'е нужен su

[root@ns01 vagrant]# cat /var/log/audit/audit.log | grep denied
type=AVC msg=audit(1608350908.598:1802): avc:  denied  { create } for  pid=4898 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0
type=AVC msg=audit(1608352836.719:1847): avc:  denied  { create } for  pid=4898 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0
type=AVC msg=audit(1608352878.676:1848): avc:  denied  { create } for  pid=4898 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

Вывод sealrt


[root@ns01 vagrant]# sealert -a /var/log/audit/audit.log
100% done
found 1 alerts in /var/log/audit/audit.log
--------------------------------------------------------------------------------

Traceback (most recent call last):
  File "/bin/sealert", line 510, in on_analyzer_state_change
    self.output_results()
  File "/bin/sealert", line 524, in output_results
    print siginfo.format_text()
UnicodeEncodeError: 'ascii' codec can't encode characters in position 8-16: ordinal not in range(128)

из лога видно, что идёт запрет на доступ к фаилу 
named.ddns.lab.view1.jnl


Из файла /etc/named.conf узнаем, распололжение файла зоны ddns.lab: /etc/named/dynamic/named.ddns.lab.view1. Смотрим тип файла в его контексте безопасности:


```
18.4. Configuration Examples
18.4.1. Dynamic DNS
BIND allows hosts to update their records in DNS and zone files dynamically. This is used when a host computer's IP address changes frequently and the DNS record requires real-time modification.
Use the /var/named/dynamic/ directory for zone files you want updated by dynamic DNS. Files created in or copied into this directory inherit Linux permissions that allow named to write to them. As such files are labeled with the named_cache_t type, SELinux allows named to write to them.
If a zone file in /var/named/dynamic/ is labeled with the named_zone_t type, dynamic DNS updates may not be successful for a certain period of time as the update needs to be written to a journal first before being merged. If the zone file is labeled with the named_zone_t type when the journal attempts to be merged, an error such as the following is logged:

named[PID]: dumping master file: rename: /var/named/dynamic/zone-name: permission denied

Also, the following SELinux denial message is logged:

setroubleshoot: SELinux is preventing named (named_t) "unlink" to zone-name (named_zone_t)

To resolve this labeling issue, use the restorecon utility as root:

~]# restorecon -R -v /var/named/dynamic
```
Добавляем контекст

[root@ns01 vagrant]# semanage fcontext -a -t named_cache_t '/etc/named/dynamic(/.*)?'

[root@ns01 vagrant]# restorecon -R -v /var/named/dynamic

Проверяем

[vagrant@client ~]$ dig @192.168.50.10 ns01.dns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.3 <<>> @192.168.50.10 ns01.dns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 40405
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;ns01.dns.lab.			IN	A

;; ANSWER SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10

;; AUTHORITY SECTION:
dns.lab.		3600	IN	NS	ns01.dns.lab.

;; Query time: 13 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Sat Dec 19 05:00:16 UTC 2020
;; MSG SIZE  rcvd: 71



