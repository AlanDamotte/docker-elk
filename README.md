Docker ELK (Elasticsearch - Logstash - Kibana) pour la centralisation de logs
=============================================================================
*Auteur : Alan DAMOTTE <alan.damotte@gmail.com>*
***
>Bonjour et bienvenue dans ce tutoriel d'installation d'une stack ELK sur un environnement Docker. Le but ici est de déployer rapidement l'outil et n'avoir à se concentrer que sur la réalisation des fichiers de configurations de Logstash. Ainsi, en seulement quelques heures, vous devriez pouvoir tester ELK sur vos fichiers de logs à partir de configurations relativement simplistes.

Installation de Docker
----------------------

1. Télécharger et installer [Docker](https://docs.docker.com/engine/installation/) 
2. Télécharger et installer [Docker Compose](https://docs.docker.com/compose/install/)
3. Créer un dossier /workspace qui servira de dossier partagé avec la vm
4. Télécharger les fichiers de configuration présents sur le dépôt et les déposer dans le workspace défini précédemment dans un dossier /workspace/Docker par exemple. Dans le cas de l'utilisation d'un autre nom, adapter la suite du tutoriel.

Récupération des fichiers de logs
---------------------------------

* Placer les logs à analyser dans un dossier /workspace/logs
* Au lancement de la stack ELK, un data-volume est monté entre le host et le conteneur Logstash. Tous les logs présents dans /workspace/logs se retrouvent dans le dossier /logs à l'intérieur du conteneur. Il est possible de rajouter des logs depuis le host après le lancement de la stack, ils seront automatiquement répliqués dans le conteneur.

Configuration de Logstash
-------------------------
* Les fichiers configurations de Logstash sont à créer/modifier dans le dossier

    ```
    /workspace/Docker/dockerfiles/logstash/conf/
    ```

* Il est conseillé d'utiliser un fichier de configuration différent pour chaque filtre. J'utilise, par exemple, cette convention :
    > De 01 à 09 pour les inputs (Exemple : 01-input.conf)

    > De 10 à 29 pour les filtres (Exemple : 10-filtre.conf)
    
    > A partir de 30 pour les outputs (Exemple : 30-output.conf)

### Définition de l'input
* Si utilisation de plusieurs fichiers de logs, la mise en place d'une arborescence de fichier est recommandée.
* Lecture depuis un fichier 

    ```
    input {
        file {
    		type => "<Le type de log>"
    		path => "<Le chemin d'accès vers le fichier de logs (/logs/...)>"
    		# Les 2 lignes suivantes sont utiles pour des tests en local, elles permettent à chaque lancement de la stack de placer le pointeur de lecture en début de fichier.
    		start_position => "beginning"
    		sincedb_path => "/dev/null"
    		# La définition de tags est utile pour la suite de la configuration de logstash. Ils permettront aussi dans Kibana d'effectuer des recherches plus rapidement
    		tags => ["tag1","tag2",...]
    	}
    	...
    }
    ```

### Définition des filtres

* Pour cette partie il n'existe pas réellement de recette miracle, et c'est à priori le seul point sur lequel il va vous falloir travailler. Je vais néanmoins donner quelques conseils en montrant, par exemple, comment parser des logs apache.
1. C'est le moment d'utiliser les tags définis précédemment. Si 2 fichiers de logs provenants de machines différentes contiennent des logs Apache, ils devraient contenir les tags "apache" et "machine1" (respectivement "apache" et "machine2"). Il est alors possible de définir un filtre générique qui parsera ces logs Apache.

    ```
     filter {
        if "apache" in [tags]{
            grok {
			    match => { "message" => "%{COMBINEDAPACHELOG}"}
		    }
		    date {
			locale => "en"
			match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
			target => "@timestamp"
			timezone => "Europe/Paris"
		    }
        }
     }
     ```
* Le filtre "grok" permet de parser une ligne de log et associer une valeur à une clé. Vous trouverez à l'[adresse suivante](https://github.com/elastic/logstash/blob/v1.4.2/patterns/grok-patterns) la liste des patterns grok prédéfinis et utilisables directement.
* La partie "date" permet de faire correspondre le champ @timestamp avec la date figurant sur le logs. Ainsi, sur Kibana, le log apparaît bien à l'heure à laquelle il a été produit et non pas celle de son analyse par logstash. De plus, il est intéressant de préciser la timezone pour éviter les décalages entre l'heure du log et celle affichée dans Kibana. 
* Pour tester ses patterns grok en ligne ca se passe [ici](http://grokconstructor.appspot.com/do/match)

2. Modification des champs : Un des plugins intéressant lors de la réalisation de filtre peut être le [mutate](https://www.elastic.co/guide/en/logstash/current/plugins-filters-mutate.html). Il permet entre autres d'ajouter des tags, ajouter/modifier/supprimer des champs...
3. (Optionnel) Présence de logs multilignes, deux possibilités :
    * Concaténation du log multiligne en entrée (input) : 

        ```
        file {
    		...
    		codec => multiline { 
    			pattern => "<pattern>"
    			what => "previous"
    			negate => true
            }
    		...
    	}
        ```
    
    Ici ce qui est fait : Tant qu'une ligne ne commence pas par le pattern défini (par exemple une date), alors elle est concaténée à la précédente. Il existe plusieurs façons de parser les logs multilignes, pour plus d'info voir la [documentation officielle](https://www.elastic.co/guide/en/logstash/current/plugins-codecs-multiline.html).
    * Concaténation du log multiligne dans le filtre :

        ```
        filter{
            ...
        	multiline {
        		pattern => "<pattern>"
        		what => "previous"
        		negate => true
            }
            ...
        }
        ```

* **NB** : Pour des raisons qui sont, pour moi, encore floues, l'utilisation du filtre multiligne avec la lecture depuis des fichiers en input semble poser problème (logs multilignes tronqués dans certains cas, je soupçonne le buffer de lecture...). J'utilise donc personnellement le codec multiligne directement en input. Cependant il faut savoir que depuis la version 1.5 de Logstash, le flushing automatique, permettant de ne pas oublier le dernier log multiligne, n'est implémenté que pour le filtre. Si vous décidez donc d'utiliser le codec, il se peut que le dernier log de votre fichier, si c'est un log multiligne, ne passe pas dans les filtres.

### Définition de l'output
* Le fichier d'output fourni devrait être suffisant pour envoyer les logs vers Elasticsearch. Aucune modification n'est donc nécessaire.
* Il est toutefois possible d'utiliser des plugins en cas de besoins. Par exemple, il peut être judicieux de conserver les logs qui n'ont pas correctement passés le filtre (ce qui est probablement dû à un problème de pattern grok) et qui contiennent le tag "_grokparsefailure".

    ```
    output {
    	if "_grokparsefailure" in [tags] {
    		file { path => "<chemin>/failed_log-%{+YYYY-MM-dd}" }
    	}
    	...
    }
    ```

Il faut cependant dans ce cas définir où écrire ces logs et modifier la configuration de Docker. N'étant pas réellement lié au sujet principal, je ne vais pas m'étendre plus sur ce point. *Vous pouvez toutefois me contacter pour que je vous aiguille.*

### Définition des fichiers de configuration dans le Dockerfile Logstash

1. Editer le fichier /workspace/Docker/dockerfiles/logstash/Dockerfile
2. Modifier les lignes correspondant à l'ajout des fichiers de configurations afin de prendre en compte les nouveaux fichiers de configurations définis précédemment.

    ```
    ADD conf/01-input.conf /etc/logstash/conf.d/
    ...
    ADD	conf/30-output.conf /etc/logstash/conf.d/
    ```

Analyse des logs
----------------
	
1. Lancer un build, les images ELK sont téléchargées et installées :

    ```
	cd /workspace/Docker
	docker-compose build
	```
	
2. A la fin de l'installation

	```
	docker-compose up
	```

    La stack devrait se lancer et après quelques instants les premiers logs devraient être parsés
3. Se rendre à l'adresse http://127.0.0.1:5082 pour accéder à Kibana
4. Se rendre à l'adresse http://127.0.0.1:9200/_plugin/marvel pour accéder au plugin marvel d'Elasticsearch

Utilisation de Kibana
---------------------
Je vais essayer dans cette partie d'expliquer les fonctionalités principales (mais aussi les plus basiques) de Kibana.
1. Après le lancement de la stack ELK, vous devriez pouvoir accéder à Kibana. La première page sur laquelle vous allez tomber est la partie "Settings". Il s'agit simplement de dire à Kibana quel format d'index il doit chercher. Si des logs ont correctement été parsés, un simple rafraichissement de la page devrait vous permettre de créer l'index. Vous aurez ainsi accès aux différents champs créés, automatiquement ou via vos filtres, par logstash lors du parsing des logs.
2. Vous pouvez désormais vous rendre dans la partie "Discover". Pour pouvoir visualiser vos logs, choisir en haut à droite de la page, la plage horaire de recherche en fonction de la date de vos logs. Vous devriez alors normalement voir vos logs. Il est possible d'effectuer de recherche directment via des requêtes, ou en utilisant la table de gauche. Par exemple pour recherche les logs "apache" de la "machine1", il est possible d'effectuer cette requête :

    ```
     tags:(apache AND machine1)
    ```

* Un autre façon est de dérouler le champs "tags" de la table (à gauche) et sélectionner les tags manuellement. Cependant, par défaut, Kibana affiche les 5 champs les plus représentés dans les 500 derniers logs. Il est possible d'augmenter cette limite dans les paramètres de Kibana. Je préfère personnellement passer par les requêtes.
3. La partie "Visualisation" vous permet de créer des graphes en fonction d'une ou plusieurs recherches. La partie "Dashboard" permet de rassembler et organiser plusieurs visualisations sur un même tableau de bord. Concernant ces deux parties, il est plus simple de découvrir par soit-même.

**NB** : Dans la partie "settings", il est possible de rafraichir les champs disponibles (certains n'étant pas existants lors du premier lancement de Kibana). Cependant, lors du rafraichissement, Kibana recupère les champs présents sur les 5 derniers index d'Elasticsearch. Certains champs peuvent ne pas être disponible immédiatement en fonction de la quantité de logs à parser.

