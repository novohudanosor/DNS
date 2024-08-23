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
