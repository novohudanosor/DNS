# DNS
DNS- настройка и обслуживание
## Описание, что нужно сделать 
1. Vagrant дано- частично.
   * добавить еще один сервер client2
   * завести в зоне dns.lab имена:
       * web1 - смотрит на клиент1
       * web2  смотрит на клиент2
* завести еще одну зону newdns.lab
* завести в ней запись
    * www - смотрит на обоих клиентов
2. настроить split-dns
  *  клиент1 - видит обе зоны, но в зоне dns.lab только web1
  *  клиент2 видит только dns.lab

3.  ``` vagrant up ``` Идалее устанавливаем на хосты необходимое ПО:
```
yum install -y bind bind-utils ntp
```
4. Вносим изменения в файл ``` resolc.conf ``` на серверах и клиентах.
```
[vagrant@ns01 ~]$ cat /etc/resolv.conf
domain dns.lab
search dns.lab
nameserver 192.168.56.10

[vagrant@ns02 ~]$ cat /etc/resolv.conf 
domain dns.lab
search dns.lab
nameserver 192.168.56.11

[vagrant@client1 ~]$ cat /etc/resolv.conf 
domain dns.lab
search dns.lab
nameserver 192.168.56.10
nameserver 192.168.56.11

[vagrant@client2 ~]$ cat /etc/resolv.conf 
domain dns.lab
search dns.lab
nameserver 192.168.56.10
nameserver 192.168.56.11
```
5. Настраиваем зоны **dns.lab** - в файле named.dns.lab вносим информацию о имени хостов web1 и web2
```
[root@ns01 ~]# cat /etc/named/named.dns.lab
$TTL 3600
$ORIGIN dns.lab.
@               IN      SOA     ns01.dns.lab. root.dns.lab. (
                            2711201407 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
                        )

                IN      NS      ns01.dns.lab.
                IN      NS      ns02.dns.lab.

; DNS Servers
ns01            IN      A       192.168.56.10
ns02            IN      A       192.168.56.11

; Web
web1			IN		A		192.168.56.15
web2			IN		A		192.168.56.16
```
6. Проверяем конфигурацию на мастере
```
[root@ns01 ~]# cat /etc/named.conf 
options {

    // network 
	listen-on port 53 { 192.168.56.10; };
	listen-on-v6 port 53 { ::1; };

    // data
	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
	statistics-file "/var/named/data/named_stats.txt";
	memstatistics-file "/var/named/data/named_mem_stats.txt";

    // server
	recursion yes;
	allow-query     { any; };
   allow-transfer { any; };
    
    // dnssec
	dnssec-enable yes;
	dnssec-validation yes;

    // others
	bindkeys-file "/etc/named.iscdlv.key";
	managed-keys-directory "/var/named/dynamic";
	pid-file "/run/named/named.pid";
	session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

// RNDC Control for client
key "rndc-key" {
    algorithm hmac-md5;
    secret "GrtiE9kz16GK+OKKU/qJvQ==";
};
controls {
        inet 192.168.56.16 allow { 192.168.56.15; } keys { "rndc-key"; }; 
};

// ZONE TRANSFER WITH TSIG
include "/etc/named.zonetransfer.key"; 
server 192.168.56.11 {
    keys { "zonetransfer.key"; };
};

// root zone
zone "." IN {
	type hint;
	file "named.ca";
};

// zones like localhost
include "/etc/named.rfc1912.zones";
// root's DNSKEY
include "/etc/named.root.key";

// lab's zone
zone "dns.lab" {
    type master;
    allow-transfer { key "zonetransfer.key"; };
    file "/etc/named/named.dns.lab";
};

// lab's zone reverse
zone "56.168.192.in-addr.arpa" {
    type master;
    allow-transfer { key "zonetransfer.key"; };
    file "/etc/named/named.dns.lab.rev";
};

// lab's ddns zone
zone "ddns.lab" {
    type master;
    allow-transfer { key "zonetransfer.key"; };
    allow-update { key "zonetransfer.key"; };
    file "/etc/named/named.ddns.lab";
};
```
7. Проверка конфигурации на слейве. Тут нужно указать type slave адрес мастер сервера и путь к файлу с описание зон на мастер серверер.
```
[root@ns02 ~]# cat /etc/named.conf
...
// lab's zone
zone "dns.lab" {
    type slave;
    masters { 192.168.56.10; };
    file "/etc/named/named.dns.lab";
};

// lab's zone reverse
zone "56.168.192.in-addr.arpa" {
    type slave;
    masters { 192.168.56.10; };
    file "/etc/named/named.dns.lab.rev";
};
```
8. Создание новой зоны newdns.lab и добавление в неё записей:
```
[root@ns01 ~]# cat /etc/named/named.newdns.lab 
$TTL 3600
$ORIGIN newdns.lab.
@		IN	SOA	ns01.dns.lab. root.dns.lab. (
			0208202402 ; serial
			3600       ; refresh (1 hour)
			600        ; retry (10 minutes)
			86400      ; expire (1 day)
			600	   ; minimum (10 minutes) 
			)

		IN 	NS	ns01.dns.lab.
		IN 	NS	ns02.dns.lab.
; DNS Servers
ns01		IN	A	192.168.56.10
ns02		IN	A	192.168.56.11

; WWW
www		IN	A	192.168.56.15
www		IN	A	192.168.56.16 
```
9. Добавление информации о новой зоне в named.conf на серверах ns01 и ns02:
```
[root@ns01 ~]# cat /etc/named.conf 
...
// lab's newdns zone
zone "newdns.lab" {
    type master;
    allow-transfer { key "zonetransfer.key"; };
    allow-update { key "zonetransfer.key"; };
    file "/etc/named/named.newdns.lab";
};
[root@ns02 ~]# cat /etc/named.conf
...
// lab's newdns zone
zone "newdns.lab" {
    type slave;
    masters { 192.168.56.10; };
    file "/etc/named/named.newdns.lab";
};
```
10. Перезапустим сервис на серерах ``` systemctl restart named ```
11. Проверяем настройки DNS
```
[vagrant@client1 ~]$ dig @192.168.56.10 ns01.dns.lab +short
192.168.56.10
[vagrant@client1 ~]$ dig @192.168.56.10 ns02.dns.lab +short
192.168.56.11
[vagrant@client1 ~]$ dig @192.168.56.10 web1.dns.lab +short
192.168.56.15
[vagrant@client1 ~]$ dig @192.168.56.10 web2.dns.lab +short
192.168.56.16
```
## Настройка Split-DNS
1. Существуют  зоны dns.lab и newdns.lab. По условию задания client1 должен видеть запись web1.dns.lab и не видеть web2.dns.lab. Client2 может видеть обе записи из домена dns.lab, но не должен видеть записи домена newdns.lab. Данные настройки осуществляются с помощью технолгии Split-DNS.
2. Настраиваем файл ``` /etc/named/named.dns.lab.client ``` для зоны dns.lab
```
$TTL 3600
$ORIGIN dns.lab.
@               IN      SOA     ns01.dns.lab. root.dns.lab. (
                            2711201408 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
                        )

                IN      NS      ns01.dns.lab.
                IN      NS      ns02.dns.lab.

; DNS Servers
ns01            IN      A       192.168.56.10
ns02            IN      A       192.168.56.11

; Web
web1 		IN	A	192.168.56.15
```
3. Split DNS-конфигурация. Конфигурация BIND, позволяющая использовать различные настройки DNS в зависимости от адреса источника запроса. Получается, что для каждого представления настраиваются только те зоны, которые разрешено видеть хостам. Все ранее описанные зоны должны быть перенесены в модули view. Вне этих модулей зон быть не должно. Зона any должна завершать конфигурацию.
```
...
key "client1-key" {
	algorithm hmac-sha256;
	secret "F45GQqtFRmiskdsxOU/FrAhZfsi3wTtDtD2vAQL4Bn4=";
};

key "client2-key" {
	algorithm hmac-sha256;
	secret "taugThYY7DLafWBaDyBx/t29bDPR3buBdNgCK4JWCOs=";
};


// ZONE TRANSFER WITH TSIG
include "/etc/named.zonetransfer.key"; 
server 192.168.56.11 {
    keys { "zonetransfer.key"; };
};

acl client1 { !key client2-key; key client1-key; 192.168.56.15; };
acl client2 { !key client1-key; key client2-key; 192.168.56.16; };

view "client1" {
	match-clients { client1; };
	
	zone "dns.lab" {
		type master;
		file "/etc/named/named.dns.lab.client";
		also-notify { 192.168.56.11 key client1-key; };
	};

	zone "newdns.lab" {
		type master;
		file "/etc/named/named.newdns.lab";
		also-notify { 192.168.56.11 key client1-key; };
	};
};

view "client2" {
	match-clients { client2; };

	zone "dns.lab" {
		type master;
		file "/etc/named/named.dns.lab";
		also-notify { 192.168.56.11 key client2-key; };
	};

	zone "56.168.192.in-addr.arpa" {
		type master;
		file "/etc/named/named.dns.lab.rev";
		also-notify { 192.168.56.11 key client2-key; };
	};
};

view "default" {
	match-clients { any; };

	// root zone
	zone "." IN {
		type hint;
		file "named.ca";
	};

	// zones like localhost
	include "/etc/named.rfc1912.zones";
	// root's DNSKEY
	include "/etc/named.root.key";

	// lab's zone
	zone "dns.lab" {
    		type master;
    		allow-transfer { key "zonetransfer.key"; };
    		file "/etc/named/named.dns.lab";
	};

	// lab's zone reverse
	zone "56.168.192.in-addr.arpa" {
    		type master;
    		allow-transfer { key "zonetransfer.key"; };
    		file "/etc/named/named.dns.lab.rev";
	};

	// lab's ddns zone
	zone "ddns.lab" {
    		type master;
    		allow-transfer { key "zonetransfer.key"; };
    		allow-update { key "zonetransfer.key"; };
    		file "/etc/named/named.ddns.lab";
	};

	// lab's newdns zone
	zone "newdns.lab" {
    		type master;
    		allow-transfer { key "zonetransfer.key"; };
    		file "/etc/named/named.newdns.lab";
	};
};

```
4. На сервере ns02, таким же образом настраивается Split-DNS, указываем type slave, адреса мастер сервера и пути к файлу с настройками зоны на мастер сервере.
5. Перезапускаем сервис на серверах:  ``` systemctl restart named ```
6. Получаем , что сервис не запустится. Из-за SELinux.   (Тут для себя подсказка - лаба по SELinux  https://github.com/novohudanosor/SELinux). Используем утилиты **audit2why** и **audit2allow**
7. **audit2allow** анализируем  audit-лог ``` /var/log/audit/audit.log ```   Для того, чтобы запутить сервис named, необходимо воспользоваться утилитой audit2allow, которая сформирует разрешающее правило для SELinux. После чего данный модуль необходимо загрузить с помощью команды semodule -i.   Далее получаем следующее:
```
[root@client1 ~]# audit2why < /var/log/audit/audit.log 
type=AVC msg=audit(1722773754.620:1294): avc:  denied  { search } for  pid=4290 comm="isc-worker0000" name="net" dev="proc" ino=7077 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:sysctl_net_t:s0 tclass=dir

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1722773754.621:1295): avc:  denied  { search } for  pid=4290 comm="isc-worker0000" name="net" dev="proc" ino=7077 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:sysctl_net_t:s0 tclass=dir

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1722773882.911:1405): avc:  denied  { search } for  pid=4518 comm="isc-worker0000" name="net" dev="proc" ino=7077 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:sysctl_net_t:s0 tclass=dir

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1722773882.912:1406): avc:  denied  { search } for  pid=4518 comm="isc-worker0000" name="net" dev="proc" ino=7077 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:sysctl_net_t:s0 tclass=dir

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

[root@client1 ~]# audit2allow -M my_isc-worker0000 --debug < /var/log/audit/audit.log
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i my_isc-worker0000.pp

[root@client1 ~]# semodule -i my_isc-worker0000.pp

```
8. Проверка настроек Split-DNS:
```

[vagrant@client1 ~]$ dig web1.dns.lab +short
192.168.56.15
[vagrant@client1 ~]$ dig www.newdns.lab +short
192.168.56.16
192.168.56.15
[vagrant@client1 ~]$ dig web2.dns.lab +short
[vagrant@client1 ~]$

[vagrant@client2 ~]$ dig web1.dns.lab +short
192.168.56.15
[vagrant@client2 ~]$ dig web2.dns.lab +short
192.168.56.16
[vagrant@client2 ~]$ dig www.newdns.lab +short

```
