# ansible-role-eyesoflog
Axians - Brice VALENZA - Juillet 2021

- [ansible-role-eyesoflog](#ansible-role-eyesoflog)
  - [Présentation](#présentation)
  - [Architecture type](#architecture-type)
  - [Prérequis](#prérequis)
  - [Utilisation](#utilisation)
  - [Playbook pour une installation basique](#playbook-pour-une-installation-basique)
  - [Gestion des répertoires](#gestion-des-répertoires)
  - [Gestion des fichiers](#gestion-des-fichiers)
    - [Nommage des ressources EoL](#nommage-des-ressources-eol)
- [Elasticsearch](#elasticsearch)
  - [Remarque sur les rôles Elasticsearch](#remarque-sur-les-rôles-elasticsearch)
  - [Gestion des certificats](#gestion-des-certificats)
    - [Pour un export depuis une PKI Windows.](#pour-un-export-depuis-une-pki-windows)
  - [Gestion des utilisateurs](#gestion-des-utilisateurs)
- [Kibana](#kibana)
- [Metricbeat - Monitoring de la Stack](#metricbeat---monitoring-de-la-stack)
- [Ingestion Syslog / UDP /TCP - Logstash ou Filebeat](#ingestion-syslog--udp-tcp---logstash-ou-filebeat)
- [Intégration de ressource](#intégration-de-ressource)
  - [Playbook d'intégration](#playbook-dintégration)
  - [Ajout de fichier de ressource](#ajout-de-fichier-de-ressource)
- [Troubleshooting](#troubleshooting)
  - [Erreur sur requête API / accés API](#erreur-sur-requête-api--accés-api)
    - [Utilisateur d'un proxy](#utilisateur-dun-proxy)
- [Evolution / Correction a prévoir](#evolution--correction-a-prévoir)
- [Notes diverses](#notes-diverses)
  - [Architecture Hot Warm et réplica](#architecture-hot-warm-et-réplica)




## Présentation
Ce dépot contient les playbooks et ressouces nécessaires au déploiement d'une architecture Eyes of Log.

La partie Déploiement / configuration de la machine  n'est pas abordé dans ce dépôt. Il pourra faire l'objet d'un dépot séparé.

L'avantage d'utiliser ces playbooks :
- Déploiement facilité sur une multitude de noeud;
- Suivi de la configuration / mise à jour a partir d'un point central notamment pour Elasticsearch (ingest pipeline, component template, index template etc.)

Bien que les playbook permettent de faciliter l'installation, l'import des configurations et des ressources, il est indispensable d'avoir un minimum de conaissance sur le fonctionnement d'Elastic pour assurer un déploiement optimale.

Cela passe notamment par le remplissage du fichier d'inventaire avec l'affectation des rôles pour chaque noeud qui doit être réfléchi.
L'inventaire contenu dans ce dépot, propose une architecture à 3 noeud de donnée, dont un portant Kibana.

Il serait trop compliqué d'aborder ici la configuration avancée d'un cluster Elasticsearch (shard allocation etc.), ce type de configuration doit être rajouté dans le playbook ou géré ultérieurement par API.

Cependant, ces playbooks proposent toute la base nécessaire pour aller plus loin, et gérer des architectures complètes (multitude de oeud, sur une multitude de datacenter etc.). Il conviendra simplement d'adapter les inventaires et les variables.

## Architecture type


- Une architecture doit avoir 3 noeuds master;
- Dans une architecture de production, le noeud Kibana doit être indépendant.

Remarque sur Logstash : Logstash n'est nécessaire qu'en cas de duplication de la donnée (sortie multiple), besoin de filter particulier, ou encore de "tampon".
Nous conseillons d'utiliser Filebeat pour recevoir des flux syslog si les autres fonctionnalités de Logstash ne sont pas nécessaires.
Les documents sur Filebeat sont disponible dans le dossier "Others".

## Prérequis

- les machines cibles sont démarrés et configurés à minima avec un utilisateur permettant de lancer des commandes en mode sudo.
- Configuration DNS ok;
- Accés internet OK / ou dépot local configuré;
- Configuration Proxy si nécessaire;
- Hostname et nom de domaine de la machine configuré
- Les certificats pour chaque machine sont générés (certificat + clé privé non chiffré). Ce point pourra être améliorer dans une prochaine version.
- Ansible doit avoir accés aux machines cibles (via clé SSH par exemple)
- Si un pare-feu est présent, les ports doivent être ouvert.

Les playbook sont testés et  fonctionnels sur :
- Centos 7;

## Utilisation

Afin d'utiliser ce playbook, les éléments suivants doivent être modifiés / adaptés à votre cible:
- Fichier d'inventaire "inventory.yml", avec:
  - nom d'hôte correspondant au nom de la VM (en nom court);
- Variables présents dans les fichiers group_vars:
  - all : Contient l'ensemble des configuration du cluster;
- variables présents dans les fichiers host_vars:
  - pour chaque hôte : les adresses IP, et le rôle du noeud dans le cluster;
- Remplacement des fichiers de certificat dans files/certs/ sous le format:
    - hostame_privatekey.pem
      - où "hostname" correspond au nom de la machine dans l'inventaire Ansible;
      - Pour une raison inconnue, je n'ai pas réussi à déployer des clé privées chiffrés (erreur systématique pour déchiffrer la clé)
    - eol-ca.crt (mettre à jour la variable si le nom est modifié);

## Playbook pour une installation basique

Pour une installation basique avec:
- Cluster Elasticsearch;
- Noeud Kibana;

Les playbooks suivants sont suffisants:
- 10_Elasticsearch_Install
- 20_Kibana_Install_redhat
- 30_Metricbeat_StackMonitor

Logstash étant optionnel et l'import de ressource également.
[voir le chapitre sur Logstash si besoin de le déployer sur un noeud](#logstash)

Dans le cadre d'une configuration minimale , rajouter les éléments suivants:
- Playbook:
  - 100_import_Ressources
- files/EyesofLog :
  - component-template:
    - eol-component-template_default_mapping.json
    - eol-component-template_default_settings.json
  - index template:
    - eol-nagios-log.json;
  - lifecycle:
    - eol-ilm-default;
  - pipelines:
    - eol-nagios-log-pipeline.

Ces éléments permettent d'avoir une configuration minimale permettant d'ingérer de la donnée via des agents Beats ou Logstash. (Voir le chapitre Logstahs ou Filebeat, et les examples de configuration Beats dans "Others").

[Voir le chapitre sur l'intégration des ressources pour plus de détail](#Intégration-de-ressource)

La partie Logstash n'est pas nécessaire car on utilisera un agent Filebeat côté EoN.

## Gestion des répertoires

Plusieurs répertoires sont présents :
- files: Contient les fichers nécessaires à intégrer à EoL:
  - Certificats;
  - Des ressouces à intégrer dans Elasticsearch (template, index, ILM, pipelines).
- group_vars : Les variables globales aux playbook;
- host_vars : Les variables pour chaque hôte (Adresse IP et rôle Elasticsearch notamment);
- Others: Ce sont des ressources mis à disposition mais non utilisé dans les playbook on y retrouve :
  - Un exemple de configuration pour les agetns Beats;
  - Une configuration "Filebeat" permettant l'ingestion de flux du type syslog (écoute sur port UDP / TCP);
  - Une configuration "Logstash" permettant l'ingestion de flux du type syslog (écoute sur port UDP / TCP);
- roles : Contenant le prérequis du role Elasticsearch;
- templates: Contenant l'ensemble des modèles et fichiers de configuration qui seront déployés (modèle en format Jinja2).


Note : Par défaut Others/Filebeat est déja intégré (Playbook 50 est fichier dans template/)

## Gestion des fichiers

### Nommage des ressources EoL

le nommage des fichiers des ressources est trés important, car utilisé pour enregistrer les ressources. En effet, le nom de chaque ressource intégré, sera le nom du fichier (sans l'extension).

les noms de pipelines sont également utilisés dans la configuration Logstash (si utilisé). Ainsi, en respectant le code mis en place (voir conf input de logstash), l'ajout de ressource est facilité.

# Elasticsearch

La mise en place de la partie Elasticsearch est basé sur le rôle oficielle ansible-elasticsearch (voir roles/requirements.yml).
```
ansible-galaxy install -r roles/requirements.yml -p roles
```
## Remarque sur les rôles Elasticsearch

Pour le monitorring, un cluster dédié est normalement préconisé. Comme c'est généralement pas le cas dans notre type d'architecture, nous conseillons l'activation du rôle ingest sur tous les noeud elasticsearch. Ainsi, Metricbeat (pour le monitoring), pourra toujours joindre un noeud d'ingestion pour le monitoring.

## Gestion des certificats

Par mesure de sécurité, nous forçons l'utilisation de certificat au sein de l'architecture Eyes of Log.

Il faut donc veiller à générer l'ensemble des certificats SSL pour chaque serveur, et les déposer dans le dossier files/certs sous la forme:
- hostname_privatekey.pem   ==> Clé privé pour le serveur "hostname"
- hostname.crt  ==> Certificat pour le serveur "hostname"
- eol-ca.crt ==> Certificat de l'autorité de certification.

La génération de certificat est possible:
- via un Active Directory avec une PKI;
- via OpenSSL;
- Via un logiciel type XCA (proposant une GUI portable pour gérer une CA et des certificats machines);

Si les clé privé sont chiffrés, ne pas oublier de remplir la variable ssl_key_passphrase présent dans group_vars/all.

### Pour un export depuis une PKI Windows.

1) Générer les certificats depuis un PC dans le domaine.
2) Exporter en format PKCS12 (.PFX)
3) Extraire la clé privée du PFX
```
openssl pkcs12 -in eol-bva-elastic-01.pfx -nocerts -nodes -out eol-bva-elastic-01_privatekey_enc.pem
```

4) Convertir la clé privée en RSA  (elle sera toujours chiffré)
```
openssl rsa -in eol-bva-elastic-01_privatekey_enc.pem -out eol-bva-elastic-01_privatekey.pem
```

5) Exporter le certificat
```
openssl pkcs12 -in eol-bva-elastic-01.pfx -clcerts -nokeys -out eol-bva-elastic-01.crt
```

## Gestion des utilisateurs

Le changement du mot de passe "api" (elastic) d'elasticsearch est le plus sensible car celui utilisé pour appliquer les modifications. Si un changement est à prévoir via le playbook, il faut :
1. conserver l'ancienne valeur dans le fichier de variable "all";
2. Décommenter les lignes sur le playbook d'installation Elasticsearch, en indiquant le nouveau mot de passe;
3. exécter le playbook;
4. vérifier que le mot de passe est bien changé (vérifier via une page web sur l'API elasticsearch). Si ce n'est pas le cas recommencer / débugger.
5. commenter la ligne elastic et la ligne password associé;
6. Mettre le nouveau mot de passe dans le fichier de variable "All".

Pour plus de sécurité, nous conseillons de créer des utilisateurs pour chaque service (logstash, filebeat, winlogbeat etc..) voir d'utiliser directement les clé APIs.

# Kibana

Il n'existe pas de rôle oficielle pour Kibana. Les playbook s'appuient donc sur un déploiement et une configuration manuelle.

Pour le moment, cela concerne uniquement un Os Cent Os / Red Hat.

- Playbook 2X

# Metricbeat - Monitoring de la Stack

Il n'existe pas de rôle oficielle pour Metricbeat. Les playbook s'appuient donc sur un déploiement et une configuration manuelle.

Metricbeat est configuré pour monitorer Elasticsearch et Kibana. Pour chaque noeud, la sortie metricbeat (vers elasticsearch) est configuré pour pointer vers le groupe "ingest".


Pour le moment, cela concerne uniquement un Os Cent Os / Red Hat.

- Playbook 3X


# Ingestion Syslog / UDP /TCP - Logstash ou Filebeat

Pour gérer l'ingestion des flux types syslog, il est possible d'utiliser Logstash ou Filebeat. Si les fonctionnalités de Logstash ne vous intéresse pas, nous conseillons l"utilisation de Filebeat.


Il n'existe pas de rôle oficielle pour Logstash. Les playbook s'appuient donc sur un déploiement et une configuration manuelle.

La configuration de logstash proposé permet un fonctionnement "clé en main", avec un pipeline par défaut (vide), ainsi qu'un index.  
Pour ce faire, il faut bien valider que les fichiers suivants sont déployé via le playbook d'importation des ressources (ou déja existant sur Elasticsearch):
- Component-tempate:
  - eol-component-template_default_mapping.json
  - eol-component-template_default_settings.json
- index_templates:
  - eol-default.json
- lifecycle:
  - eol-ilm-default.json
- pipelines:
  - eol-default-pipeline.json


Le playbook permet également de mettre en place le système "Dead Queue letter" permettant d'assurer l'ingestion des journaux en cas d'erreur (sur la partie pipeline notamment).

Pour le moment, cela concerne uniquement un Os Cent Os / Red Hat.

- Playbook 4X


La configuration Filebeat proposé permet un fonctionnement "Clé en main", avec un pipeline par défaut, ainsi qu'un index.
Les mêmes fichiers de ressources que Logstash sont nécessaires.


Dans les deux cas, La solution permet de filtrer la source pour appliquer des index et des pipelines (via l'utilisation d'un champ) différents en fonction des besoins. Pour cela il est nécessaire de modifier le fichier de configuration (filebeat.j2 ou main-10_Syslog-input.j2)

# Intégration de ressource

## Playbook d'intégration
L'intégration des ressources se basent sur un ensemble de fichier mis à disposition par l'équipe.

Ceux-ci sont stockés a stocker dans le répertoires /files/EyesOfLog/ sous les dossiers suivants:
- component-template :
  - Fichier "composant" de template, contient notamment un mapping des champs par défaut.
- index_templates :
  - fichier modèle des index;
  - Configuration de l'index : nous conseillons d'utiliser les "component-template";
  - Définir l'index template en tant que Datastream (permet de s'affranchir de la gestion des index / alias / rollover)
- lifecycle: 
  - Définition de du cycle de vie de la donnée (hot/warm etc.)
- pipelines : 
  - Définition de pipeline d'ingestion.


Tout document se trouvant dans un de ses repertoires sera déployé sur le cluster Elasticsearch.
Si on ne souhaite pas déployer (ou redéployer) un fichier, un dossier not_deploy est a disposition dans chaque dossier.

Les playbooks d'intégrations de ressources: 
- Playbook 10X

**WARNING** : Attention aux modifications manuelles qui pourrait être réalisés sur le cluster et effacés par les ressources mises à disposition.

## Ajout de fichier de ressource

Pour ajouter des ressources (template, pipeline etc.), nous conseillons de conserver un nommage cohérent (CF chapitre sur le nommage).

Pour ajouter un index:
- Créer le fichier index_templates sur la base d'un autre disponible;
- Lancer le playbook d'import de ressource.
- Envoyer de la donner vers ce Datastream (voir index_patterns définit à l'intérieur). C'est lors de l'ingestion de la première donnée que le Datastream apparaitra sur Elasticsearch / Kibana.

# Troubleshooting

## Erreur sur requête API / accés API

### Utilisateur d'un proxy
Si un proxy est configuré au niveau du système, il faut penser à exclures les URLS locales (127.0.0.1 et localhost notamment).

# Evolution / Correction a prévoir

1. Faire fonctionner Elasticsearch avec des clé privés chiffrés.
2. Protection de tous les mots de passes indiqué dans les playbooks:
   1. api elasic
   2. Kibana
   3. clé passphrase SSL
   4. etc.
   
3. Exemple de configuration avancée :
   1. Allocation Shard;
   2. Nombre de réplica;
4. Récupération des journaux de la suite( Monitor via Filebeat)

5. Gestion des certificats
   1. Génération de requêtes pour signature externe;
   2. Génération de certificat auto-signé via Ansible.

6. Gestion pare-feu

# Notes diverses

## Architecture Hot Warm et réplica

Un réplica d'un index hot, peut-être stocké que sur un noeud Hot.
De la même façon un réplica d'un warm, ne peut être stocké que sur un noeud warm.
Il faut donc faire attention sur une petite architecture:
 - D'avoir au moin deux noeuds de chaque type (si réplica = 1);
 - Ne pas faire de distinction hot / warm si il n'y a pas vraiment d'utilité (même stockage, même compute etc.)