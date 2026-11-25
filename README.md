# TP – OpenVPN sur pfSense

<aside>

**Objectif** : permettre à un poste situé à l'extérieur (côté WAN) de se connecter au LAN de l'entreprise à travers un tunnel VPN chiffré, avec OpenVPN installé sur pfSense.

</aside>

## Contexte et adressage

On reprend l'infrastructure du **TP PfSense / DMZ** (même VM pfSense, mêmes IP).

- pfSense WAN (vtnet0, vmbr0) : 192.168.20.174/24 en DHCP
- pfSense LAN (vtnet1, vmbr1) : 192.168.10.5/24
- pfSense DMZ / OPT1 (vtnet2, vmbr2) : 192.168.30.254/24
- Client Windows 11 du LAN : 192.168.10.10/24, passerelle 192.168.10.5
- Réseau du tunnel VPN : 10.0.8.0/24 (réseau choisi pour le VPN, il ne doit pas exister ailleurs)
- Client VPN : une machine côté WAN (réseau 192.168.20.0/24), par exemple une VM avec sa carte sur vmbr0

<img width="695" height="238" alt="image-0" src="https://github.com/user-attachments/assets/fb1b49b4-3725-43a8-adc7-2f0083c68349" />


## 1. Créer l'autorité de certification (CA) 

**Où** : System > Certificates (ou Cert. Manager selon la version) > onglet Authorities > Add

- Descriptive name : `CA-OpenVPN` (nom affiché dans pfSense)
- Method : Create an internal Certificate Authority (pfSense crée sa propre autorité)
- Common Name : `CA-OpenVPN` (nom inscrit dans le certificat)
- Le reste par défaut > Save

**Pourquoi** : OpenVPN authentifie le serveur et les clients avec des certificats. La CA signe ces certificats : comme le serveur et le client sont signés par la même CA, ils se font confiance.

formulaire de création de la CA

<img width="858" height="169" alt="image-1" src="https://github.com/user-attachments/assets/e3ea5b97-62b8-463a-ad59-f0c17709d676" />


 CA-OpenVPN dans la liste des Authorities

<img width="812" height="123" alt="image-2" src="https://github.com/user-attachments/assets/c840bd5d-3907-479b-bbea-d30ba7164296" />


## 2. Créer le certificat du serveur 

**Où** : System > Certificates > onglet Certificates > Add/Sign

- Method : Create an internal Certificate
- Descriptive name : `Cert-Serveur-OpenVPN`
- Certificate authority : `CA-OpenVPN` (c'est elle qui signe ce certificat)
- Common Name : `Cert-Serveur-OpenVPN`
- Certificate Type : **Server Certificate** (indique que ce certificat sert à identifier le serveur VPN)
- Le reste par défaut > Save

**Pourquoi** : c'est la « carte d'identité » du serveur VPN. Le client vérifie ce certificat pour être sûr de se connecter au bon serveur.

**Résultat** : le certificat `Cert-Serveur-OpenVPN` apparaît dans la liste des Certificates avec le type Server Certificate (Server : Yes) et l'émetteur (Issuer) `CA-Openvpn`, la CA créée à l'étape 1. Il est valable 10 ans.

<img width="1145" height="257" alt="image-3" src="https://github.com/user-attachments/assets/0afb1e2a-5515-4251-ad75-74ca12df6576" />


## 3. Créer l'utilisateur VPN et son certificat 

**Où** : System > User Manager > onglet Users > Add

- Username : `uservpn`
- Password : un mot de passe (à retenir, il sera demandé à la connexion VPN)
- Cocher **Click to create a user certificate**
    - Descriptive name : `Cert-uservpn`
    - Certificate authority : `CA-OpenVPN`
- Save

**Pourquoi** : on a choisi un VPN avec double authentification : le client doit avoir le certificat **et** connaître l'identifiant / mot de passe. Si l'un des deux est volé, ça ne suffit pas.

création de l'utilisateur uservpn avec la case certificat cochée

<img width="884" height="41" alt="image-4" src="https://github.com/user-attachments/assets/c0e4508a-2342-4c39-b585-2794f9ad0bde" />


<img width="846" height="181" alt="image-5" src="https://github.com/user-attachments/assets/2e26f768-a8ad-4acd-8ea3-75b1ac77daa3" />


## 4. Installer le paquet d'export client 

**Où** : System > Package Manager > onglet Available Packages

- Rechercher `openvpn-client-export` > Install > Confirm
- Attendre le message de succès

**Pourquoi** : ce paquet génère automatiquement un installeur OpenVPN déjà configuré (certificats + adresse du serveur) pour le client. Sans lui, il faudrait copier les fichiers à la main.

<aside>


pfSense doit avoir accès à Internet par son WAN 

</aside>

 openvpn-client-export installé

<img width="888" height="95" alt="image-6" src="https://github.com/user-attachments/assets/25178d9c-dfa3-4f04-9794-74ec27382b69" />


<img width="533" height="57" alt="image-7" src="https://github.com/user-attachments/assets/0e53f74d-0a8e-4d9e-97c9-22c260a1c670" />


## 5. Créer le serveur OpenVPN 

**Où** : VPN > OpenVPN > onglet Servers > Add

- Server mode : **Remote Access (SSL/TLS + User Auth)** (certificat + identifiant/mot de passe, comme expliqué à l'étape 3)
- Backend for authentication : Local Database (les utilisateurs créés dans User Manager)
- Device mode : tun (le VPN transporte des paquets IP, mode routé)
- Protocol : UDP on IPv4 only (UDP est le protocole par défaut d'OpenVPN, plus rapide)
- Interface : WAN (le serveur écoute du côté extérieur)
- Local port : 1194 (port standard d'OpenVPN)
- TLS Configuration : laisser Use a TLS Key coché (clé générée automatiquement, protège la négociation)
- Peer Certificate Authority : `CA-OpenVPN`
- Server certificate : `Cert-Serveur-OpenVPN`
- IPv4 Tunnel Network : `10.0.8.0/24` (adresses données aux clients dans le tunnel)
- IPv4 Local network(s) : `192.168.10.0/24` (réseau que le client VPN pourra joindre : le LAN)
- Le reste par défaut > Save

**Pourquoi** : c'est le service qui attend les connexions des clients VPN et leur donne accès au LAN.

configuration du serveur (mode, interface, port)

<img width="804" height="219" alt="image-8" src="https://github.com/user-attachments/assets/e0b4e848-d43d-4718-a563-64c4eaea922a" />


partie certificats et réseaux tunnel/local

<img width="867" height="554" alt="image-9" src="https://github.com/user-attachments/assets/57fd818f-e9e7-4f6d-bf9c-d658b111fe4c" />


le serveur dans la liste VPN > OpenVPN > Servers

<img width="1135" height="153" alt="image-10" src="https://github.com/user-attachments/assets/20eac073-c4a6-4230-bfc8-8f71f970dfd0" />


## 6. Créer les règles de pare-feu

### 6.a Autoriser OpenVPN sur le WAN 

**Où** : Firewall > Rules > onglet WAN > Add

- Action : Pass
- Protocol : UDP
- Source : any
- Destination : WAN address
- Destination Port Range : OpenVPN (1194)
- Save > Apply Changes

**Pourquoi** : par défaut le WAN bloque tout ce qui entre. Cette règle laisse passer uniquement les connexions VPN vers pfSense.

règle WAN UDP 1194 :

<img width="1132" height="165" alt="image-11" src="https://github.com/user-attachments/assets/4ec9150a-467a-4cac-a7db-90fec66d7653" />


<img width="995" height="36" alt="image-12" src="https://github.com/user-attachments/assets/fb4aa9d3-753f-4bde-832e-8640a3edeef2" />


### 6.b Autoriser le trafic dans le tunnel 

**Où** : Firewall > Rules > onglet OpenVPN > Add

- Action : Pass
- Protocol : Any
- Source : any
- Destination : LAN subnets
- Save > Apply Changes
<img width="1152" height="160" alt="image-13" src="https://github.com/user-attachments/assets/fa0d7008-8634-4d90-b4d4-0d73a662c468" />


**Pourquoi** : une fois connecté, le trafic du client arrive par l'interface OpenVPN. Sans règle, il serait bloqué et le client ne pourrait pas joindre le LAN.

### 6.c Ne plus bloquer les réseaux privés sur le WAN 

**Où** : Interfaces > WAN

- Décocher **Block private networks and loopback addresses**
- Save > Apply Changes

<img width="1144" height="139" alt="image-14" src="https://github.com/user-attachments/assets/b33d4af1-2e02-4d2a-84cc-a989f1364e9d" />


**Pourquoi** : notre « extérieur » est le réseau de l'école 192.168.20.0/24, qui est un réseau privé. Avec cette option cochée, pfSense bloquerait le client VPN avant même de lire nos règles.

### 7.a Installer OpenVPN sur le client (côté WAN) 

- Copier l'installeur sur la machine côté WAN et l'exécuter
- Lancer OpenVPN GUI > clic droit sur l'icône près de l'horloge > Connect
- Saisir `uservpn` et son mot de passe

fenêtre de connexion OpenVPN

<img width="578" height="372" alt="image-15" src="https://github.com/user-attachments/assets/3dfbbe6d-4a71-4891-9449-cea8fbb1c6e4" />


icône OpenVPN verte = connecté

<img width="282" height="115" alt="image-16" src="https://github.com/user-attachments/assets/ed6dedb7-e66b-4e92-879b-edbad94b8af1" />


### 7.b Tester la connexion ✅

- Sur le client : `ipconfig` → une carte OpenVPN avec une IP en 10.0.8.x doit apparaître
- Sur le client : `ping 192.168.10.5` → pfSense répond par le LAN à travers le tunnel
- Sur le client : ouvrir `https://192.168.10.5` → la page de connexion pfSense s'affiche
- Sur pfSense : Status > OpenVPN → `uservpn` apparaît comme connecté

<aside>
💡

Un ping vers le Windows 11 du LAN (192.168.10.10) peut échouer même si le VPN marche : le pare-feu Windows bloque les pings entrants par défaut. C'est pour ça qu'on teste vers pfSense.

</aside>

 ipconfig avec l'IP 10.0.8.x

<img width="720" height="144" alt="image-17" src="https://github.com/user-attachments/assets/77c4c465-4a0a-4fd0-a3c7-701a576ec60b" />


ping 192.168.10.5 

<img width="670" height="230" alt="image-18" src="https://github.com/user-attachments/assets/650ea4c7-14f5-4369-af56-dfc964707b37" />


Status > OpenVPN avec le client connecté

<img width="1141" height="143" alt="image-19" src="https://github.com/user-attachments/assets/fd88ad26-da9f-4e5a-852c-294ab7ad3b8c" />


**Résultat** : dans Status > OpenVPN, `uservpn` est connecté avec la Real Address **192.168.20.148** (le client vient bien du réseau de l'école, donc de l'extérieur, côté WAN), la Virtual Address **10.0.8.2** (son IP dans le tunnel) et le chiffrement **AES-256-GCM**. Le client accède au LAN (192.168.10.5) à travers le tunnel.

## Conclusion

Le serveur OpenVPN installé sur pfSense fonctionne : un poste situé à l'extérieur (réseau 192.168.20.0/24, côté WAN) se connecte au LAN de l'entreprise à travers un tunnel chiffré (AES-256-GCM). L'accès est sécurisé par une double authentification : le certificat de l'utilisateur, signé par notre autorité de certification, et son identifiant / mot de passe. Côté pare-feu, seul le port UDP 1194 est ouvert sur le WAN, et les clients VPN n'ont accès qu'au LAN.

---

<aside>

Ressource : [vidéo YouTube fournie](https://youtu.be/QDaRpaIH4GE)

</aside>
