# potatoes
PATATATATATATATA

# Préparation des Templates

On va utiliser les ISOs suivants :

```
c2691-entservicesk9-mz.124-13b.bin
c3745-advipservicesk9-mz.124-25d.bin
```

Le `c2691` va servir à créer les Acces Switch, tant dis que le `c3745` va servir aux switchs L3, donc distriubtion et core.

## Options

Pour le `c2691` il faut :

- 128 MiB de RAM
- Interfaces :
	+ slot 0: GT96100-FE
	+ slot 1: NM-16ESW
- C'est un etherswitch

Pour le `c3745` il faut :

- 256 MiB de RAM
- Interfaces :
	+ slot 0: GT96100-FE
	+ slot 1: NM-1FE-TX
	+ slot 2: NM-1FE-TX
	+ slot 3: NM-16ESW
- C'est un etherswitch

# Configuration VLAN, Trunk Mode & CDP - Dsw & Asw

## Creation VLAN

### Dsw-01 & Dsw-02

Meme commandes sur les 2 switchs de Distribution, les 2 sont en mode serveurs.

```
enable
vlan database
vlan 99 name mgmt
vlan 200 name user-vl-200
vlan 201 name user-vl-201
vlan 220 name user-vl-220
vlan 221 name user-vl-221
vtp domain efpnet
```

### Asw-01 & Asw-02

Config switch acces en mode client VTP, même commandes sur les 2.

```
vlan database
vtp domain efpnet
vtp client
apply
exit
```

## Mode Trunk & CDP

### Dsw-01 & Dsw-02

Voici les commandes pour le Dsw-01, ce sont les mêmes pour le Dsw-02 sauf qu'il faut inverser les VLANs 200-201 avec 220-221 entre les interfaces f3/0 et f3/2.

```
enable
conf t

int f3/0

switchport mode trunk
switchport trunk vlan 99
switchport trunk vlan 200
switchport trunk vlan 201

speed 100
duplex full

no shutdown
exit

int f3/2

switchport mode trunk
switchport trunk vlan 99
switchport trunk vlan 220
switchport trunk vlan 221

speed 100
duplex full

no shutdown
exit

cdp run
exit
```

### Asw-01 & Asw-02

Ici on donne la config pour le Asw-01, c'est la même config pour le Asw-02 mais il suffit de adapter les VLAN autorisé sur les interfaces

Interfaces f1/0

```
enable
conf t

int f1/0

switchport mode trunk
switchport trunk vlan 99
switchport trunk vlan 200
switchport trunk vlan 201

speed 100
duplex full
cdp enable

no shutdown
exit
```

Interfaces f1/2

```
enable
conf t

int f1/2

switchport mode trunk
switchport trunk vlan 99
switchport trunk vlan 200
switchport trunk vlan 201

speed 100
duplex full
cdp enable

no shutdown
exit
```

# Conf Lien Etherchannel Distribution-Distribution

Ici il s'agit bien d'un lien de niveau 2 contrairement au lien Distribution-Core que nous allons voir plus loin.

Si le lien entre un Asw-x et son Dsw-x tombe mais que le Dsw-x reste en vie, ce dernier va garder la priorité du HSRP sur les VLANs sur lesquels il est prioritaire. Donc il faudra switcher le traffic des VLANs en question vers ce switch dans tous les cas. Pour ce faire, le trafic des VLANs concernés va passer par le Asw-x puis par le Dsw-Y et enfin vers le Dsw-x et ce grâce au lien qu'on va placer entre les 2 Dsw.

La config est la même pour les 2 Dsw.

```
enable
conf t

int range f3/14 - 15
speed 100
duplex full
switchport mode trunk
exit

int range f3/14 - 15
channel-group 1 mode on
exit

int port-channel 1
switchport mode trunk
switchport trunk vlan 99
switchport trunk vlan 200
switchport trunk vlan 201
switchport trunk vlan 220
switchport trunk vlan 221
exit
```

# Config des Mode Access sur les Access Switchs

## Asw-01

```
conf t
int f1/10
switchport mode access
switcport access vlan 200
spanning-tree portfast
speed 100
duplex full
```

```
conf t
int f1/12
switchport mode access
switcport access vlan 201
spanning-tree portfast
speed 100
duplex full
```

## Asw-02

```
conf t
int f1/10
switchport mode access
switcport access vlan 220
spanning-tree portfast
speed 100
duplex full
```

```
conf t
int f1/12
switchport mode access
switcport access vlan 221
spanning-tree portfast
speed 100
duplex full
```

# Configuration du DHCP

## Dsw-02

```
conf t
ip dhcp pool user-vl-200
network 192.168.200.0 255.255.255.0
default-router 192.168.200.1
dns-server 1.1.1.1
exit
ip dhcp excluded-address 192.168.200.1 192.168.200.99
```

```
conf t
ip dhcp pool user-vl-201
network 192.168.201.0 255.255.255.0
default-router 192.168.201.1
dns-server 1.1.1.1
exit
ip dhcp excluded-address 192.168.201.1 192.168.201.99
```

```
conf t
ip dhcp pool user-vl-220
network 192.168.220.0 255.255.255.0
default-router 192.168.220.1
dns-server 1.1.1.1
exit
ip dhcp excluded-address 192.168.220.1 192.168.220.99
```

```
conf t
ip dhcp pool user-vl-221
network 192.168.221.0 255.255.255.0
default-router 192.168.221.1
dns-server 1.1.1.1
exit
ip dhcp excluded-address 192.168.221.1 192.168.221.99
```

### Actier le service DHCP

```
service dhcp # starts the service
```

# Config IP Switchs Distribution

## Dsw-02

Faire pour `vlan_id = {200, 201, 220, 221}`. Attention, ici c'est l'adresse IP qui se termine par `.2`

```
enable
conf t

int vlan vlan_id
ip address 192.168.vlan_id.2 255.255.255.0
no shutdown
exit
```

## Dsw-01

Faire pour `vlan_id = {200, 201, 220, 221}`. Attention, ici c'est l'adresse IP qui se termine par `.3`

```
enable
conf t

int vlan vlan_id
ip address 192.168.vlan_id.3 255.255.255.0
no shutdown
exit
```

# Configuration du STP : Switchs Distribution et Acces

## Dsw-01

```
enable
conf t

spanning-tree vlan 99 root primary
spanning-tree vlan 200 root primary
spanning-tree vlan 201 root primary
spanning-tree vlan 220 root secondary
spanning-tree vlan 221 root secondary
```

Et on est censé avoir ça :  

```
Dsw-01#show spanning-tree summary 
Root bridge for: VLAN99, VLAN200, VLAN201.
PortFast BPDU Guard is disabled
UplinkFast is disabled
BackboneFast is disabled

Name                 Blocking Listening Learning Forwarding STP Active
-------------------- -------- --------- -------- ---------- ----------
VLAN1                0        0         6        0          6         
VLAN99               0        0         2        0          2         
VLAN200              0        0         2        0          2         
VLAN201              0        0         2        0          2         
VLAN220              0        0         2        0          2         
VLAN221              0        0         2        0          2         
-------------------- -------- --------- -------- ---------- ----------
             6 VLANs 0        0         16       0          16        
```

## Dsw-02

On fait l'inverse sur le Dsw-02

# Config du HSRP

## Dsw-01 : priority for VLANs 99 200 201

```
enable
conf t

int vlan 99
standby 99 ip 192.168.99.1 # config de l'IP virtuelle
standby 99 priority 110 # la prio la plus elevee l'emporte
standby 99 preempt # pour reprendre la main apres resurection.
exit
```

Répéter pour les vlan 200 et 201.

Pour les vlan 220 et 221 priority 105 et pas de preempt

## Dsw-02 : priority for VLANs 220 221

```
enable
conf t

int vlan 220
standby 220 ip 192.168.220.1 # config de l'IP virtuelle
standby 220 priority 110 # la prio la plus elevee l'emporte
standby 220 preempt # pour reprendre la main apres resurection.
exit
```

Répéter pour les vlan 221.

Pour les vlan 99, 200 et 201 priority 105 et pas de preempt


# Liens Switchs Core-Distribution : VLAN et Etherchannel

## Csw-01 - Dsw-01

config du vlan 18 dans le Csw-01 et Dsw-01 pour l'intercommunication :

### Dsw-01

```
vlan database
vlan 18 name lien-C1D1
```

```
enable
conf t

int range f3/6 - 8
switchprot mode access
switchpport access vlan 18
speed 100
duplex full
no shutdown
exit

int port-channel 2 # c'est le numero de l'etherchannel pour differencier avec etherchannel 1
switchport mode access
switchport access vlan 18
no shutdown
exit

int range f3/6 - 8
channel-group 2 mode on # pour faire le lien entre le port-channel et le groupe des 2 interfaces
no shutdown
exit

int vlan 18
ip address 192.168.255.6 255.255.255.248
exit
```

### Csw-01

```
vlan database
vtp transparent # pour laisser passer tous les vlan car on travaille surtout sur la couche ip
apply
vlan 18 name lien-C1D1
```

```
enable
conf t

int range f3/1 - 3
switchport mode access
switcport access vlan 18
speed 100
duplex full
no shutdown
exit

int port-channel 2
switchport acc vlan 18
no shutdown
exit

int range f3/1 - 3
channel-group 2 mode on
no shutdown
exit

int vlan 18
ip address 192.168.255.1 255.255.255.248
exit
```

## Csw-02 - Dsw-02

config du vlan 18 dans le Csw-01 et Dsw-01 pour l'intercommunication :

### Dsw-01

```
vlan database
vlan 20 name lien-C2D2 # on va utiliser le vlan 20 dans ce cas.
```

```
enable
conf t

int range f3/6 - 8
switchprot mode access
switchpport access vlan 20
speed 100
duplex full
no shutdown
exit

int port-channel 2 # le numero de l'etherchannel, on peut utiliser le meme que pour C/Dsw-01
switchport mode access
switchport access vlan 20
no shutdown
exit

int range f3/6 - 8
channel-group 2 mode on # pour faire le lien entre le port-channel et le groupe des 2 interfaces
no shutdown
exit

int vlan 20
ip address 192.168.255.9 255.255.255.248
exit
```

### Csw-01

```
vlan database
vtp transparent # pour laisser passer tous les vlan car on travaille surtout sur la couche ip
apply
vlan 20 name lien-C2D2
exit
```

```
enable
conf t

int range f3/1 - 3
switchport mode access
switcport access vlan 20
speed 100
duplx full
no shutdown
exit

int port-channel 2
switchport acc vlan 20
no shutdown
exit

int range f3/1 - 3
channel-group 2 mode on
no shutdown
exit

int vlan 20
ip address 192.168.255.14 255.25.255.248
no shutdown
exit
```

# Config du OSPF

## Csw-01 et Dsw-01

Les configurations pour les Csw-02 et Dsw-02 sont symétriques à ceux des Csw-01 et Dsw-01

### Csw-01

```
enable
conf t

ip routing # activer le routage
router ospf 1
network 192.168.255.0 0.0.0.7 area 0 # on annonce la route directement connecter

passive-interface default # pour bloquer le broadcast ospf sur toutes les interfaces
no passive-interface vlan 18 # sauf sur le vlan 18 pour qu'il puisse atteindre le Dsw-01
```

### Dsw-01

```
enable
conf t

ip routing # activer le routage
router ospf 1
network 192.168.255.0 0.0.0.7 area 0
network 192.168.200.0 0.0.0.255 area 0
network 192.168.201.0 0.0.0.255 area 0
network 192.168.220.0 0.0.0.255 area 0
network 192.168.221.0 0.0.0.255 area 0

passive-interface default

no passive-interface vlan 18
no passive-interface vlan 99
```


# Config du NAT

## Csw-01

Ajout d'une route static sur le switch core. Étant donné que le NAT va se faire sur le Dsw-01, alors l'IP virtuelle qui est `10.10.10.100` sera derrière Dw-01. et ajout de la route `172.18.0.0/24` sur l'OSPF

```
enable
conf t

ip route 10.10.10.0 255.255.255.0 192.168.255.6
end
```

On va également connecter Csw-01 au réseau `172.18.0.0/24` avec l'adresse `172.18.0.2`, puis annoncer le réseau via OSPF.

```
enable
conf t

int f0/1
ip address 172.18.0.2 255.255.255.0
no shutdown
exit

router ospf 1
network 172.18.0.0 0.0.0.255 area 0
exit
```

## Dsw-01

On va dans un premier lieu définir le réseau du VLAN 200 comme `inside`, donc une zone de confiance maximale.

```
enable
conf t

int vlan 200
ip nat inside
exit
```

Ensuite on va définir une régle qui pour chaque paquet en provenance du réseau `inside` (donc le VLAN 200), si l'adresse IP source est `192.168.200.100`, alors elle va la traduire en `10.10.10.100`.

```
enable
conf t

ip nat inside source static 192.168.200.100 10.10.10.100
exit
```

Pas besoin du block suivant car la regle inverse est automatiquement ajoutée.

```
enable
conf t

int vlan 18
ip nat outside
exit

ip nat outside destination 10.10.10.100 192.168.200.100
exit
```

Pas besoin non plus du bloc suivant, car les IP des regles de NAT c'est considéré comme directement attaché

```
ip route 10.10.10.0 255.255.255.0 192.168.200.1
```

## IP des PCs

### PC1

```
PC5> ip 192.168.200.100/24 192.168.200.1
```

### PC5

```
PC5> ip 172.18.0.55/24 172.18.0.2
Checking for duplicate address...
PC5 : 172.18.0.55 255.255.255.0 gateway 172.18.0.2
```

# Commandes pour débug et vérifier les config

```
show vlan-sw
show vtp status

show cdp neighbors

show ip interface brief
show interface status
show ethernchannel X detail

show spann summary

show ip route
show run | section ospf

Show standby brief

show ip nat translation
```

## Wtf

C'est quoi ça je sais pas ce que j'ai noté :

```
conf t
int f0/0 # les ports f0/x sont dedier au routage, on ne peut activer le dhcp que sur eux
no shutdown
ip add d
```
