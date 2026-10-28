# Odoo High Availability Architecture

**Conception d'une architecture hautement disponible et sécurisée pour Odoo ERP**

![Status](https://img.shields.io/badge/Project-Engineering%20Project-blue)
![Platform](https://img.shields.io/badge/Platform-Linux-orange)
![Application](https://img.shields.io/badge/Application-Odoo-purple)
![Database](https://img.shields.io/badge/Database-PostgreSQL-blue)
![Security](https://img.shields.io/badge/Security-ISO%2FIEC%2027001%20aligned-green)

## 1. Présentation du projet

Ce projet porte sur la conception et la mise en place d'une architecture de haute disponibilité pour un ERP Odoo destiné à une entreprise du secteur des Technologies de l'Information et de la Communication (TIC).

Odoo est une solution de gestion intégrée qui centralise plusieurs processus métiers : ventes, achats, facturation, comptabilité, stocks et relation client. Une interruption de ce service peut perturber les opérations de l'entreprise et limiter l'accès aux données essentielles.

L'objectif est de réduire les interruptions de service grâce à la redondance des composants critiques, à la répartition de charge, à la réplication des données, à la segmentation réseau et aux mécanismes de sauvegarde et de supervision.

Le projet contribue aux objectifs de disponibilité et de continuité d'un Système de Management de la Sécurité de l'Information (SMSI) aligné sur ISO/IEC 27001. La mise en œuvre de cette architecture ne constitue pas, à elle seule, une certification ISO 27001.

### Objectifs

- Réduire les points uniques de défaillance (*Single Points of Failure — SPOF*).
- Distribuer les requêtes entre plusieurs instances Odoo.
- Permettre le basculement de l'adresse IP virtuelle du Load Balancer.
- Répliquer les données PostgreSQL vers un serveur de secours.
- Séparer les zones réseau et limiter les communications autorisées.
- Préserver la cohérence du filestore Odoo et de la base de données.
- Superviser les composants et préparer la restauration après incident.
- Tester et mesurer les mécanismes de reprise.

## 2. Architecture générale

```text
                 UTILISATEURS
                      |
                    HTTPS
                      |
                ROUTEUR / WAN
                      |
                FIREWALL / ACL
                      |
             +--------+--------+
             |                 |
          LB01                LB02
       HAProxy +           HAProxy +
       Keepalived          Keepalived
             |                 |
             +------- VIP -----+
                      |
             +--------+--------+
             |                 |
          ODOO01              ODOO02
             |                 |
             +--------+--------+
                      |
              ENDPOINT DATABASE
                      |
               +------+------+
               |             |
             DB01           DB02
           PRIMARY         STANDBY
               |             ^
               +-- Replication
                      |
              Shared/Replicated
                  Filestore

       MANAGEMENT / MONITORING / BACKUP
```

**Remarque :** ce schéma représente la cible logique. La disponibilité réelle dépend de la configuration des mécanismes de basculement, du stockage, du réseau et de la base de données. Les routeurs, firewalls, le stockage et les services de supervision doivent être considérés comme redondants uniquement si cette redondance est effectivement mise en œuvre.

## 3. Rôle des composants

| Composant | Fonction |
|---|---|
| LB01 / LB02 | Répartir les requêtes et fournir une entrée Web redondante |
| HAProxy | Distribuer les requêtes vers les serveurs applicatifs disponibles |
| Keepalived / VRRP | Gérer la VIP et permettre son déplacement entre les Load Balancers |
| ODOO01 / ODOO02 | Exécuter l'application Odoo |
| DB01 | Serveur PostgreSQL primaire, qui accepte les écritures |
| DB02 | Serveur PostgreSQL standby recevant les données répliquées |
| Endpoint DB | Fournir une adresse stable à laquelle Odoo se connecte après un basculement |
| Filestore | Stocker les pièces jointes et fichiers associés aux données Odoo |
| Backup | Conserver des copies récupérables de la base, des fichiers et des configurations |
| Monitoring | Détecter les pannes, surveiller les ressources et générer des alertes |
| Firewall / ACL | Autoriser uniquement les flux nécessaires entre les zones réseau |

### Pourquoi séparer DMZ et réseau interne ?

La DMZ héberge le point d'entrée Web contrôlé. Les serveurs Odoo sont placés dans une zone applicative interne, et PostgreSQL dans une zone de données isolée.

Cette séparation limite l'exposition des serveurs et réduit les possibilités de communication en cas de compromission d'un composant.

Un firewall contrôle les flux réseau. Un WAF, lorsqu'il est installé, ajoute des contrôles spécifiques aux requêtes Web. Ces deux fonctions ne sont pas équivalentes.

## 4. Plan d'adressage IP indicatif

Le tableau suivant constitue un exemple à adapter au laboratoire. Il ne doit pas être considéré comme la configuration réellement déployée sans vérification.

| Machine | Adresse indicative | Rôle |
|---|---|---|
| LB01 | 192.168.100.10 | Load Balancer principal |
| LB02 | 192.168.100.11 | Load Balancer secondaire |
| ODOO01 | 192.168.100.20 | Application Odoo 1 |
| ODOO02 | 192.168.100.21 | Application Odoo 2 |
| DB01 | 192.168.100.30 | PostgreSQL Primary |
| DB02 | 192.168.100.31 | PostgreSQL Standby |
| STORAGE01 | 192.168.100.40 | Stockage du filestore, si utilisé |
| VIP | 192.168.100.100 | Adresse virtuelle des Load Balancers |

Dans une architecture segmentée, chaque zone possède son propre sous-réseau. Il faut alors définir les passerelles, les routes et les règles de filtrage correspondantes.

**Avant d'exécuter une commande :** vérifier les adresses, les interfaces, les noms d'hôtes, les versions et les ports réellement utilisés.

## 5. Prérequis

- Machines virtuelles ou serveurs Linux accessibles.
- Système d'exploitation compatible avec les paquets retenus, par exemple RHEL 9.x.
- Deux Load Balancers pour le scénario de basculement.
- Deux instances Odoo configurées de manière cohérente.
- Deux serveurs PostgreSQL pour la réplication.
- Connectivité réseau entre les composants.
- Accès administrateur pour l'installation.
- Espace de stockage suffisant pour les bases et le filestore.
- Sauvegardes et espace de restauration.
- Ports autorisés uniquement entre les sources et destinations nécessaires.

Les commandes suivantes sont des exemples pour un environnement RHEL 9. Elles doivent être adaptées à la version exacte des logiciels et à la configuration du laboratoire.

## 6. Installation et configuration des Load Balancers

### 6.1. Installer HAProxy et Keepalived

Sur LB01 et LB02 :

```bash
sudo dnf install -y haproxy keepalived
sudo systemctl enable haproxy keepalived
```

Vérifier les versions :

```bash
haproxy -v
keepalived --version
```

### 6.2. Configurer HAProxy

Fichier : `/etc/haproxy/haproxy.cfg`

Exemple minimal à adapter :

```haproxy
global
    log /dev/log local0
    maxconn 2000

defaults
    mode http
    log global
    option httplog
    timeout connect 5s
    timeout client 60s
    timeout server 60s

frontend odoo_frontend
    bind *:9090
    default_backend odoo_backend

backend odoo_backend
    balance roundrobin
    option httpchk GET /web/login
    http-check expect status 200
    server odoo01 192.168.100.20:8069 check
    server odoo02 192.168.100.21:8069 check
```

Cet exemple expose HAProxy sur le port 9090 pour les tests. En production, prévoir une configuration HTTPS adaptée, la gestion des certificats et les règles nécessaires au mode WebSocket d'Odoo.

Le port interne 8069 doit être accessible depuis les Load Balancers uniquement, selon les règles de sécurité du laboratoire.

Valider la configuration :

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```

Redémarrer et vérifier le service :

```bash
sudo systemctl restart haproxy
sudo systemctl status haproxy --no-pager
sudo journalctl -u haproxy -n 50 --no-pager
```

### 6.3. Configurer Keepalived

Keepalived doit être configuré sur les deux Load Balancers avec :

- une interface réseau réelle ;
- une VIP libre sur le même réseau de couche 2 ;
- un identifiant VRRP commun ;
- des priorités distinctes ;
- un mécanisme de détection de l'état du Load Balancer ;
- une configuration adaptée aux restrictions réseau et à la politique de sécurité.

Exemple indicatif pour LB01 dans `/etc/keepalived/keepalived.conf` :

```text
vrrp_instance VI_1 {
    state MASTER
    interface ens160
    virtual_router_id 51
    priority 150
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass CHANGE_ME
    }

    virtual_ipaddress {
        192.168.100.100/24
    }
}
```

Sur LB02, adapter la configuration :

```text
state BACKUP
priority 100
```

Les deux instances doivent partager le même `virtual_router_id`, le même type de réseau et la même VIP. Remplacer `ens160` par le véritable nom d'interface et modifier le masque si nécessaire.

La configuration doit également être adaptée pour surveiller HAProxy : une VIP active sur un serveur dont HAProxy est arrêté ne garantit pas la disponibilité du service.

Après validation :

```bash
sudo keepalived -t -f /etc/keepalived/keepalived.conf
sudo systemctl enable --now keepalived
sudo systemctl status keepalived --no-pager
ip addr
```

**Attention :** le comportement exact de l'option de test varie selon la version de Keepalived. Si elle n'est pas prise en charge, utiliser la validation de configuration indiquée par la version installée et consulter les journaux.

## 7. Configuration des serveurs Odoo

Les deux instances doivent utiliser des versions compatibles, une configuration cohérente et un accès commun ou synchronisé aux fichiers nécessaires.

### 7.1. Vérifier le service

Sur chaque serveur Odoo :

```bash
sudo systemctl status odoo --no-pager
sudo journalctl -u odoo -n 100 --no-pager
```

Si Odoo est exécuté dans Docker :

```bash
docker ps
docker logs --tail 100 <nom_du_conteneur_odoo>
```

Remplacer le nom du conteneur par celui réellement utilisé.

### 7.2. Paramètres importants

Dans le fichier de configuration Odoo, vérifier notamment :

```ini
proxy_mode = True
```

Ce paramètre doit être utilisé lorsque Odoo est correctement placé derrière un reverse proxy de confiance. Il ne faut pas activer ce mode sans maîtriser les accès directs et les en-têtes transmis par le proxy.

Vérifier également :

- la connexion à PostgreSQL ;
- les paramètres de workers et de ressources ;
- les ports applicatifs réellement utilisés ;
- le routage WebSocket ou gevent si la version et la configuration l'exigent ;
- la cohérence des modules et des versions entre ODOO01 et ODOO02.

Le port 8072 est utilisé dans certaines configurations Odoo pour le trafic gevent/WebSocket. Confirmer le port exact dans la configuration déployée.

### 7.3. Vérifier l'accès HTTP

Depuis une machine autorisée :

```bash
curl -I http://192.168.100.20:8069/web/login
curl -I http://192.168.100.21:8069/web/login
```

Un code HTTP 200 est un résultat attendu pour certaines configurations, mais les redirections ou d'autres codes peuvent être normaux selon la version et les paramètres. Compléter par un test de connexion fonctionnelle.

## 8. Configuration de PostgreSQL Primary/Standby

### 8.1. Vérifier PostgreSQL

Sur les deux serveurs :

```bash
psql --version
sudo systemctl status postgresql --no-pager
```

Le nom du service peut varier selon la méthode d'installation et la version.

### 8.2. Préparer la réplication

Pour une réplication physique en streaming, vérifier notamment :

- `wal_level = replica` sur le primaire ;
- un nombre suffisant de connexions de réplication ;
- les paramètres `max_wal_senders` et `max_replication_slots` selon la conception ;
- une règle `pg_hba.conf` limitant la réplication à l'adresse du standby ;
- un compte dédié à la réplication avec des privilèges minimaux ;
- une méthode d'authentification sûre ;
- la compatibilité des versions PostgreSQL ;
- une synchronisation horaire cohérente.

Après modification des paramètres nécessitant un redémarrage, appliquer la procédure appropriée à la version installée.

### 8.3. Contrôler le rôle des serveurs

Sur chaque instance PostgreSQL, dans une session autorisée :

```sql
SELECT pg_is_in_recovery();
```

Interprétation :

- `false` : instance primaire en fonctionnement normal ;
- `true` : instance en récupération, généralement un standby.

Sur le primaire :

```sql
SELECT application_name, client_addr, state, sync_state
FROM pg_stat_replication;
```

Sur le standby :

```sql
SELECT status, sender_host, received_lsn, latest_end_lsn
FROM pg_stat_wal_receiver;
```

Les vues disponibles et leurs colonnes peuvent varier selon la version. Les résultats doivent être interprétés en fonction du mode de réplication et de l'état réel du cluster.

### 8.4. Point essentiel : le basculement PostgreSQL

La réplication ne transforme pas automatiquement un standby en primaire.

Pour une haute disponibilité complète, il faut concevoir et tester :

1. la détection de la panne du primaire ;
2. la décision de promotion du standby ;
3. la prévention du *split-brain* ;
4. le déplacement ou la mise à jour de l'endpoint DB ;
5. la reconnexion des instances Odoo ;
6. la resynchronisation de l'ancien primaire avant son retour au service.

Un outil de gestion HA, tel que Patroni associé à un mécanisme de consensus correctement dimensionné, peut être étudié. Pour un laboratoire plus simple, une promotion contrôlée et documentée peut être retenue, en précisant qu'elle ne représente pas un failover automatique.

Ne jamais promouvoir arbitrairement un standby en présence d'un primaire potentiellement encore actif : cela peut créer deux primaires et compromettre l'intégrité des données.

## 9. Filestore, sauvegarde et restauration

### 9.1. Pourquoi le filestore est essentiel

Odoo stocke des pièces jointes et des fichiers en dehors des tables PostgreSQL, généralement dans son filestore. La base de données et les fichiers doivent rester cohérents.

Si ODOO01 et ODOO02 n'accèdent pas au même filestore ou à des copies cohérentes, certains documents peuvent être absents après un changement de serveur.

Un partage NFS peut être utilisé, mais un serveur NFS unique devient lui-même un point de défaillance. La solution de stockage doit donc être documentée et testée séparément.

### 9.2. Vérifier le stockage

Sur un serveur autorisé :

```bash
df -h
findmnt
ls -ld /srv/odoo-filestore
```

Adapter le chemin au véritable emplacement des fichiers Odoo.

### 9.3. Sauvegarde PostgreSQL

Une sauvegarde logique peut être réalisée avec `pg_dump`, si cette méthode correspond au besoin et à la taille de la base :

```bash
pg_dump -Fc -d <nom_base> -f odoo_backup.dump
```

La commande doit être exécutée avec un compte autorisé. Configurer l'authentification de manière sûre et éviter de placer les mots de passe en clair dans l'historique du shell.

Pour restaurer dans une base de test vide :

```bash
createdb <base_test>
pg_restore -d <base_test> odoo_backup.dump
```

La restauration des rôles, extensions, propriétaires et dépendances peut nécessiter des étapes supplémentaires.

Sauvegarder aussi le filestore, les configurations nécessaires et les éléments permettant de reconstruire l'environnement. La réplication n'est pas un remplacement des sauvegardes : une suppression ou une corruption peut également être répliquée.

## 10. Commandes de diagnostic

### 10.1. Réseau

```bash
ip addr
ip route
ping -c 4 <adresse_cible>
```

Vérifier le routage, le masque, la passerelle, les VLAN et les règles de filtrage.

### 10.2. Connectivité des ports

```bash
nc -vz <adresse_lb> 9090
nc -vz <adresse_odoo> 8069
nc -vz <adresse_db> 5432
```

Ces tests doivent être lancés depuis une machine autorisée et uniquement vers les ports nécessaires. Un port ouvert ne prouve pas que l'application fonctionne correctement.

### 10.3. Services et journaux

```bash
sudo systemctl --failed
sudo journalctl -p err -b --no-pager
sudo journalctl -u haproxy -n 100 --no-pager
sudo journalctl -u keepalived -n 100 --no-pager
```

Adapter le nom du service PostgreSQL et le mode de journalisation à l'environnement installé.

### 10.4. Vérifier la VIP

```bash
ip addr
```

La VIP doit être présente sur le Load Balancer actif selon la configuration VRRP. Pendant un test, vérifier également les journaux Keepalived et la connectivité depuis un client.

## 11. Scénarios de test

Les tests doivent être réalisés dans un environnement de laboratoire maîtrisé, avec une sauvegarde préalable et un plan de retour arrière.

### Test 1 — Indisponibilité d'Odoo01

**Procédure :**

1. Vérifier l'accès normal à Odoo.
2. Arrêter le service Odoo01 ou son conteneur de manière contrôlée.
3. Vérifier l'état des backends HAProxy.
4. Tester l'accès via l'URL du Load Balancer.
5. Vérifier les journaux et la cohérence des données.

**Résultat attendu :** les nouvelles requêtes sont dirigées vers Odoo02 si celui-ci est sain et correctement configuré.

### Test 2 — Indisponibilité de LB01

**Procédure :**

1. Vérifier la présence de la VIP sur LB01.
2. Arrêter Keepalived ou provoquer une panne contrôlée de LB01.
3. Vérifier la VIP sur LB02.
4. Tester l'accès depuis le client.
5. Mesurer le délai de reprise.

**Résultat attendu :** LB02 reprend la VIP et fournit le service, si VRRP, le réseau et HAProxy sont correctement configurés.

### Test 3 — Indisponibilité du primaire PostgreSQL

**Procédure :**

1. Vérifier l'état de la réplication.
2. Vérifier la présence d'une sauvegarde récente.
3. Simuler la panne dans un environnement contrôlé.
4. Appliquer la procédure de promotion prévue.
5. Vérifier l'endpoint DB et la reconnexion d'Odoo.
6. Contrôler les transactions et les données.

**Résultat attendu :** la procédure de reprise rétablit les accès à la base sans split-brain. Le résultat dépend du mécanisme de promotion réellement implémenté.

### Test 4 — Restauration d'une sauvegarde

**Procédure :**

1. Restaurer la base dans un environnement isolé.
2. Restaurer le filestore correspondant.
3. Démarrer Odoo avec la configuration de test.
4. Vérifier les utilisateurs, les enregistrements et les pièces jointes.

**Résultat attendu :** l'environnement restauré permet d'accéder aux données et aux fichiers attendus.

### Test 5 — Contrôle d'accès réseau

**Procédure :**

1. Tester la connectivité vers PostgreSQL depuis un réseau non autorisé.
2. Vérifier que la connexion est bloquée.
3. Vérifier que les accès légitimes depuis Odoo restent fonctionnels.
4. Examiner les journaux du firewall si disponibles.

**Résultat attendu :** seuls les flux explicitement autorisés sont acceptés.

## 12. Indicateurs de reprise

| Indicateur | Définition |
|---|---|
| RTO | Durée entre l'interruption et le retour à un service utilisable |
| RPO | Perte de données maximale acceptable, exprimée en temps |
| Disponibilité | Part du temps pendant laquelle le service est utilisable |
| Intégrité | Vérification de la cohérence des données et des fichiers |
| Temps de détection | Durée entre la panne et sa détection |
| Temps de basculement | Durée nécessaire au changement vers le composant de secours |

Pour chaque scénario, consigner l'heure de la panne, l'heure de détection, l'heure du retour à la normale, les erreurs observées, les données vérifiées et les preuves associées.

Ne pas publier de résultats chiffrés avant d'avoir exécuté les tests.

## 13. Problèmes fréquents et solutions

### HAProxy ne démarre pas

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo journalctl -u haproxy -n 100 --no-pager
```

Vérifier la syntaxe, les ports déjà utilisés, les adresses des backends et les permissions.

### La VIP ne bascule pas

```bash
sudo systemctl status keepalived --no-pager
sudo journalctl -u keepalived -n 100 --no-pager
ip addr
```

Vérifier l'interface, les priorités, le `virtual_router_id`, la connectivité VRRP, les restrictions réseau et le mécanisme de surveillance de HAProxy.

### Odoo est inaccessible derrière HAProxy

```bash
curl -I http://<ip_odoo>:8069/web/login
curl -I http://<ip_lb>:9090/web/login
```

Vérifier l'état d'Odoo, les ports, les règles firewall, les health checks et la configuration du reverse proxy.

### PostgreSQL refuse les connexions

```bash
sudo systemctl status postgresql --no-pager
ss -lntp
```

Vérifier `listen_addresses`, `pg_hba.conf`, les règles firewall, l'authentification et l'adresse configurée dans Odoo.

### La réplication est interrompue

Contrôler l'état du WAL receiver sur le standby et les informations de réplication sur le primaire. Vérifier la connectivité, l'authentification, l'espace disque, les journaux PostgreSQL et la rétention des WAL.

Ne pas recréer le standby ni supprimer des données avant d'avoir identifié la cause et sécurisé les données existantes.

### Les pièces jointes sont absentes

Vérifier le chemin du filestore, les permissions, le montage NFS éventuel et la correspondance entre la base restaurée et les fichiers sauvegardés.

### Le service revient mais certaines données manquent

Vérifier les journaux, le retard de réplication, les transactions en cours et les sauvegardes. Une réplication asynchrone peut perdre les dernières transactions non reçues par le standby au moment de la panne.

## 14. Sécurité et bonnes pratiques

- Ne jamais exposer PostgreSQL directement à Internet.
- Restreindre SSH au réseau d'administration.
- Ne jamais publier de mots de passe, clés privées ou jetons.
- Protéger les sauvegardes et tester régulièrement leur restauration.
- Utiliser HTTPS et protéger les certificats.
- Maintenir les systèmes et composants à jour.
- Limiter les privilèges des comptes de service.
- Journaliser les événements importants et surveiller les alertes.
- Documenter les procédures de failover et de retour arrière.
- Ne pas déclarer un composant hautement disponible tant que le mécanisme correspondant n'a pas été testé.

## 15. État du projet

Ce dépôt est destiné à documenter l'architecture, les configurations, les procédures de validation et les résultats du laboratoire.

Les éléments doivent être classés explicitement selon leur état :

- **Conçu :** décrit dans l'architecture.
- **Configuré :** présent dans l'environnement.
- **Testé :** vérifié par une procédure reproductible.
- **Validé :** résultat documenté avec des preuves et des critères définis.

Cette distinction évite de confondre une architecture cible avec une solution entièrement déployée et validée.

## 16. Références officielles

- [Documentation Odoo — Déploiement](https://www.odoo.com/documentation/18.0/fr/administration/on_premise/deploy.html)
- [Documentation PostgreSQL — Haute disponibilité](https://www.postgresql.org/docs/current/high-availability.html)
- [Documentation HAProxy](https://docs.haproxy.org/)
- [Documentation Keepalived](https://www.keepalived.org/manpage.html)
- [ISO — ISO/IEC 27001](https://www.iso.org/standard/27001)

## Auteurs

**Chaima-naili**

Projet d'ingénierie — Haute disponibilité, cybersécurité et continuité de service.
