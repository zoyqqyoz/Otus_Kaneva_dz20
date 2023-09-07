ДЗ: Сценарии iptables
Описание/Пошаговая инструкция выполнения домашнего задания:

Что нужно сделать?

    реализовать knocking port

    centralRouter может попасть на ssh inetrRouter через knock скрипт
    пример в материалах.

    добавить inetRouter2, который виден(маршрутизируется (host-only тип сети для виртуалки)) с хоста или форвардится порт через локалхост.
    запустить nginx на centralServer.
    пробросить 80й порт на inetRouter2 8080.
    дефолт в инет оставить через inetRouter.
    Формат сдачи ДЗ - vagrant + ansible
    реализовать проход на 80й порт без маскарадинга*


1. реализовать knocking port:  Создаём Vagrantfile с 4-мя ВМ: inetRouter, inetRouter2, centralRouter, centralServer, запускаем его:

```
vagrant up
neva@Uneva:~$ vagrant status
Current machine states:

inetRouter                running (virtualbox)
inetRouter2               running (virtualbox)
centralRouter             running (virtualbox)
centralServer             running (virtualbox)
```

Пробуем подключиться по ssh на стандартный порт к inetRouter с centralRouter

```
neva@Uneva:~$ vagrant ssh centralRouter
[vagrant@centralRouter ~]$ ssh 192.168.255.1
^C
```
Подключение не открывается.


Выполняем на centralRouter port knocking через nmap в последовательности, разрешающей подключение по ssh: порты 8881, 7777, 9991

```	
[vagrant@centralRouter ~]$ ^C
[vagrant@centralRouter ~]$ for x in 8881 7777 9991; do sudo nmap -Pn --host_timeout 100 --max-retries 0 -p $x 192.168.255.1; done

Starting Nmap 6.40 ( http://nmap.org ) at 2023-09-06 14:51 UTC
Warning: 192.168.255.1 giving up on port because retransmission cap hit (0).
Nmap scan report for 192.168.255.1
Host is up (0.00056s latency).
PORT     STATE    SERVICE
8881/tcp filtered unknown
MAC Address: 08:00:27:D3:47:EE (Cadmus Computer Systems)

Nmap done: 1 IP address (1 host up) scanned in 0.34 seconds

Starting Nmap 6.40 ( http://nmap.org ) at 2023-09-06 14:51 UTC
Warning: 192.168.255.1 giving up on port because retransmission cap hit (0).
Nmap scan report for 192.168.255.1
Host is up (0.00076s latency).
PORT     STATE    SERVICE
7777/tcp filtered cbt
MAC Address: 08:00:27:D3:47:EE (Cadmus Computer Systems)

Nmap done: 1 IP address (1 host up) scanned in 0.36 seconds

Starting Nmap 6.40 ( http://nmap.org ) at 2023-09-06 14:51 UTC
Warning: 192.168.255.1 giving up on port because retransmission cap hit (0).
Nmap scan report for 192.168.255.1
Host is up (0.00053s latency).
PORT     STATE    SERVICE
9991/tcp filtered issa
MAC Address: 08:00:27:D3:47:EE (Cadmus Computer Systems)

Nmap done: 1 IP address (1 host up) scanned in 0.33 seconds
[vagrant@centralRouter ~]$ ssh 192.168.255.1
The authenticity of host '192.168.255.1 (192.168.255.1)' can't be established.
ECDSA key fingerprint is SHA256:QU+WTtrYu22grqYhmg/63mvi7yllYZzn2Ch5CBlk6bA.
ECDSA key fingerprint is MD5:9b:e9:0c:42:51:1e:e4:4c:fa:dc:8d:b7:20:42:f6:91.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.255.1' (ECDSA) to the list of known hosts.
vagrant@192.168.255.1's password:
[vagrant@inetRouter ~]$
```
Подключение удалось.

2. Добавить inetRouter2, который виден(маршрутизируется (host-only тип сети для виртуалки)) с хоста или форвардится порт через локалхост: реализовано в Vagrantfile:

```
box.vm.network "forwarded_port", guest: 8080, host: 1212, host_ip: "127.0.0.1", id: "nginx"
```

3. Запустить nginx на centralServer. Реализовано в Vagrantfile следующим образом:

```
sudo yum install -y epel-release; sudo yum install -y nginx; sudo systemctl enable nginx; sudo systemctl start nginx
```

Проверим статус nginx:

```
neva@Uneva:~$ vagrant ssh centralServer
[vagrant@centralServer ~]$ systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2023-09-06 06:48:37 UTC; 24min ago
  Process: 623 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 611 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 603 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 626 (nginx)
   CGroup: /system.slice/nginx.service
           ├─626 nginx: master process /usr/sbin/nginx
           └─628 nginx: worker process 
```
 
4. Пробросить 80й порт на inetRouter2 8080. Реализовано следующим образом:
 когда мы переходим с хостовой машины на  127.0.0.1:1212, то попадаем на порт 8080 inetRouter2 eth0. По правилам маршрутизации он форвардит трафик серез centralRouter (192.168.255.3) на centralServer порт 80 (nginx). Обратно трафик возвращается таким же путём. [См. схему](https://github.com/zoyqqyoz/Otus_Kaneva_dz20/blob/master/plan.pdf).

```
#Forward 8080 to nginx 80
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 192.168.0.2:80
#Back to inetRouter2
sudo iptables -t nat -A POSTROUTING -d 192.168.0.2/32 -p tcp -m tcp --dport 80 -j SNAT --to-source 192.168.255.2
```
[см. скриншот](https://github.com/zoyqqyoz/Otus_Kaneva_dz20/blob/master/nginx.JPG)

5. Дефолт в инет оставить через inetRouter
Шлюз по умолчанию прописан на inetRouter

```
echo "DEFROUTE=no" >> /etc/sysconfig/network-scripts/ifcfg-eth0 
echo "GATEWAY=192.168.0.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
```
