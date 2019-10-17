# TP2 : Network low-level, Switching

## I. Simplest setup

**Topologie :**
```
+-----+        +-------+        +-----+
| PC1 +--------+  SW1  +--------+ PC2 |
+-----+        +-------+        +-----+
```

**Plan d'adressage :**


| Machines | net1 |
| -------- | -------- |
| PC1      | 10.2.1.1/24 |
| PC2      | 10.2.1.2/24 |


La machine 1 peut bien ping la machine 2
```
PC1> ping 10.2.1.2
84 bytes from 10.2.1.2 icmp_seq=1 ttl=64 time=0.227 ms
```

La machine 1 va des lors envoyer une requete ARP au Broadcast pour demander à qui appartient l'ip pingé en signant par la sienne

```
9.474239	Private_66:68:00	Broadcast	ARP	64	Who has 10.2.1.2? Tell 10.2.1.1
```

La machine 2 ayant recu la requete ARP, lui répond à son tour
```
9.475016	Private_66:68:01	Private_66:68:00	ARP	64	10.2.1.2 is at 00:50:79:66:68:01
```

Les deux machines ecrivent ces nouvelles informations dans la table ARP
```
PC1> show arp
00:50:79:66:68:01  10.2.1.2 expires in 118 seconds

PC2> show arp
00:50:79:66:68:00  10.2.1.1 expires in 118 seconds
```

Ils peuvent donc enfin se ping avec le protocole ICMP (capture Wireshark réalisé sur le réseau "Switch/PC1")
```
PC1>PC2
9.475209	10.2.1.1	10.2.1.2	ICMP	98	Echo (ping) request  id=0x875c, seq=1/256, ttl=64 (reply in 10)
PC2>PC1
9.475362	10.2.1.2	10.2.1.1	ICMP	98	Echo (ping) reply    id=0x875c, seq=1/256, ttl=64 (request in 9)
```

Le switch, "pont multi-port" agissant sur la couche 2 du modele OSI, n'a pas d'IP (couche internet neessitant une signature IP commencant couche 3)

Les machines ont besoin d'une IP pour se reconnaitre et pouvoir communiquer (nécessitant justement les différentes couches >3)
Elles ont aussi une Addr MAC unique à chacune pour les différencier plus précisément qu'une IP attribuable donc modifiable

## II. More switches

**Topologie :**

```
                        +-----+
                        | PC2 |
                        +--+--+
                           |
                           |
                       +---+---+
                   +---+  SW2  +----+
                   |   +-------+    |
                   |                |
                   |                |
+-----+        +---+---+        +---+---+        +-----+
| PC1 +--------+  SW1  +--------+  SW3  +--------+ PC3 |
+-----+        +-------+        +-------+        +-----+


```

**Plan d'adressage :**


| Machines | net1 |
| -------- | -------- |
| PC1      | 10.2.2.1/24 |
| PC2      | 10.2.2.2/24 |
| PC3      | 10.2.2.3/24 |


**🌞 faire communiquer les trois PCs:**

```
PC1> ping 10.2.2.2
84 bytes from 10.2.2.2 icmp_seq=1 ttl=64 time=0.324 ms
PC1> ping 10.2.2.3
84 bytes from 10.2.2.3 icmp_seq=1 ttl=64 time=0.429 ms

PC2> ping 10.2.2.3
84 bytes from 10.2.2.3 icmp_seq=1 ttl=64 time=0.511 ms
```

**🌞 analyser la table MAC d'un switch:**

```
IOU1#show mac address-table
          Mac Address Table
-------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
   1    0050.7966.6800    DYNAMIC     Et0/0 |
   1    0050.7966.6801    DYNAMIC     Et0/2 |--- Switch1
   1    0050.7966.6802    DYNAMIC     Et0/1 |
   1    aabb.cc00.0200    DYNAMIC     Et0/0  --- Machine1
   1    aabb.cc00.0300    DYNAMIC     Et0/0  --- Switch2
   1    aabb.cc00.0310    DYNAMIC     Et0/1  --- Switch3
```
Chaque ligne représente la mac des différents points de connexions du switch1, les 3 premieres sont celles le concernant directement, ses ports, les 3 autres sont les 3 autres connexions aux machines


**🐙 en lançant Wireshark sur les liens des switches, il y a des trames CDP qui circulent. Quoi qu'est-ce ?**


```
15.986495	aa:bb:cc:00:02:00	CDP/VTP/DTP/PAgP/UDLD	CDP	456	Device ID: IOU2  Port ID: Ethernet0/0
```

CDP ( pour Cisco Discovery Protocol ): protocole de message en dessous des couches réseaux (couche de liaison, la 2ieme) permettant la transmission d'information, de configuration, entre les switchs (qui se situent à la couche 2)

**Mise en évidence du Spanning Tree Protocol**

```
IOU1#show spanning-tree
Root ID    Priority    32769
             Address     aabb.cc00.0100
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
```
le Switch1 est actuellement le root-bridge


Switch1 (Root-Bridge)
```
Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Et0/0               Desg FWD 100       128.1    Shr
```

Switch2 (donne bien priorité au root donc Switch1)
```
Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Et0/0               Root FWD 100       128.1    Shr
Et0/1               Desg FWD 100       128.2    Shr
```

Switch3 (bloque un de ses ports en priorité pour eviter le boadcast storm puis le root avant le reste || aussi visible avec la commande `show spanning-tree summary`)
```
Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Et0/0               Altn BLK 100       128.1    Shr
Et0/1               Root FWD 100       128.2    Shr
```

**SW1 << : > SW2
SW2 < : > SW3
SW1 << : >/ SW3**

**<< : priorité || < : au plus rapide || </ : BID le plus haut, passe l'etat de son port en BLK (Blocking) pour eviter la boucle**



Ping bloqué entre PC3 et PC2 (plus précisément sur le port PC3 vers PC2), il passe par PC1 pour arriver à PC2:

Tentative de connexion en vain quand ecout entre PC2 et 3
```
8.909600	Private_66:68:02	Broadcast	ARP	64	Who has 10.2.2.2? Tell 10.2.2.3
...
...
8.909621	Private_66:68:02	Broadcast	ARP	64	Who has 10.2.2.2? Tell 10.2.2.3
```

Connexion du ping entre PC3 et PC2 quand ecout entre PC1 et PC2
```
7.960238	Private_66:68:02	Broadcast	ARP	64	Who has 10.2.2.2? Tell 10.2.2.3
7.962405	Private_66:68:00	Private_66:68:02	ARP	64	10.2.2.2 is at 00:50:79:66:68:00
7.964092	10.2.2.3	10.2.2.2	ICMP	98	Echo (ping) request  id=0xdb93, seq=1/256, ttl=64 (reply in 18)
7.964293	10.2.2.2	10.2.2.3	ICMP	98	Echo (ping) reply    id=0xdb93, seq=1/256, ttl=64 (request in 17)
```

**Reconfigurer STP**

```
IOU2#show spanning-tree

VLAN0001
  Spanning tree enabled protocol rstp
  Root ID    Priority    32769
             Address     aabb.cc00.0100
             Cost        100
             Port        1 (Ethernet0/0)
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
```
Le Root Bridge est celui qui a l'BID le plus bas, dans ce cas la l'ID 32769 appartient au Bridge du Switch1


On passe le Switch2 avec un BID de 4096 (etant le minimum donc le premier root possible)
```
IOU2(config)#spanning-tree vlan 1 priority 4096
```

```
IOU2#show spanning-tree

VLAN0001
  Spanning tree enabled protocol rstp
  Root ID    Priority    4097
             Address     aabb.cc00.0200
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
```
Le Switch2 est bien passé Root Bridge (ayant comme Bridge ID 4097 )


**🐙 capturer les échanges qui suivent une reconfiguration STP avec Wireshark**

Capture de ping PC2 vers PC3
```
->Spanning Tree Protocol 
   -Root Identifier: 4096 / 1 / aa:bb:cc:00:02:00
```
Puis : 
```
->Spanning Tree Protocol 
   -Root Identifier: 8192 / 1 / aa:bb:cc:00:02:00
```


### 🐙 STP & Perfs

Avant : 
Un message CDP est envoyé du Switch2 au PC2 lors d'un ping du PC2 vers les autres Machines
```
3.582675	aa:bb:cc:00:02:20	CDP/VTP/DTP/PAgP/UDLD	CDP	456	Device ID: IOU2  Port ID: Ethernet0/2  
```

Apres : 
Le messge CDP n'est plus la, le port est desactivé sur le Switch2, il refuse d'envoyer et de recevoir des CDP
```
0.000000	Private_66:68:00	Broadcast	ARP	64	Who has 10.2.2.3? Tell 10.2.2.2
0.002338	Private_66:68:02	Private_66:68:00	ARP	64	10.2.2.3 is at 00:50:79:66:68:02
0.003050	10.2.2.2	10.2.2.3	ICMP	98	Echo (ping) request  id=0xc3a0, seq=1/256, ttl=64 (reply in 7)
0.003360	10.2.2.3	10.2.2.2	ICMP	98	Echo (ping) reply    id=0xc3a0, seq=1/256, ttl=64 (request in 6)
```

## III. Isolation

### 1. Simple

**Topologie :**

```
+-----+        +-------+        +-----+
| PC1 +--------+  SW1  +--------+ PC3 |
+-----+      10+-------+20      +-----+
                 20|
                   |
                +--+--+
                | PC2 |
                +-----+
```

**Plan d'adressage :**



| Machine | IP net1 | VLAN |
| -------- | -------- | -------- |
| PC1 | 10.2.3.1/24 | 10 |
| PC2 | 10.2.3.2/24 | 20 |
| PC3 | 10.2.3.3/24 | 20 |



**Mise en place :**
```
IOU1(config)#vlan 10
IOU1(config-vlan)#name PC_vlan10
IOU1(config)#vlan 20
IOU1(config-vlan)#name PC_vlan20

IOU1(config)#interface Ethernet 0/0
IOU1(config-if)#switchport mode access
IOU1(config-if)#switchport access vlan 10

IOU1(config)#interface Ethernet 0/1
IOU1(config-if)#switchport mode access
IOU1(config-if)#switchport access vlan 20

IOU1(config)#interface Ethernet 0/2
IOU1(config-if)#switchport mode access
IOU1(config-if)#switchport access vlan 20
```


PC2 peut bien ping PC3
```
PC2> ping 10.2.3.3
84 bytes from 10.2.3.3 icmp_seq=1 ttl=64 time=0.199 ms
```

Alors que PC1, sur un vlan différent de PC2 et 3 ne peut pas les ping
```
PC1> ping 10.2.3.2
host (10.2.3.2) not reachable
PC1> ping 10.2.3.3
host (10.2.3.3) not reachable
```

### 2. Avec trunk

```
+-----+        +-------+        +-------+        +-----+
| PC1 +--------+  SW1  +--------+  SW2  +--------+ PC4 |
+-----+      10+-------+        +-------+20      +-----+
                 20|              10|
                   |                |
                +--+--+          +--+--+
                | PC2 |          | PC3 |
                +-----+          +-----+
```

**Plan d'adressage :**



| Machine | IP net1 | IP net2 | VLAN |
| -------- | -------- | -------- | -------- |
| PC1 | 10.2.10.1/24 | X | 10 |
| PC2 | X | 10.2.20.1/24 | 20 |
| PC3 | 10.2.10.2/24 | X | 10 |
| PC4 | X | 10.2.20.2/24 | 20 |


A effectuer pour les 2 Switchs
```
IOU1(config)#vlan 10
IOU1(config-vlan)#name PC_vlan10
IOU1(config)#vlan 20
IOU1(config-vlan)#name PC_vlan20
IOU1(config)#interface Ethernet 0/1
IOU1(config-if)#switchport mode access
IOU1(config-if)#switchport access vlan 10
IOU1(config)#interface Ethernet 0/2
IOU1(config-if)#switchport mode access
IOU1(config-if)#switchport access vlan 20
IOU1(config)#interface Ethernet 0/0
IOU1(config-if)#switchport trunk encapsulation dot1q
IOU1(config-if)#switchport mode trunk
IOU1(config-if)#switchport trunk allowed vlan 10,20
```


Le PC1 ne peut ping que le PC3
```
PC1> ping 10.2.10.2
84 bytes from 10.2.10.2 icmp_seq=1 ttl=64 time=0.491 ms
PC1> ping 10.2.20.x
No gateway found
```

Le PC4 ne peut ping que le PC2
```
PC4> ping 10.2.20.1
84 bytes from 10.2.20.1 icmp_seq=1 ttl=64 time=0.345 ms
PC4> ping 10.2.10.x
No gateway found
```

Le ping passe bien entre le PC1 et le PC3
```
46.791110	Private_66:68:02	Broadcast	ARP	68	Who has 10.2.10.2? Tell 10.2.10.1
46.791557	Private_66:68:01	Private_66:68:02	ARP	68	10.2.10.2 is at 00:50:79:66:68:01
46.791998	10.2.10.1	10.2.10.2	ICMP	102	Echo (ping) request  id=0x10b3, seq=1/256, ttl=64 (reply in 57)
46.792166	10.2.10.2	10.2.10.1	ICMP	102	Echo (ping) reply    id=0x10b3, seq=1/256, ttl=64 (request in 56)
```
Mais ne passe pas entre le PC 1 et le PC 4, la seule frame qu'on retrouve c'est ce message CDP entre les 2 Switchs
```
23.695805	aa:bb:cc:00:01:00	CDP/VTP/DTP/PAgP/UDLD	CDP	456	Device ID: IOU1  Port ID: Ethernet0/0  
```

### IV. Need perfs

**Topologie :**

```
+-----+        +-------+--------+-------+        +-----+
| PC1 +--------+  SW1  |        |  SW2  +--------+ PC4 |
+-----+      10+-------+--------+-------+20      +-----+
                 20|              10|
                   |                |
                +--+--+          +--+--+
                | PC2 |          | PC3 |
                +-----+          +-----+
```

**Plan d'adressage**



| Machine | IP net1 | IP net2 | VLAN |
| -------- | -------- | -------- | -------- |
| PC1 | 10.2.10.1/24 | X | 10 |
| PC2 | X | 10.2.20.1/24 | 20 |
| PC3 | 10.2.10.2/24 | X | 10 |
| PC4 | X | 10.2.20.2/24 | 20 |


Config LACP sur les Switchs
```
IOU1(config)#interface range Ethernet 0/0-1
IOU1(config-if)#channel-group 1 mode active  ##mode passif pour le Switch2##
IOU1(config)#interface port-channel 1
IOU1(config-if)#switchport trunk encapsulation dot1q
IOU1(config-if)#switchport mode trunk
```


Le ping passe entre PC1 et PC2 
```
PC1> ping 10.2.10.2
84 bytes from 10.2.10.2 icmp_seq=1 ttl=64 time=0.324 ms
```

Bonne nuit