# Openthread Border Router configuration
Vous trouverez ici les étapes nécessaire à l'installation et la configuration du border router openthread.

## Flash RCP
Flasher la shield RPI border router avec le soft [ot-rcp.hex](https://github.com/BLUEGRioT/ot-br-posix/blob/Project/AuditKit/otbr-bg/hex/ot-rcp.hex) qui à été généré via la lib openthread.

## Configuration du RPI
1. Télécharger l'image [Raspbian](http://downloadsraspberrypi.org/raspbian_lite/images/.raspbian_lite-2019-04-09/2019-04-08-raspbian-stretch-lite.zip). 
2. Flasher la carte SD ([linux](https://www.raspberrypi.org/documentation/installation/installing-images/linux.md), [MacOS](https://www.raspberrypi.org/documentation/installation/installing-images/mac.md), [Windows](https://www.raspberrypi.org/documentation/installation/installing-images/windows.md)).

3. Via raspi-config :
    - Changer le mot de passe
    - Activer le spi
    - Activer autoriser les connections ssh
4. Ensuite mettre à jour les packages :
```bash
sudo apt-get update
sudo apt-get upgrade
```
5. Installer git :
```bash
sudo apt-get install git
```
## Installation de ot-br-posix
### 1. Recuperer le repositorie
```bash
git clone https://github.com/BLUEGRioT/ot-br-posix.git 
```
Recupérer la branche Projet/AuditKit:

```bash
git clone https://github.com/BLUEGRioT/ot-br-posix.git 
```

### 2. Installer les dépendances
```bash
cd ot-br-posix
./script/bootstrap
```
### 3. Compilation et installation d'otbr posix
```bash
export OTBR_OPTIONS="-DOT_POSIX_CONFIG_RCP_BUS=SPI"

NETWORK_MANAGER=0

./script/setup
```
### 4. Configuration du port RCP otbr-agent
Ouvrir le fichier de configuration:
```bash
sudo nano /etc/default/otbr-agent
```
Ajouter la ligne suivante:
```bash
OTBR_AGENT_OPTS="-I wpan0 spinel+spi:///dev/spidev0.1?gpio-int-device=/dev/gpiochip0&gpio-int-line=21&gpio-reset-device=/dev/gpiochip0&gpio-reset-line=05&no-reset=0&spi-speed=1000000"
``` 
### 5. Redémarrer le Raspberry pi et vérifier que tout est bien configuré avec :
```bash
sudo systemctl status
```
Et vérifier que les services suivants fonctionnent correctement: 
- avahi-daemon.service
- otbr-agent.service
- otbr-web.service

### 6. Vérifier le bon fonctionnement du RCP
```cmd
sudo ot-ctl state
> Disable
```

## Configuration de Tayga
### 1. Installation de tayga (Deja installé ???)
```bash
sudo apt-get install tayga
``` 
### 2. Configuration de Tayga
```bash
sudo nano /etc/tayga.conf
``` 
Définir les paramètres suivants:

```cmd
tun-device nat64

ipv4-addr 172.16.0.1

ipv6-addr fdaa:bb:1::1

prefix 2001:db8:1:ffff::/96

dynamic-pool 172.16.0.0/21

data-dir /var/spool/tayga
```

### 3. Activation de tayga
```bash
sudo nano /etc/default/tayga
``` 
Configurer les paramètres suivants:
```bash
# Change this to "yes" to enable tayga
RUN="yes"
# Configure interface and set the routes up
CONFIGURE_IFACE="yes"
# Configure NAT44 for the private IPv4 range
CONFIGURE_NAT44="yes"
# Additional options that are passed to the Daemon.
DAEMON_OPTS=""
# IPv4 address to assign to the NAT64 tunnel device
IPV4_TUN_ADDR="172.16.0.2"
# IPv6 address to assign to the NAT64 tunnel device
IPV6_TUN_ADDR="fdaa:bb:1::2"
```

### 4. Activer le service tayga:

```bash
sudo systemctl enable tayga.service
``` 

### 5. Activer la redirection IPV4 et IPV6

```bash
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
sudo sh -c "echo 1 > /proc/sys/net/ipv6/conf/all/forwarding"
```
Configurer pour que cela reste actif au redémarrage:
- Ouvrir le fichier /etc/sysctl.conf
- Decommenter la ligne suivante et configurer à 1

```bash
net.ipv4.ip_forward=1
```

Configuration NAT44
```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```
Pour que cela reste actif après un redémarrage:
```bash
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```
Ouvrir le fichier **/etc/rc.local**
```bash
sudo nano /etc/rc.local
``` 
Et ajouter la ligne suivante avant la ligne exit 0:
```bash
iptables-restore < /etc/iptables.ipv4.nat
```

## Ajout service Border router
1. Copier le fichier [/otbr-bg/border_router/border_router.conf](https://github.com/BLUEGRioT/ot-br-posix/blob/Project/AuditKit/otbr-bg/script/border_router/border_router.conf) dans le dossier **/etc/**

2. Copier le fichier [/otbr-bg/border_router/border_router](https://github.com/BLUEGRioT/ot-br-posix/blob/Project/AuditKit/otbr-bg/script/border_router/border_router) dans le dossier **/usr/sbin/**

3. Rendre le fichier exécutable:
```bash
sudo chmod a+x /usr/sbin/border_router
```
4. Copier le fichier [/otbr-bg/border_router/border_router.service](https://github.com/BLUEGRioT/ot-br-posix/blob/Project/AuditKit/otbr-bg/script/border_router/border_router.service) dans le dossier **/etc/systemd/system/**

5. Actualiser la configuration du manager systemd:
```bash
sudo systemctl daemon-reload
```
6. Activer le service:

```bash
sudo systemctl enable border_router.service
``` 

## Configurer l'ajout automatique d'une route (on-mesh prefix) 
1. Remplacer le fichier **/usr/sbin/ncp_state_notifier** avec celui ci : [/otbr-bg/ncp_state_notifier/ncp_state_notifier/](https://github.com/BLUEGRioT/ot-br-posix/blob/Project/AuditKit/otbr-bg/script/ncp_state_notifier/ncp_state_notifier)

2. Copier le fichier [/otbr-bg/ncp_state_notifier/10-gateway_configurator](https://github.com/BLUEGRioT/ot-br-posix/blob/Project/AuditKit/otbr-bg/script/ncp_state_notifier/10-gateway_configurator) **/etc/ncp_state_notifier/dispatcher.d/**

3. Rendre le script exécutable:

```bash
sudo chmod a+x /etc/ncp_state_notifier/dispatcher.d/10-gateway_configurator
```

4. Supprimer le fichier dhcpcd_reloader(A voir)
```bash
sudo rm /etc/ncp_state_notifier/dispatcher.d/dhcpcd_reloader
```
5. Redémarrer le border router

6. Vérifier la configuration du réseaux openthread via ot-ctl:

```cmd
sudo ot-ctl

> state
leader
Done
> channel
20
Done
> networkname
OT_BLUEGRIoT
Done
> prefix
fdde:ad00:0:0::/64 paros med
Done
> panid
0xabcd
Done
> extpanid
dead00beef00cafe
Done
> masterkey
00112233445566778899aabbccddeeff
Done
```
 
## Configuration de la clé 4G
### 1. Configuration de la clé huawei
Si la clé 4G n'a jamais été configurer suivre les tutoriel suivant:
- [Pour démarrer la clé](https://lecrabeinfo.net/se-connecter-a-internet-dongle-cle-usb-3g-4g-sans-box-sans-wifi-en-vacances-en-france-et-etranger.html)
- [Pour configurer le service coriolis](https://www.coriolis.com/faq/wp-content/uploads/2018/10/180921-Tutoriels-Huawei-002.pdf)
Sinon passer directement à l'étape suivante.

### 2. Configuration du réseaux RPi
Pour que la route utilisée par défaut soit celle configurée par eth1 lorsque cette dernière est présente:
Ajouter dans le fichier /etc/dhcpcd.conf la configuration ci dessous:
```bash
interface eth1
iaid 2
ipv6rs
ia_na 3
ia_pd 4/::/63 wpan0/1
metric 200
```
Modifier la configuration de l'interface eth0 dans ce même fichier:

```bash
interface eth0
iaid 1
ipv6rs
ia_na 2
ia_pd 3/::/63 wpan0/1
metric 300
```
### 3. Connexion clé 4G

Connecter la clé 4G au port USB. Vérifier que la clé est bien connectée via lsusb:
```bash
Bus 001 Device 010: ID 12d1:14dc Huawei Technologies Co., Ltd. E33372 LTE/UMTS/GSM HiLink Modem/Networkcard
```
Et est bien en mode router avec l'usb id correspondant : **12d1:14dc**
 
Vous devriez voir apparaître l'interface eth1 avec comme adresse ip **192.168.8.100** (via ifconfig ou ip a show eth1):
```bash
8: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 0c:5b:8f:27:9a:64 brd ff:ff:ff:ff:ff:ff
    inet 192.168.8.100/24 brd 192.168.8.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::e5b:8fff:fe27:9a64/64 scope link
       valid_lft forever preferred_lft forever
```
Vérifier que vous avez bien accès a internet et que les paquets passe bien par l'interface eth1 avec la commande ci dessous (envoie un ping au serveur dns google (8.8.8.8) et indique la route utiliser pour y parvenir):
```bash
ip route get 8.8.8.8
8.8.8.8 via 192.168.8.1 dev eth1 src 192.168.8.100 cache
```
## Configuration de reverse ssh
Description du principe [ici](https://routeur4g.fr/discussions/discussion/1882/tuto-acceder-a-son-lan-derriere-un-routeur-4g-depuis-lexterieur-avec-ssh).
### 1. Configuration coté serveur
Sur le RPI générer une clé privé ssh (en root)
```bash
ssh-keygen -t rsa
```
Cela va générer deux fichiers: 
- /root/.ssh/id_rsa
- /root/.ssh/id_rsa.pub.

Le **contenue** du fichier **/root/.ssh/id_rsa.pub** doit être ajouté dans la liste des clés autorisées coté serveur: **/home/revssh/.ssh/authorized_keys**

### 2. Configuratin du service rev autossh

Installer autossh:
```bash
apt-get install autossh
```
Autossh permet de gérer automatiquement un tunnel ssh en cas de perte de connexion.

Copier le fichier [auto_revssh.conf](https://github.com/BLUEGRioT/ot-br-posix/blob/Project/AuditKit/otbr-bg/script/auto-revssh/auto-revssh.conf) dans etc/default/auto-revssh. 
Dans ce fichier il faudra configurer pour chaque border router un port spécifique (de 22000 à 22999), Le port utilisé ici pour cette configuration est 22000.

Copier le fichier [auto_revssh.service](https://github.com/BLUEGRioT/ot-br-posix/blob/Project/AuditKit/otbr-bg/script/auto-revssh/auto-revssh.service) dans le dossier /etc/systemd/system/auto-revssh.service.

Actualiser la configuration de systemd:
```bash
sudo systemctl daemon-reload
```
Redémarrer le border router. On peut désormais se connecter en ssh avec:
```bash
ssh -p 22000 pi@auditkit.bluegriot.com
```
## Configuration du watchdog
Installer watchdog

```bash
sudo apt install watchdog
```
Configurer le driver watchdog, ouvrir le fichier /etc/watchdog.conf. Décommenter / Modifier les lignes suivantes:

```cmd
interface = eth1
max-load-1 = 24
watchdog-device = /dev/watchdog
watchdog-timeout = 15
interval = 4
realtime = yes
priority = 1
```
Ouvrir le fichier suivant /etc/systemd/system.conf Décommenter / Modifier la ligne suivante :

```bash
 RuntimeWatchdogSec=14
``` 
Actualiser la configuration de systemd:

```bash
sudo systemctl daemon-reload
``` 

Redémarrer le service:
```bash
sudo systemctl restart watchdog.service
``` 

Ou activer si il n'est pas actif:

```bash
sudo systemctl enable watchdog.service
```