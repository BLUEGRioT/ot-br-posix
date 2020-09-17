# Openthread Border Router configuration
Vous trouverez ici les étapes nécessaire à l'installation et la configuration du border router openthread.

## Flash RCP
Flasher la shield RPI border router avec le soft [ot-rcp.hex](https://github.com/BLUEGRioT/ot-br-posix/blob/Project/AuditKit/) qui à été généré via la lib openthread.

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
Installer git :
```bash
sudo apt-get install git
```

```bash
```

## Installation de ot-br-posix
1. Cloner le repositorie
```bash
git clone https://github.com/openthread/ot-br-posix
```
2. Installer les dépendances
```bash
cd ot-br-posix
./script/bootstrap
```
3. Compilation et installation d'otbr posix
```bash
export OTBR_OPTIONS="-DOT_POSIX_CONFIG_RCP_BUS=SPI"
NETWORK_MANAGER=0
./script/setup
```

4. Configuration du port RCP otbr-agent
Ouvrir le fichier de configuration:
```bash
sudo nano /etc/default/otbr-agent
```
Ajouter la ligne suivante:
```bash
OTBR_AGENT_OPTS="-I wpan0 spinel+spi:///dev/spidev0.1?gpio-int-device=/dev/gpiochip0&gpio-int-line=21&gpio-reset-device=/dev/gpiochip0&gpio-reset-line=05&no-reset=0&spi-speed=1000000"
``` 
5. Redémarrer le Raspberry pi et vérifier que tout est bien configuré avec :
```bash
sudo systemctl status
```
Vérifier que les services suivants fonctionnent correctement: 
- avahi-daemon.service
- otbr-agent.service
- otbr-web.service

6. Vérifier le bon fonctionnement du RCP
```bash
sudo ot-ctl state
> Disable
```

## Configuration de Tayga
1. Installation de tayga (Deja installé ???)
```bash
sudo apt-get install tayga
``` 
2. Configuration de Tayga
```bash
sudo nano /etc/tayga.conf
``` 
 Définir les paramètres suivants:

```bash
tun-device nat64

ipv4-addr 172.16.0.1

ipv6-addr fdaa:bb:1::1

prefix 2001:db8:1:ffff::/96

dynamic-pool 172.16.0.0/21

data-dir /var/spool/tayga
``` 

3. Activation de tayga
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

4. Activer le service tayga:

```bash
sudo systemctl enable tayga.service
``` 

5. Activer la redirection IPV4 et IPV6

```bash
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
sudo sh -c "echo 1 > /proc/sys/net/ipv6/conf/all/forwarding"
```
Configurer pour que cela reste actif au redémarrage
- Ouvrir le fichier /etc/sysctl.conf
- Decommenter la ligne suivante et configurer à 1:

```bash
net.ipv4.ip_forward=1
```
Configuration NAT44
```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```
Ajouter les lignes suivantes pour que cela reste actif après un redémarrage:
```bash
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```
Ouvrir le fichier /etc/rc.local
```bash
sudo nano /etc/rc.local
``` 
Et ajouter la ligne suivante avant la ligne exit 0:
```bash
iptables-restore < /etc/iptables.ipv4.nat
```

## Ajout service Border router
1. Copier le fichier /otbr-bg/border_router/border_router.conf dans le dossier /etc/

2. Copier le fichier /otbr-bg/border_router/border_router dans le dossier /usr/sbin/

3. Rendre le fichier exécutable:
```bash
sudo chmod a+x /usr/sbin/border_router
```
4. Copier le fichier /otbr-bg/border_router/border_router.service dans le dossier /etc/systemd/system/
5. Actualiser la configuration du manager systemd:
```bash
sudo systemctl daemon-reload
```
6. Activer le service:

```bash
sudo systemctl enable border_router.service
``` 
## Configurer l'ajout automatique d'une route 
1. Remplacer le fichier /usr/sbin/ncp_state_notifier avec /otbr-bg/ncp_state_notifier/ncp_state_notifier/

2. Copier le fichier /otbr-bg/ncp_state_notifier/10-gateway_configurator /etc/ncp_state_notifier/dispatcher.d/

3. Rendre le script exécutable:

```bash
sudo chmod a+x /etc/ncp_state_notifier/dispatcher.d/10-gateway_configurator
```

4. Supprimer le fichier dhcpcd_reloader
```bash
sudo rm /etc/ncp_state_notifier/dispatcher.d/dhcpcd_reloader
```
5. Redémarrer le border router

6. Vérifier la configuration du réseaux openthread via ot-ctl:

```bash
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
## Configuration de reverse ssh
## Configuration du watchdog
