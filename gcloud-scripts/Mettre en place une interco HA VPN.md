# Mettre en place une interconnexion avec un VPN HA

## Présentation

Un VPN haute disponibilité est une solution Cloud VPN offrant une disponibilité élevée. Ce type de VPN vous permet de connecter en toute sécurité votre réseau local à votre réseau de cloud privé virtuel (VPC) via une connexion VPN IPsec située dans une seule région. Les VPN haute disponibilité garantissent un taux de disponibilité de 99,99 %.

Le VPN haute disponibilité est une solution VPN régionale et par VPC. Les passerelles VPN haute disponibilité possèdent deux interfaces, chacune ayant sa propre adresse IP publique. Lorsque vous créez une passerelle VPN haute disponibilité, deux adresses IP publiques sont automatiquement sélectionnées dans différents pools d'adresses. Lorsque le VPN haute disponibilité est configuré avec deux tunnels, Cloud VPN offre un taux de disponibilité de 99,99 %.

Dans cet atelier, vous allez créer un VPC mondial nommé vpc-demo, avec deux sous-réseaux personnalisés dans us-east1 et us-central1. Dans ce VPC, vous ajouterez une instance Compute Engine dans chaque région. Vous créerez ensuite un second VPC nommé on-prem pour simuler le centre de données sur site d'un client. Dans ce second VPC, vous ajouterez un sous-réseau dans la région us-central1 et une instance Compute Engine exécutée dans cette région. Enfin, vous ajouterez un VPN haute disponibilité et un routeur cloud dans chaque VPC, puis exécuterez deux tunnels à partir de chaque passerelle VPN haute disponibilité avant de tester la configuration pour vérifier que le contrat de niveau de service atteint 99,99 %.

![alt ha-vpn](./img/HA-VPN-schema.png)

## Objectifs

Dans cet atelier, vous allez apprendre à effectuer les tâches suivantes :

- Créer deux instances et réseaux VPC
- Configurer des passerelles VPN haute disponibilité
- Configurer le routage dynamique avec les tunnels VPN
- Configurer le mode de routage dynamique global
- Vérifier et tester la configuration de la passerelle VPN haute disponibilité

## Mise en place d'un VPC sur Google Cloud

### setup d'un Global VPC

```
gcloud compute networks create [vpc-name] --subnet-mode custom
```

### setup du subnet

```
gcloud compute networks subnets create [vpc-name-subnet] \
--network [vpc-name] --range 10.1.1.0/24 --region [Region]

gcloud compute networks subnets create [vpc-name-subnet2] \
--network vpc-demo --range 10.2.1.0/24 --region [Region2]
```

### Créer une règle de pare-feu pour autoriser tout le trafic personnalisé au sein du réseau

```
gcloud compute firewall-rules create [name-vpc-allow-custom] \
  --network [vpc-name] \
  --allow tcp:0-65535,udp:0-65535,icmp \
  --source-ranges 10.0.0.0/8
```

### Créer une règle de pare-feu pour autoriser le trafic SSH et ICMP en provenance de n'importe où

```
gcloud compute firewall-rules create [vpc-name-allow-ssh-icmp] \
    --network vpc-demo \
    --allow tcp:22,icmp
```

### création des VMs pour les subnets

```
gcloud compute instances create [vpc-name-instance1] --zone [Zone] --subnet [vpc-name-subnet]

gcloud compute instances create [vpc-name-instance2] --zone [Zone2] --subnet [vpc-name-subnet2]
```

## Simuler un VPC network onPrem sur GCP

### Créer le VPC

```
gcloud compute networks create [name-on-prem] --subnet-mode custom
```

### Créer le subnet

```
gcloud compute networks subnets create [name-on-prem-subnet] \
--network [name-on-prem] --range 192.168.1.0/24 --region [Region]
```

### créer la règle firewall "allow all custom traffic within the network"

```
gcloud compute firewall-rules create [name-on-prem-allow-custom] \
  --network [name-on-prem] \
  --allow tcp:0-65535,udp:0-65535,icmp \
  --source-ranges 192.168.0.0/16
```

### créer la règle firewall "allow SSH, RDP, HTTP, and ICMP traffic"

```
gcloud compute firewall-rules create [name-on-prem-allow-ssh-icmp] \
    --network [on-prem] \
    --allow tcp:22,icmp
```

### Créer la VM

```
gcloud compute instances create [name-on-prem-instance1] --zone [Region] --subnet [name-on-prem-subnet]
```

## Configuration du HA VPN

### créer le HA VPN [vpc-name]

```
gcloud compute vpn-gateways create [vpc-name-vpn-gw1] --network [vpc-name] --region [Region]
```

### créer le HA VPN [name-on-prem]

```
gcloud compute vpn-gateways create [name-on-prem-vpn-gw1] --network [name-on-prem] --region [Region]
```

### Voir la description du VPN Gateway

```
gcloud compute vpn-gateways describe [vpc-name-vpn-gw1] --region [Region]
gcloud compute vpn-gateways describe [name-on-prem-vpn-gw1] --region [Region]
```

## Création du Router

### Créer le router sur [vpc-name]

```
gcloud compute routers create [vpc-name-router] \
    --region [Region] \
    --network [vpc-name] \
    --asn 65001
```

### Créer le router sur [name-on-prem]

```
gcloud compute routers create [name-on-prem-router] \
    --region [Region] \
    --network [vpc-name] \
    --asn 65002
```

## Création de 2 tunnels VPNs

### créer le premier tunnel sur [vpc-name]

```
gcloud compute vpn-tunnels create [vpc-name-tunnel0] \
    --peer-gcp-gateway [name-on-prem-vpn-gw1] \
    --region [Region] \
    --ike-version 2 \
    --shared-secret [SHARED_SECRET] \
    --router [vpc-name-router] \
    --vpn-gateway [vpc-name-vpn-gw1] \
    --interface 0
```

### créer le second tunnel VPN sur [vpc-name]

```
gcloud compute vpn-tunnels create [vpc-name-tunnel1] \
    --peer-gcp-gateway [name-on-prem-vpn-gw1] \
    --region [Region] \
    --ike-version 2 \
    --shared-secret [SHARED_SECRET] \
    --router [vpc-name-router] \
    --vpn-gateway [vpc-name-vpn-gw1] \
    --interface 1
```

### créer le premier tunnel sur [name-on-prem]

```
gcloud compute vpn-tunnels create [name-on-prem-tunnel0] \
    --peer-gcp-gateway [vpc-name-vpn-gw1] \
    --region [Region] \
    --ike-version 2 \
    --shared-secret [SHARED_SECRET] \
    --router [name-on-prem-router] \
    --vpn-gateway [name-on-prem-vpn-gw1] \
    --interface 0
```

### créer le second tunnel VPN sur [name-on-prem]

```
gcloud compute vpn-tunnels create [name-on-prem-tunnel1] \
    --peer-gcp-gateway [vpc-name-vpn-gw1] \
    --region [Region] \
    --ike-version 2 \
    --shared-secret [SHARED_SECRET] \
    --router [name-on-prem-router] \
    --vpn-gateway [name-on-prem-vpn-gw1] \
    --interface 1    
```

## Créer un peering BGP (Border Gateway Protocol) pour chaque tunnel

### créer l'interface router pour le tunnel0 dans le [vpc-name]

```
gcloud compute routers add-interface [vpc-name-router] \
    --interface-name if-tunnel0-to-on-prem \
    --ip-address 169.254.0.1 \
    --mask-length 30 \
    --vpn-tunnel [vpc-name-tunnel0] \
    --region [Region]
```

### créer le BGP peer pour tunnel0 dans le [vpc-name]

```
gcloud compute routers add-bgp-peer [vpc-name-router] \
    --peer-name bgp-on-prem-tunnel0 \
    --interface if-tunnel0-to-on-prem \
    --peer-ip-address 169.254.0.2 \
    --peer-asn 65002 \
    --region [Region]
```

### créer l'interface router pour le tunnel0 dans le [vpc-name]

```
gcloud compute routers add-interface [vpc-name-router] \
    --interface-name if-tunnel1-to-on-prem \
    --ip-address 169.254.1.1 \
    --mask-length 30 \
    --vpn-tunnel [vpc-name-tunnel1] \
    --region [Region]
```

### créer le BGP peer pour tunnel0 dans le [vpc-name]

```
gcloud compute routers add-bgp-peer [vpc-name-router] \
    --peer-name bgp-on-prem-tunnel1 \
    --interface if-tunnel1-to-on-prem \
    --peer-ip-address 169.254.1.2 \
    --peer-asn 65002 \
    --region [Region]
```

### créer l'interface router pour le tunnel0 dans le [name-on-prem]

```
gcloud compute routers add-interface [name-on-prem-router] \
    --interface-name if-tunnel0-to-vpc-demo \
    --ip-address 169.254.0.2 \
    --mask-length 30 \
    --vpn-tunnel [name-on-prem-tunnel0] \
    --region [Region]
```

### créer le BGP peer pour tunnel0 dans le [name-on-prem]

```
gcloud compute routers add-bgp-peer [name-on-prem-router] \
    --peer-name bgp-vpc-demo-tunnel0 \
    --interface if-tunnel0-to-vpc-demo \
    --peer-ip-address 169.254.0.1 \
    --peer-asn 65001 \
    --region [Region]
```

### créer l'interface router pour le tunnel1 dans le [name-on-prem]

```
gcloud compute routers add-interface [name-on-prem-router] \
    --interface-name if-tunnel1-to-vpc-demo \
    --ip-address 169.254.1.2 \
    --mask-length 30 \
    --vpn-tunnel [name-on-prem-tunnel1] \
    --region [Region]
```

### créer le BGP peer pour tunnel1 dans le [name-on-prem]

```
gcloud compute routers add-bgp-peer [name-on-prem-router] \
    --peer-name bgp-vpc-demo-tunnel1 \
    --interface if-tunnel1-to-vpc-demo \
    --peer-ip-address 169.254.1.1 \
    --peer-asn 65001 \
    --region [Region]
```    
    
## Vérification des configurations de router

```
gcloud compute routers describe [vpc-name-router] \
    --region [Region]

gcloud compute routers describe [name-on-prem-router] \
    --region [Region]
```

## Autoriser le trafic de [name-on-prem] vers [vpc-name] 

```
gcloud compute firewall-rules create [vpc-name-allow-subnets-from-on-prem] \
    --network [vpc-name] \
    --allow tcp,udp,icmp \
    --source-ranges 192.168.1.0/24
```

## Autoriser le trafic de [vpc-name] vers [name-on-prem] 

```
gcloud compute firewall-rules create [on-prem-allow-subnets-from-vpc-name] \
    --network [name-on-prem] \
    --allow tcp,udp,icmp \
    --source-ranges 10.1.1.0/24,10.2.
```

## Vérifier le statut des tunnels   

### liste des tunnels

```
gcloud compute vpn-tunnels list    
```

### Vérifier que [vpc-name-tunnel0] est Up

```
gcloud compute vpn-tunnels describe vpc-demo-tunnel0 \
      --region [Region]
```

### Vérifier que [vpc-name-tunnel1] est Up

```
gcloud compute vpn-tunnels describe vpc-demo-tunnel1 \
      --region [Region]      
```

### Vérifier que [name-on-prem-tunnel0] est Up

```
gcloud compute vpn-tunnels describe on-prem-tunnel0 \
      --region [Region]          
```

### Vérifier que [name-on-prem-tunnel1] est Up

```
gcloud compute vpn-tunnels describe on-prem-tunnel1 \
      --region [Region]
```      
      
## Vérifier la connectivité privée sur VPN

### Connectez-vous en SSH sur la VM Instance [name-on-prem-instance1]

```
gcloud compute ssh [name-on-prem-instance1] --zone [Zone]
```

Faites "y" pour confirmer de continuer

Taper Enter pour squizzer la partie password

Depuis l'instance [name-on-prem-instance1] dans le réseau [name-on-prem], pour atteindre les instances dans le réseau [vpc-name], ping 10.1.1.2:

```
ping -c 4 10.1.1.2     
```      
      
## Global routing avec VPN

HA VPN est une ressource régionale et un routeur cloud qui, par défaut, ne voit que les routes de la région dans laquelle il est déployé. Pour atteindre des instances situées dans une région différente de celle du routeur en nuage, vous devez activer le mode de routage global pour le VPC. Cela permet au routeur en nuage de voir et d'annoncer des itinéraires provenant d'autres régions.

### Mettre à jour le mode bgp-routing pour [vpc-name] à GLOBAL      

```
gcloud compute networks update [vpc-name] --bgp-routing-mode GLOBAL
```

### Vérifier que le changement est ok

```
gcloud compute networks describe [vpc-name]  
```    
      
### Connectez-vous en SSH sur la VM Instance [name-on-prem-instance1]

```
gcloud compute ssh [name-on-prem-instance1] --zone [Zone]    
```

### Ping l'instance [vpc-name-instance2] dans l'autre région [Region 2]

```
ping -c 2 10.2.1.2
```

## Tester et vérifier la configuration des tunnels HA VPN

### Supprimer le [vpc-name-tunnel0] dans la région [Region]

```
gcloud compute vpn-tunnels delete [vpc-name-tunnel0]  --region [Region]
```

Répondre "y". Le tunnel0 correspondant dans le réseau [name-on-prem] sera mis hors service.

### Vérifier que le tunnel est down

```
gcloud compute vpn-tunnels describe [name-on-prem-tunnel0]  --region [Region]
```

### Revenir sur la session SSH et tester le ping suivant

```
ping -c 3 10.1.1.2
```

Le resultat doit être en succès car le trafic sera redirigé vers le second tunnel.

## Nettoyer un projet de toutes configurations

### Supprimer les tunnels

```
gcloud compute vpn-tunnels delete [name-on-prem-tunnel0]  --region [Region]

gcloud compute vpn-tunnels delete vpc-demo-tunnel1  --region [Region]

gcloud compute vpn-tunnels delete on-prem-tunnel1  --region [Region]
```

### Supprimer BGP peering

```
gcloud compute routers remove-bgp-peer [vpc-name-router] --peer-name bgp-on-prem-tunnel0 --region [Region]
gcloud compute routers remove-bgp-peer [vpc-name-router] --peer-name bgp-on-prem-tunnel1 --region [Region]
gcloud compute routers remove-bgp-peer [name-on-prem-router] --peer-name bgp-vpc-demo-tunnel0 --region [Region]
gcloud compute routers remove-bgp-peer [name-on-prem-router] --peer-name bgp-vpc-demo-tunnel1 --region [Region]
```

### Supprimer les cloud routers

```
gcloud compute  routers delete [name-on-prem-router] --region [Region]
gcloud compute  routers delete [vpc-name-router] --region [Region]
```

### Supprimer VPN gateways

```
gcloud compute vpn-gateways delete [vpc-name-vpn-gw1] --region [Region]
gcloud compute vpn-gateways delete [name-on-prem-vpn-gw1] --region [Region]
```

### Supprimer les VM instances

```
gcloud compute instances delete [vpc-name-instance1] --zone [Zone]
gcloud compute instances delete [vpc-name-instance2] --zone [Zone2]
gcloud compute instances delete [name-on-prem-instance1] --zone [Zone]
```

### Supprimer les firewall rules

```
gcloud compute firewall-rules delete [name-vpc-allow-custom]
gcloud compute firewall-rules delete [on-prem-allow-subnets-from-vpc-name]
gcloud compute firewall-rules delete [on-prem-allow-ssh-icmp]
gcloud compute firewall-rules delete [on-prem-allow-custom]
gcloud compute firewall-rules delete [vpc-name-allow-subnets-from-on-prem]
gcloud compute firewall-rules delete [vpc-name-allow-ssh-icmp]
```

### Supprimer les subnets

```
gcloud compute networks subnets delete [vpc-name-subnet1] --region [Region]
gcloud compute networks subnets delete [vpc-name-subnet2] --region [Region2]
gcloud compute networks subnets delete [name-on-prem-subnet] --region [Region]
```

### Supprimer les VPCs

```
gcloud compute networks delete [vpc-name]
gcloud compute networks delete [name-on-prem]
```    