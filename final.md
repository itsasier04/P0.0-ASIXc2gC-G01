# Projecte d'Infraestructura de Xarxa Segura (OpenData BCN)

Aquest projecte té com a objectiu principal el disseny i la implementació d'una infraestructura de sistemes segura, escalable i professional per allotjar una aplicació web. Aquesta aplicació consumeix i gestiona dades obertes (Open Data) reals provinents de l'Ajuntament de Barcelona sobre equipaments educatius.

L'entorn simula una arquitectura empresarial real, prioritzant la seguretat mitjançant la segmentació de xarxes (DMZ vs Intranet), l'ús de sistemes operatius Linux (Ubuntu Server) i la configuració manual de serveis crítics d'infraestructura com l'enrutament, el tallafocs, el DHCP i el DNS.

## 1\. Arquitectura de Xarxa i Disseny

### 1.1. Esquema de la Topologia

```text
      ☁️ INTERNET (Xarxa Default / NAT)
                  |
                  v
      +--------------------------------+
      |      🛡️ ROUTER (R-N01)        |
      |   Firewall / DHCP / DNS        |
      |   IPs: 192.168.10.1 / 110.1    |
      +-------+----------------+-------+
              |                |
              |                |
   (Trànsit Filtrat)    (Trànsit Filtrat)
              |                |
              v                v
+---------------------+  +---------------------+
|  🌐 DMZ (G1a)       |  |  🏠 INTRANET (G1)   |
|  192.168.110.0/24   |  |  192.168.10.0/24    |
+---------------------+  +---------------------+
|                     |  |                     |
| [🖥️ W-N01 (Web)]    |  | [🛢️ B-N01 (BBDD)]   |
|   IP: .110.10       |  |   IP: .10.10        |
|                     |  |                     |
| [📁 F-N01 (FTP)]    |  | [🐧 Clients]        |
|   IP: .110.11       |  |   IP: DHCP (.100+)  |
|                     |  |                     |
+---------------------+  +---------------------+

       FLUXOS DE DADES PERMESOS:
       -------------------------
       1. Clients -> Web (Port 80)
       2. Clients -> FTP (Ports 20/21 + Passius)
       3. Web -> BBDD (Port 3306 - MySQL)
```

### 1.2. Justificació del Disseny

Hem implementat una arquitectura de "Defensa en Profunditat" dividida en tres zones:

  * **DMZ (Zona Desmilitaritzada):** Allotja els serveis públics (Web i FTP). Aïlla possibles atacs externs, impedint que comprometin la xarxa interna.
  * **Intranet:** Protegeix els actius crítics (Base de Dades i Clients). És inaccessible directament des d'Internet.
  * **Router Central:** Actua com a tallafocs i gestor de trànsit. L'única comunicació permesa entre zones és la estrictament necessària, garantint la màxima seguretat.

## 2\. Esquema de Necessitats Tecnològiques

| Rol / Màquina | Sistema Operatiu | Programari Clau | Justificació de l'Elecció |
| :--- | :--- | :--- | :--- |
| **R-N01 (Router)** | Ubuntu Server 22.04 | `iptables`, `isc-dhcp-server`, `bind9` | Nucli de la xarxa. La versió Server garanteix estabilitat i baix consum per gestionar l'encaminament, NAT i seguretat. |
| **B-N01 (BBDD)** | Ubuntu Server 22.04 | `mysql-server` | Servidor dedicat a dades. MySQL és l'estàndard robust per a projectes web LAMP. |
| **W-N01 (Web)** | Ubuntu Server 22.04 | `apache2`, `php` | Situat a la DMZ. Sistema lleuger i fàcil de securitzar. |
| **F-N01 (FTP)** | Ubuntu Server 22.04 | `vsftpd` | Servidor optimitzat per a la transferència ràpida i segura de fitxers. |
| **Clients** | Ubuntu Desktop / Windows | Navegadors, `ssh` | Equips amb entorn gràfic (GUI) necessaris per simular l'usuari final i administrar els servidors visualment. |

-----

## 3\. Desplegament del Nucli de Xarxa (R-N01)

L'objectiu d'aquesta fase ha estat crear el node central que interconnecta totes les xarxes.

### 3.1. Configuració de Xarxa (Netplan)

Hem configurat tres interfícies físiques per separar el trànsit:

  * `enp1s0` (NAT): Sortida a Internet.
  * `enp2s0` (Intranet): Porta d'enllaç `192.168.10.1`.
  * `enp3s0` (DMZ): Porta d'enllaç `192.168.110.1`.

`[IMAGE: Captura del fitxer netplan del router]`

### 3.2. Enrutament i NAT

Hem habilitat l'`ip_forwarding` al nucli (`/etc/sysctl.conf`) i configurat el NAT amb `iptables` perquè les xarxes privades tinguin sortida a Internet:

```bash
sudo iptables -t nat -A POSTROUTING -o enp1s0 -j MASQUERADE
```

### 3.3. Serveis d'Infraestructura (DHCP i DNS)

Per automatitzar la gestió de clients, hem implementat:

  * **DHCP (`isc-dhcp-server`):** Assigna IPs dinàmiques (rang `.100-.200`) als clients de la Intranet.
  * **DNS (`bind9`):** Resol noms interns (`W-N01`, `B-N01`, etc.) facilitant l'administració sense necessitat de recordar IPs.

-----

## 4\. Configuració dels Clients

Hem validat la infraestructura desplegant clients tant Linux com Windows a la Intranet (`G1`). Aquests equips reben la configuració de xarxa automàticament via DHCP i s'utilitzen com a consoles d'administració (SSH, SCP) i per a proves d'usuari final.

`[IMAGE: Captura d'un ipconfig/ifconfig al client mostrant IP i DNS correctes]`

-----

## 5\. Desplegament de la Capa de Dades (B-N01)

El servidor de base de dades s'ha ubicat a la Intranet per maximitzar la seguretat, sent inaccessible directament des d'Internet.

### 5.1. Instal·lació i Securització

S'ha instal·lat MySQL Server i s'ha executat el procés de securització (`mysql_secure_installation`) per eliminar accessos anònims i restringir l'usuari root. A més, s'ha configurat `bind-address` per permetre connexions remotes controlades.

### 5.2. Importació de Dades i Resolució d'Incidències

L'objectiu era carregar un fitxer CSV d'OpenData BCN. Durant el procés, vam trobar i solucionar dos problemes crítics:

1.  **Codificació Incorrecta:** El fitxer original (`UTF-16LE`) generava caràcters corruptes a Linux.
      * *Solució:* Conversió a UTF-8 mitjançant `iconv`.
2.  **Claus Duplicades:** El camp ID del CSV no era únic.
      * *Solució:* Vam reestructurar la taula afegint un camp `id_intern` autoincremental com a clau primària.

**Script SQL definitiu de càrrega:**

```sql
CREATE TABLE equipaments (
    id_intern INT AUTO_INCREMENT PRIMARY KEY,
    id_registre VARCHAR(20),
    nom VARCHAR(255),
    latitud DECIMAL(11, 8),
    longitud DECIMAL(11, 8),
    ...
);

LOAD DATA INFILE '/var/lib/mysql-files/final.csv'
INTO TABLE equipaments
CHARACTER SET utf8mb4
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 LINES;
```

`[IMAGE: Captura d'un SELECT count(*) mostrant els 2.466 registres]`

-----

## 6\. Obertura de Ports i Seguretat (Firewall)

El tallafocs del Router bloqueja tot el trànsit entre xarxes per defecte. Per permetre el funcionament de l'aplicació, hem obert "túnels" específics amb `iptables`.

### 6.1. Connexió Web -\> BBDD

Perquè el servidor web (DMZ) pugui consultar les dades (Intranet), hem permès el trànsit TCP pel port 3306:

```bash
sudo iptables -A FORWARD -i enp3s0 -o enp2s0 -s 192.168.110.10 -d 192.168.10.10 -p tcp --dport 3306 -j ACCEPT
```

-----

## 7\. Desplegament del Servidor FTP (F-N01)

Hem configurat un servidor `vsftpd` a la DMZ (`192.168.110.11`) per a la transferència de fitxers.

### 7.1. Configuració de Ports Passius

Per garantir que l'FTP funcioni a través del tallafocs, hem configurat el mode passiu i hem obert el rang de ports necessari al Router:

```bash
# Ports de control (21), dades actives (20) i passius (10000-10100)
sudo iptables -A FORWARD -i enp2s0 -o enp3s0 -p tcp --dport 21 -d 192.168.110.11 -j ACCEPT
sudo iptables -A FORWARD -i enp2s0 -o enp3s0 -p tcp --dport 10000:10100 -d 192.168.110.11 -j ACCEPT
```

`[IMAGE: Captura d'una sessió FTP exitosa des del client]`

-----

## 8\. Desplegament del Servidor Web i Aplicació (W-N01)

El servidor web (`192.168.110.10`) allotja l'aplicació final desenvolupada.

### 8.1. Stack Tecnològic

Hem utilitzat un entorn LAMP lleuger (Apache + PHP + Mòdul MySQL) sense base de dades local, ja que connecta al servidor remot B-N01.

### 8.2. Desenvolupament de l'Aplicació

Hem creat una aplicació web moderna amb les següents característiques:

  * **Backend (PHP):** Connecta a la BBDD remota, executa la consulta i retorna un JSON net.
  * **Frontend (JS + Leaflet):** Visualitza les dades en format taula, targetes i un mapa interactiu.
  * **Solució Mapa:** Vam implementar filtres de coordenades en JS per evitar errors de visualització (punts 0,0) i un ajust automàtic del zoom.

`[IMAGE: Captura de la web funcionant (Vista Mapa)]`

-----

## 9\. Proves de Sistema i Conclusions

Per validar l'èxit del projecte, s'han realitzat bateries de proves funcionals des dels clients de la Intranet:

| Prova | Origen | Destí | Resultat |
| :--- | :--- | :--- | :--- |
| **Ping** | Client Intranet | Router / Web / FTP | ✅ Èxit |
| **SSH** | Client Intranet | Tots els servidors | ✅ Èxit |
| **MySQL Remot** | Servidor Web (DMZ) | Servidor BBDD (Intranet) | ✅ Èxit (Port 3306 obert) |
| **FTP Upload** | Client Intranet | Servidor FTP (DMZ) | ✅ Èxit (Ports passius OK) |
| **Navegació Web** | Client Windows | Servidor Web (DMZ) | ✅ Èxit |

**Conclusió Final:**
La infraestructura desplegada compleix rigorosament amb els requisits de seguretat i funcionalitat. L'arquitectura de xarxa segmentada protegeix les dades sensibles a la Intranet mentre ofereix serveis públics (Web/FTP) a la DMZ, tot gestionat de manera centralitzada pel Router.