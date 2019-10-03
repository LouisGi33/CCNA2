# TP1 : Back to basics

## I. Gather informations

 - 🌞 récupérer une **liste des cartes réseau** avec leur nom, leur IP et leur adresse MAC:
 La cmd "ip a" nous donne acces à ces informations
 
 - 🌞 déterminer si les cartes réseaux ont récupéré une **IP en DHCP** ou non:
 ```
 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000                                      
 link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00                                                                   
 inet 127.0.0.1/8 scope host lo                                                                                             
 valid_lft forever preferred_lft forever                                                                              
 inet6 ::1/128 scope host                                                                                                   
 valid_lft forever preferred_lft forever                                                                          
 2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default 
 qlen 1000                       
 link/ether 08:00:27:f5:a5:81 brd ff:ff:ff:ff:ff:ff                                                                      
 inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute enp0s3                                                 
 valid_lft 84896sec preferred_lft 84896sec                                                                            
 inet6 fe80::d212:7240:e789:9378/64 scope link noprefixroute                                                                
 valid_lft forever preferred_lft forever                                                                          
 3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default 
 qlen 1000                       
 link/ether 08:00:27:e5:02:91 brd ff:ff:ff:ff:ff:ff                                                                      
 inet 192.168.76.3/24 brd 192.168.76.255 scope global dynamic noprefixroute enp0s8                                          
 valid_lft 897sec preferred_lft 897sec                                                                                
 inet6 fe80::3b84:f1d:8b32:534d/64 scope link noprefixroute                                                                 
 valid_lft forever preferred_lft forever  
 ```
On observe bien par ex que la carte enp0s8 (Host-Only) possede un parametre "dynamic" et non "static" donc celle ci est en effet bien une ip dynamique (DHCP)

On peut aussi le voir dans /etc/sysconfig/network-scripts/ifcfg-enp0s8 -> BOOTPROTO="dhcp"

Voici le bail DHCP de notre carte enp0s8 ( dispo dans /var/lib/NetworkManager ou dans /var/lib/dhclient ) :
```
ADDRESS=192.168.76.3
NETMASK=255.255.255.0
SERVER_ADDRESS=192.168.76.2
T1=600
T2=1050
LIFETIME=1200
CLIENTID=01080027e50291
```

 - 🌞 afficher la **table de routage** de la machine et sa **table ARP**
```
[root@localhost /]# ip route                                                                                            
default via 10.0.2.2 dev enp0s3 proto dhcp metric 100                                                                   
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100                                                 
192.168.76.0/24 dev enp0s8 proto kernel scope link src 192.168.76.3 metric 101
```
La premiere ligne représente la route par défaut permettant de recupérer une ip par le dhcp. Les deux autres sont les routes pour accéder aux diff réseaux .

```
[root@localhost /]# ip n                                                                                                
10.0.2.2 dev enp0s3 lladdr 52:54:00:12:35:02 REACHABLE                                                                  
192.168.76.2 dev enp0s8 lladdr 08:00:27:ba:b4:59 STALE                                                                  
192.168.76.1 dev enp0s8 lladdr 0a:00:27:00:00:14 DELAY 
```

- 🌞 récupérer **la liste des ports en écoute** (_listening_) sur la machine (TCP et UDP)

```
[root@localhost /]# ss -ltu
Netid      State        Recv-Q       Send-Q                    Local Address:Port               Peer Address:Port       
udp        UNCONN       0            0                             127.0.0.1:323                     0.0.0.0:*          
udp        UNCONN       0            0                   192.168.76.3%enp0s8:bootpc                  0.0.0.0:*          
udp        UNCONN       0            0                      10.0.2.15%enp0s3:bootpc                  0.0.0.0:*          
udp        UNCONN       0            0                                 [::1]:323                        [::]:*          
tcp        LISTEN       0            128                             0.0.0.0:ssh                     0.0.0.0:*          
tcp        LISTEN       0            128                                [::]:ssh                        [::]:*
```

- 🌞 récupérer **la liste des DNS utilisés par la machine**
```
[root@localhost /]# cat /etc/resolv.conf                                                                                
# Generated by NetworkManager                                                                                           
search home                                                                                                             
nameserver 192.168.1.1                                                                                                  
nameserver 129.250.35.251 
```
```
[root@localhost /]# dig www.reddit.com                                                                                                                                                                                                          
; <<>> DiG 9.11.4-P2-RedHat-9.11.4-17.P2.el8_0.1 <<>> www.reddit.com                                                    
;; global options: +cmd                                                                                                 
;; Got answer:                                                                                                          
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 23303                                                               
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1                                                                                                                                                                            
;; OPT PSEUDOSECTION:                                                                                                   
; EDNS: version: 0, flags:; udp: 1460                                                                                   
; COOKIE: 7834a71e3af5e2ff0632d0835d94cf97d1a809dd66371250 (good)                                                       
;; QUESTION SECTION:                                                                                                    
;www.reddit.com.                        IN      A                                                                                                                                                                                               
;; ANSWER SECTION:                                                                                                      
www.reddit.com.         23      IN      CNAME   reddit.map.fastly.net.                                                  
reddit.map.fastly.net.  11      IN      A       151.101.121.140                                                                                                                                                                                 
;; Query time: 30 msec                                                                                                  
;; SERVER: 192.168.1.1#53(192.168.1.1)                                                                                  
;; WHEN: mer. oct. 02 18:25:59 CEST 2019                                                                                
;; MSG SIZE  rcvd: 122  
```
On utilise bien le DNS enregistré sous l'Addr 192.168.1.1

- 🌞 afficher **l'état actuel du firewall**
```
[root@localhost /]# systemctl status firewalld                                                                          
â-? firewalld.service - firewalld - dynamic firewall daemon                                                                
Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)                             
Active: active (running) since Wed 2019-10-02 17:57:05 CEST; 33min ago                                                    
Docs: man:firewalld(1)                                                                                              
Main PID: 750 (firewalld)                                                                                                  
Tasks: 2 (limit: 5065)                                                                                                 
Memory: 35.6M                                                                                                           
CGroup: /system.slice/firewalld.service                                                                                         
â""â"?750 /usr/libexec/platform-python -s /usr/sbin/firewalld --nofork --nopid                                                                                                                                                       
oct. 02 17:57:03 localhost.localdomain systemd[1]: Starting firewalld - dynamic firewall 
daemon...                      
oct. 02 17:57:05 localhost.localdomain systemd[1]: Started firewalld - dynamic firewall daemon. 
```

```
[root@localhost /]# firewall-cmd --list-interfaces                                                                      
enp0s3 enp0s8
```
```
[root@localhost /]# firewall-cmd --list-all                                                                             
public (active)
  ...
  services: cockpit dhcpv6-client ssh
  ports:
  ...
```
```
[root@localhost /]# nft list table filter
table ip filter {
        chain INPUT {
                type filter hook input priority 0; policy accept;
        }

        chain FORWARD {
                type filter hook forward priority 0; policy accept;
        }

        chain OUTPUT {
                type filter hook output priority 0; policy accept;
        }
}
```

## II. Edit configuration

### 1. Configuration cartes réseau

Nouvelle config sur enp0s8 :
`[root@localhost /]# vi /etc/sysconfig/network-scripts/ifcfg-enp0s8`
```
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="none"
NAME="enp0s8"
DEVICE="enp0s8"
ONBOOT="yes"
IPADDR=192.168.76.5
NETMASK=255.255.255.0
```
Nouvelle carte : 
```
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="none"
NAME="enp0s9"
DEVICE="enp0s9"
ONBOOT="yes"
IPADDR=192.168.76.6
NETMASK=255.255.255.0
```
```
4: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 
1000                       
link/ether 08:00:27:76:7b:3b brd ff:ff:ff:ff:ff:ff                                                                      
inet 192.168.76.6/24 brd 192.168.76.255 scope global noprefixroute enp0s9                                                  
valid_lft forever preferred_lft forever                                                                              
inet6 fe80::a00:27ff:fe76:7b3b/64 scope link                                                                               
valid_lft forever preferred_lft forever 
```

- 🐙 mettre en place un NIC _teaming_ (ou _bonding_)

`[root@localhost loki]# dnf -y install teamd `

On cherche quelles cartes vont etre utilisées (dans cet ex, ce sera enp0s8 et 0s9) puis on supprime leur connexion
```
[root@localhost loki]# nmcli connection show                                                                            
NAME                 UUID                                  TYPE      DEVICE                                             
enp0s10              38a4a9f5-87b5-2aec-a00c-ca0f62e0dea9  ethernet  enp0s10                                            
enp0s3               b3cd966e-ca92-47c7-9c56-b4fc5ad6311b  ethernet  enp0s3                                             
enp0s8               00cb8299-feb9-55b6-a378-3fdc720e0bc6  ethernet  enp0s8                                             
enp0s9               93d13955-e9e2-a6bd-df73-12e3c747f122  ethernet  enp0s9                                             
Connexion filaire 1  6374934a-4b2e-38dd-8d2f-57db95e73d2d  ethernet  --       
```

```
[root@localhost loki]# nmcli connection delete 00cb8299-feb9-55b6-a378-3fdc720e0bc6                                     
Connexion Â«Â enp0s8Â Â» (00cb8299-feb9-55b6-a378-3fdc720e0bc6) supprimÃ©e.                                             
[root@localhost loki]# nmcli connection delete 93d13955-e9e2-a6bd-df73-12e3c747f122                                     
Connexion Â«Â enp0s9Â Â» (93d13955-e9e2-a6bd-df73-12e3c747f122) supprimÃ©e.   
```

```
[root@localhost loki]# nmcli device status
DEVICE   TYPE      STATE     CONNECTION
enp0s3   ethernet  connectÃ©  enp0s3
enp0s10  ethernet  connectÃ©  enp0s10
enp0s8   ethernet  connectÃ©  Connexion filaire 2
enp0s9   ethernet  connectÃ©  Connexion filaire 3
lo       loopback  non-gÃ©rÃ©  --
```
La connexion de ces 2 cartes est desactivée (meme si sont toujours connecté par le nat)

```
[root@localhost loki]# nmcli connection add type team con-name team0 ifname team0 \
> config '{ "runner": {"name": "loadbalance"}, "link_watch": {"name": "ethtool"}}'
Connexion Â«Â team0Â Â» (95d6951c-dc23-4783-a729-c291d13f2ac4) ajoutÃ©e avec succÃ¨s.
```

```
[root@localhost loki]# nmcli con mod team0 ipv4.addresses 192.168.18.18/24
[root@localhost loki]# nmcli con mod team0 ipv4.method manual
[root@localhost loki]# nmcli con mod team0 connection.autoconnect yes
```

On créé les cartes "esclaves":
```
[root@localhost loki]# nmcli con add type team-slave con-name team-slave0 ifname enp0s8 master team0
Connexion Â«Â team-slave0Â Â» (86818813-5372-4742-ae29-ae32a2900a48) ajoutÃ©e avec succÃ¨s.
[root@localhost loki]# nmcli con add type team-slave con-name team-slave1 ifname enp0s9 master team0
Connexion Â«Â team-slave1Â Â» (041c439a-92c0-4764-ae73-fbb9d32ce4c5) ajoutÃ©e avec succÃ¨s.
```
On lance la connexion de la nouvelle interface:
```
[root@localhost loki]# nmcli connection up team0
Connexion activÃ©e (master waiting for slaves) (Chemin D-Bus actif : /org/freedesktop/NetworkManager/ActiveConnection/14)
```

       

###	 2. Serveur SSH

Autoriser le port du ssh dans le firewall (ici present le port 2222, modifiable dans /etc/ssh/sshd_config)
```
[root@localhost firewalld]# firewall-cmd --permanent --add-port=2222/tcp                                                  
success
[root@localhost firewalld]# firewall-cmd --permanent --remove-port=22/tcp 
[root@localhost firewalld]# systemctl restart firewalld 
```
Vidage table arp :`ip n flush all`

![](https://i.imgur.com/ms6VkKF.png)

On constate bien les diff requetes de connexions : SYN SYN/ACK ACK
L'ip 192.168.76.1 (notre carte reseau) demande au localhost de se connecter (SYN) puis Localhost renvoit une requete comme quoi l'ip 192.168.76.1 peut se connecter (SYN/ACK) pour qu'enfin l'ip 192.168.76.1 renvoit une confirmation de reception (ACK)
La connexion peut enfin etre etablie

( plus simple a lire sur Wireshark que par TCPdump )

## III. Routage simple



| Machines | 192.168.100.0 | 192.168.200.0 | NAT |
| -------- | -------- | -------- | -------- |
| Client1  | 192.168.100.2 |               |  NO |
| Client2  |               | 192.168.200.2 |  NO |
| Router1  | 192.168.100.3 | 192.168.200.3 | YES |


On configure les différentes routes entre les cartes : 

![](https://i.imgur.com/lbwV6NB.png)

![](https://i.imgur.com/VJGRnNC.png)

On passe la 3eme VM en router : 
`firewall-cmd --add-masquerade --permanent`

`sysctl net.ipv4.ip_forward=1`
ou modification dans le fichier/etc/sysctl.conf

La connexion est bien etablie entre les 2 cartes Host-Only, et reliées au NAT du router
![](https://i.imgur.com/Ib8mJj7.png)



## IV. Autres applications et métrologie

### 1. Commandes
Nécessite libpcap et libncurses puis, 
`yum install epel-release`
`yum install iftop`

Iftop est un utilitaire qui va nous permettre de visualiser le trafic réseau selon les services. Il nous indique la bande passante de chacun d'entre eux

### 2. Cockpit

![](https://i.imgur.com/LZbmMdi.png)


`firawall-cmd --list-all` montre qu'aucun port n'est ouvert

`firewall-cmd --permanent --add-port=9090/tcp`

🌞 explorer Cockpit, plus spécifiquement ce qui est en rapport avec le réseau

Acces web par `https://192.168.100.2:9090`

Cockpit, un gestionnaire de server, ultra complet, acces aux performances actuelles, la bande passante utilisé par tel ou tel carte réseau et offre meme la main dessus par un terminal, possibilité de voir le firewall, ses services et ports, ajouter des dns.... Tellement de choses ! C'est trop bien !

### 3. Netdata

Installation : `bash <(curl -Ss https://my-netdata.io/kickstart.sh)`

Ajouter l'acces au port 19999 : `firewall-cmd --permanent --add-port=19999/tcp`

Netdata permet d'afficher les differentes configurations de plusieurs VM sur une seule interface
