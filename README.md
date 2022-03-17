# Rapport de projet : Qualité de l’air et santé des populations 

> Bertrand Baudeur
> Gilles Mertens



## Rappel sujet

Le projet porte sur la mesure de qualité de l'air à Grenoble. Il est porté par Marie-Laure Aix de l'équipe EPSP du laboratoire TMIC. Cette dernière cherche à mesurer les concentrations de particules fines à différents endroits de la ville pour en tirer des conclusions sur la santé des populations.
Dans le cadre de cette démarche, notre objectif est de développer une station de mesure équipée de capteurs de particule fine, de température et d'humidité et de la connecter via le réseau LoRa à un serveur qui affichera les mesures relevées.

Étant surtout de l'ordre de l'expérimentation, le cahier des charges n'était pas très précis et restrictif.
Il y avait cependant les objectifs suivants :
 - Connecter la carte via le réseau LoRa. (Pas de spécification sur le réseau à utiliser.)
 - Alimenter la carte sur secteur. Et, si le temps le permettait, vérifier la faisabilité de l'utilisation de batterie ou d'énergie solaire.
 - Tester quelle quantité de données pouvait être reçue
 - Développer un serveur qui reçoit, stock et affiche les données émises par la carte.
 - Implémenter la possibilité de changer les paramètres de la carte depuis le serveur.

## Technologies employées
Notre projet ce décompose en deux grandes parties :
### La partie capteur 

Sur cette partie, ce sont des cartes électroniques "LORA e5 mini" qui ont été utilisées. Elles ont été programmées en C à l'aide du mini système d'exploitation **RIOT OS**. Ce système d'exploitation nous permet d'utiliser le radio **LoRa** pour communiquer avec le serveur de visualisation. Il nous permet également de communiquer avec les différents capteurs pour relever de l'information. Les technologies de communicataion étaient : 
 - **I2C** pour le capteur d'humidité et température (BME280)
 - **UART** pour le capteur de pollution (PMS7003). 

Grâce à la communauté autour de RIOT, il existait déjà un driver pour le BME280 qui permet l'auto-configuration et expose des méthodes pour récupérer les données facilement. En revanche pour le PMS7003 aucun driver n'était disponible, nous l'avons donc écrit a partir de l'interface **UART** fournie par RIOT.

### La partie serveur

La partie serveur se décompose en 3 parties :

#### La collecte des donnéee.

Développée en **Python** la partie qui collecte les données se connecte à TTN ou CampusIOT afin de récupérer les paquets descendants et d'envoyer des paquets montants.
Cette partie gère également un serveur HTML qui affiche des pages de configuration, permettant de gérer les cartes déployées et de gérer leur configuration.
Les bibliothèques utilisées sont :
 - paho-mqtt (pour la connexion mqtt)
 - http.server (pour le serveur http)
 - influxdb_client (pour la connexion à influxdb)

#### Le stockage des données.

Le stockage de données est géré par **influxDB** qui est un système de stockage de données orienté série temporelle.
On y stocke les données de pollution de l'air ainsi que des informations sur la santé du réseau LoRa de chaque carte.

#### L'affichage des données.

L'affichage des données se fait grâce à **Grafana** qui puise les données sur **influxDB**. On y affiche les données de pollution ainsi que des tableaux de bord indiquant la santé des réseaux des cartes.

## Architecture techniques

```


                                                 x   x       LoRa
                                                  x x                      x   x
                                                   x   - - - - - - - - - -  x x
                                                   x                         x
                      ┌───────────────────────────────┐                      x
                      │                               │                    ┌────────────────────┐
                      │                               │                    │                    │
                      │      Carte LoRa-e5 mini       │                    │    Gateway LoRa    │
                      │                               │                    │                    │
                      │                               │                    └─────┬──────────────┘
                      └─┬──────┬──────────────────────┘                          │
                        │      │                                                 │ HTTP
                    i2c │      │                                                 │
                        │      │                                                 │
  ┌─────────────────────┴──┐   │                                          ┌──────┴────────────────────┐
  │       PMS7003          │   │ uart                                     │                           │
  │ (capteur de pollution) │   │                                          │    Serveur réseau LoRa    │
  └────────────────────────┘   │                                          │     (TTN / CampusIOT)     │
                               │                                          │                           │
   ┌───────────────────────────┴─────┐                                    └─────┬─────────────────────┘
   │             BME280              │                                          │
   │ Capteur température et humidité │                                          │
   └─────────────────────────────────┘                                          │ HTTP/MQTT
                                                                                │
                                                                                │
                                                                                │
                                               ┌────────────────────────────────┴───────┐
                                               │                                        │
                                               │             Serveur backend            │
                                               │  (Graphana / Influx / configuration)   │
                                               │                                        │
                                               └────────────────────────────────────────┘

```

## Réalisations techniques

### Diver PMS7003
Nous avons développé un diver complet pour le pms7003 permettant d'exploiter toutes ses fonctionnalités nottamment le mode veille. Pour cela nous avons réalisé une machine à états prennant en entrée différents évènements envoyés par l'utilisateur, par le pms et par différents timers permettant de se conformer aux spécifications physiques du capteur (par exemple l'attente de 30 secondes après le démarrage du ventilateur pour avoir une mesure valide et représentative de l'environnement). 
Le schema de cette machine à états est disponible ci dessous.
![](https://i.imgur.com/UCzzG3b.jpg)

#### Le câblage
Le câblage des différents capteurs avec la carte LoRa-e5 mini: 
![](https://i.imgur.com/SfEcEL6.png)

#### Le boitier

![](https://i.imgur.com/1PuJQ0E.png)
![](https://i.imgur.com/1MPDI25.png)
![](https://i.imgur.com/871HXZW.png)

Le boîtier est conçu pour être étanche, mais aussi pour exposer les capteurs à l'air ambiant.
De plus, nous avons testé deux configurations, l'une ou le capteur de température est placé sous le boîtier, ce qui permet une découpe plus simple, et est plus sûr en terme d'étanchéité, mais il existe un risque que l'air à proximité du capteur soit piégé, et donc peu représentatif de l'air ambiant.
La configuration sur le côté, plus dur à découper et rendre étanche, règle possiblement le problème de l'air piégé.

#### L'alimentation

La carte est allimentée en 5v par USB. Nous utilisions une alimentaion de téléphone pour faire la conversion du courrant secteur. Pour l'installation nous avons prévu connecter cette alimentation à une ralonge dans un boitier de dérivation rendu étanche afin d'éviter les dégats causés par l'eau, la poussière et la lumiere.

### Côté serveur

#### Docker

Les différentes parties du serveur sont gérées et mise en relation par docker, il permet à leur configuration de fonctionner de manière durable et de ne pas entrer en collision avec les applications déjà déployée sur le serveur d'accueil.

#### Nginx

Pour les communications entre le navigateur de l'utilisateur, un conteneur nginx est utilisé afin d'obtenir des redirection d'url vers les bon conteneurs et d'encapsuler les module derrière le même domaine. Par exemple `exemple.com/api` pour l'api du backend python, `exemple.com/influx` pour accéder a influxdb et `example.com` pour acceder à Grafana.

#### Le backend Python

Le serveur python gère les différentes stations, il gère la récupération des paquets perdus ainsi que le stockage des mesures envoyées. Il contient aussi un serveur HTTP qui délivre des pages HTML/CSS/JS pour permettre de configurer les cartes.

#### InfluxDB

La base de données influxDB stocke séparément les données mesurées par toutes les stations. Il stocke également des données tel que le nombre de paquets perdus et le nombre de paquets récupérés.

## Gestion de projet (méthode, planning prévisionnel et effectif, gestion des risques, rôles des membres ...)
### Gantt
Nous avons réalisé un gantt au début du projet surtout pour décomposer le projet en taches et pour pouvoir nous répartir le travail. 
![](https://i.imgur.com/qvF6Y7F.png)

### Agilité
Nous avons travaillé en agilité avec les porteurs du projet en faisant des réunions chaque semaine afin de discuter de l'avancement, répondre à leurs questions, et discuter des fonctionnalités.

### Rôles
Bertrand Baudeur : Scrum Master
Gilles Mertens : Chef de projet

Étant donné que nous n'étions que deux sur ce projet, nous n'avons pas beaucoup mis en application ces rôles, les décisions étant communes et les réunions n'ayant pas besoin d'être monitorée.

## Outils (collaboration, CD/CI ...)

### GitHub

Nous avons créé une organisation sur GitHub qui contient 3 dépots :
 - Firmware, il contient le code du firmware de la carte.
 - backend, il contient les fichiers permettant de déployer le backend.
 - documents, il contient toute la documentation qui permettra l'utilisation ou la reprise de notre projet.

### Discord

N'étant une équipe que de deux personnes, l'utilisation d'un trello ou de github project aurait été très vite inutilisée et abandonnée. Nous nous sommes donc cantonné à des communications via un espace de discussion discord que nous avons tout de même disjoint de nos conversations privées.

### Zoom

Pour toutes les réunions avec les porteurs de projet, nous utilisions Zoom. Ces réunions avaient lieu 1 fois par semaine. En plus des nombreux échanges de mails réguliers.

### NextCloud & OnlyOffice

Ce sont des outils permettant de stocker des fichiers, documents texte / images / diapositive et de les rédiger en collaboration.


## Métriques logiciels : lignes de code, langages, performance, temps ingénieur (d'après vos journaux), la répartition des lignes de code et des commits en pourcentage entre les membres du projet ...)

### Lignes de code et languages:
Firmware de la carte : ~
 - ~1000 sloc de **code C**

Serveur backend : 
 - ~ 150 sloc de **configuration** (docker-compose, configurations de influx/nginx/graphana)
 - ~ 700 sloc de **python** (gestion des données et de l'api utilisateur)
 - ~ 500 gggsloc d'**html**/**css**/**javascript** pour interface de configuration utilisateur

## Conclusion (Retour d'expérience)

Dans l'ensemble le projet était stimulant et enrichissant, la finalité était en accord avec nos valeurs et la réalisation nous a fait découvrir de nombreuses technologies et savoir faire.

Notre plus grande frustration réside dans le manque de temps accordé par l'enseignement au projet. En effet nous avions pendant la moitié du projet une autre mission plus urgente et tout aussi exigeante en terme d'investissement. Ce qui n'était apparemment pas le cas les années précédentes.
Avec plus de temps nous aurions pu aborder d'autres sujets intéressants comme le support d'un maximum de réseaux LoRa ou encore la recherche sur les possibles en terme d'alimentation solaire.

## Remerciement

Nous remercions nos porteurs de projet Marie-Laure AIX et Dominique BICOUT avec qui nous avons aprécié travailler et avec qui nous avons pu mener à bien le projet.

Nous remercions également les deux managers du fablab, Germain et Noé pour leur expertise et leurs conseils pour la réalisation du boitier;  ainsi que Dorian PASTIAU pour la visite et l'accès à la tour Perret.

Merci à Didier DONSEZ pour ses conseil sur les question techniques du projet.

Merci à Simon Pervier pour les informations fournie sur le serveur hôte.

## Bibliographie

Documentation technique de la Lora-E5 Mini Seeed Studio :
https://wiki.seeedstudio.com/LoRa_E5_mini/

Documentation technique du PMS7003 :
https://download.kamami.pl/p564008-PMS7003%20series%20data%20manua_English_V2.5.pdf

Documentation technique du BME280 :
https://www.bosch-sensortec.com/products/environmental-sensors/humidity-sensors-bme280/

Cours de découverte de RIOT :
https://github.com/riot-os/riot-course



## Glossaire

*Backend* : Serveur faisant généralement le traitement et le stockage des données 
*BME280* : Capteur de pression, humidité et température.
*CSS* : "Cascading Style Sheets" Langage informatique qui décrit la présentation des documents HTML et XML.
*Docker* : Plateforme permettant de lancer certaines applications dans des conteneurs logiciels.
*HTML* : "HyperText Markup Language" Langage de balisage conçu pour représenter les pages web.
*HTTP* : "Hypertext Transfer Protocol" Protocole de communication client-serveur développé pour le World Wide Web.
*I2C* : Bus informatique permettant la communication entre des microcontrolleurs.
*Javascript* : Langage de programmation de scripts principalement employé dans les pages web interactives
*LoRa-E5* : Microcontroleur utilisé pour piloter les capteurs et le modem LoRa.
*LoRa* : Technologie sans fil permettant des communications longue distance et a faible consommation.
*MQTT* : "Message Queuing Telemetry Transport" Protocole de messagerie publish-subscribe basé sur le protocole TCP/IP
*PMS7003* : Capteur de particules fines.
*Uart* : "Universal Asynchronous Receiver Transmitter" Un émetteur-récepteur asynchrone universel utilisé pour de la communication entre composants electroniques ou avec un ordinateur.
