# Construction d'un environnement avec Docker et Traefik

## Contexte

Nous allons construire un site web contenant le backend d'un blog, le backend d'un catalogue produit et une admin capable d'administrer les 2 backend.
Dans le monde du microservice, rien de plus normal. 3 fonctionnalités distinctes, 3 briques distinctes. 

Nous allons utiliser api-platform pour simplifier la partie "développement" à proprement parler et se concentrer sur l'infrastructure sur le poste de dev. 

### Et sans Traefik ?

#### Dans l'environnement de dev

Sans cet article, vous seriez bien entendu tenté d'utiliser différents ports. Avec une liste à maintenir de mapping container <-> host.

* blog : 
	* blog_php : localhost:9000
	* blog_nginx API : localhost:8080
	* blog_pgsql : localhost:5432
	* blog_varnish API : localhost:80
* catalogue :
	* catalog_php : localhost:9001
	* catalog_nginx API : localhost:8081
	* catalog_pgsql : localhost:5433
	* catalog_varnish API : localhost:81
* admin :
	* admin_nginx : localhost:82


Est-ce que le port 8082 est pris ? A-t-on déjà un mysql déclaré sur le port 3306? 
Personne n'en sait plus rien. Et nous ne sommes qu'au début de l'application microservice !

Si j'ajoute à cela le fait que depuis l'extérieur, l'url du backend blog sera "localhost:80" (pour atteindre le varnish), mais depuis les containers l'URL sera 'blog_varnish' par exemple. De quoi vous assurer des sueurs froides pendant le debug... (Et une belle liste de favoris...)

De plus, si vous développez d'autres applications "legacy" à côté, vous devrez les stopper pour libérer le port 80 ou venir les mapper sur des ports disponibles (A ajouter bien entendu au "manifest" présent au dessus)

#### Dans l'environnement de preprod/prod

Alors la, rien de plus simple ! 
Un seul Apache/Nginx, des vhost/sites différents, des pools php par site (ou pas), une bdd partagée. Et un serveur qui fonctionne, mais complexe à maintenir/mettre à jour. 

### Ok. Et Traefik ?

Traefik est un routeur de trafic. C'est "tout simplement" un reverse proxy & load balancer dynamique. 
Il est possible de l'installer et de le faire fonctionner simplement sous docker, kubernetes, Marathon, ... 

#### Dans l'environnement de dev 

En dev, traefik fait disparaitre les petits tracas du quotidien. C'est le seul point d'entrée de tout le trafic local, il est le seul à écouter les ports "classiques" (80/443/3306/...)
Et il vient permettre l'utilisation de noms de domaine en local. 

Le "mapping" devient ainsi un mapping de noms de domaine : 

* Blog : 
	* blog.localhost : blog_varnish:80
	* pgsql.blog.localhost : blog_pgsql:5432
	* php.blog.localhost : blog_php:9000
	* nginx.blog.localhost : blog_nginx:80

* Catalog
	* catalog.localhost : catalog_varnish:80
	* pgsql.catalog.localhost : catalog_pgsql:5432
	* php.catalog.localhost : catalog_php:9000
	* nginx.catalog.localhost : catalog_nginx:80

* Admin
	* admin.localhost : admin_nginx:80

"Et voilà !", plus de port farfelu, plus de conflit avec le legacy (il suffira de créer des noms de domaine dédiés). 
Finalement, qu'un container d'application se permette d'écouter sur le port 80 parait tout à coup assez présomptueux! 

Le découpage est simple, et évident. ([service].[application].[local])
Vous souhaitez ajouter un elasticsearch au catalogue ? Je vous laisse deviner le nom de domaine associé...  `elasticsearch.catalog.localhost` ! Un kibana ? `kibana.catalog.localhost`

Facile ! 


#### Et en QA/Preprod/Prod ?

Justement ! Le principe est exactement le même. On gomme déjà une différence entre le local et la QA, la preprod et la prod. 
Le déploiement est identique, les configuration docker sont les mêmes, les ports sont les mêmes, et ce sont ceux "par défaut" de chaque service (mysql, elasticsearch, kibana, ...)

Seuls les noms de domaines changent, et nous ne mapperont plus les ports des services à ne pas exposer. 

* blog.app_demo_traefik.obtao.com : blog_varnish:80
* catalog.app_demo_traefik.obtao.com : catalog_varnish:80
* admin.app_demo_traefik.obtao.com : admin_nginx:80

Terminé ! Maintenant que le plan de bataille est prêt, allons-y...


## Configurer Traefik 

Traefik est capable de se brancher sur ce que vous utilisez pour déployer vos containers. 
Pour l'instant nous utilisons docker-compose et docker, mais nous verrons par la suite qu'il réagit également très bien avec Kubernetes.

Pour l'installer, rien de plus simple : 
* Un docker-compose (pour plus de lisibilité)
* Un network docker (qui sera le réseau 'central' de notre application)
* Une configuration de Traefik : `traefik.toml`

### Traefik.toml

La configuration de Traefik est assez simple, si vous souhaitez aller plus loin, n'hésitez pas à aller voir la doc. 

```toml
defaultEntryPoints = ["http"]

[entryPoints]
    [entryPoints.http]
    address = ":80"

[docker]
exposedbydefault = false

[web]
address = ":8080"
```

Le trafic arrive sur le port http 80, sur tous les hosts. (et c'est le point d'entrée par défaut)
Les containers ne sont pas automatiquement exposés, et l'admin est accessible sur le port 8080. 


### Docker-compose.yml

```yaml
version: '3'

services:

  traefik:
    image: traefik:latest
    init: true
    command: --api --docker
    networks:
      local_traefik:
        aliases:
          - traefik
    ports:
      - '80:80'
      - '8080:8080'
    volumes:
      - ./src/traefik/rootfs/etc/traefik:/etc/traefik
      - /var/run/docker.sock:/var/run/docker.sock:ro

networks:
  local_traefik:
    external:
      name: local_traefik

``` 

Aucune subtilité ici.

**Réseau applicatif** 

> Pour la partie réseau, nous faisons le choix de créer un réseau docker par backend dans l'application (`blog` / `catalog`) et de créer un réseau commun (`app_demo_traefik`) dans lequel les briques peuvent communiquer. Cela va permettre 2 choses : si un service ne rejoint pas le réseau commun, il reste inaccessible, et les services peuvent porter des noms explicites à l'intérieur de leur réseau applicatif (`php`, `varnish`, `pgsql`)
> La configuration des applications reste ainsi simple `pg_host: pgsql` plutôt que `pg_host: blog_pgsql`. 

**Réseau local** 

> Pour le local, nous allons créer un réseau "local_traefik" que vos containers devront rejoindre pour être accessibles depuis l'extérieur. 


Allez-y `docker create network local_traefik && docker-compose up -d`, Traefik tourne et est prêt à recevoir les requêtes. 
Vous pouvez accéder à l'[admin de traefik](http://localhost:8080).

En bref, lorsque vous faites du développement il vous suffit de démarrer traefik pour profiter du reverse-proxy. 


Nous sommes prêts à lancer nos applications! 


## Configurer les applications

Après avoir sagement récupéré les sources de 2 api-platform, nous allons maintenant les brancher sur le traefik installé précédemment. 

Traefik fonctionne grâce aux labels des containers Docker.
Comme vous avez pu le remarquer, lors du démarrage de Traefik, nous montons la socker docker à l'intérieur du container. 
```
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro
```


Nous allons donc labelliser nos images docker pour configurer traefik. 
Voici un exemple avec le container nginx du backend `blog`

### La configuration avant modification

Telle qu'elle est récupérée sur le site api-platform.

```
cache-proxy:
  image: ${CONTAINER_REGISTRY_BASE}/varnish
  build:
    context: ./api
    dockerfile: Dockerfile.varnish
    cache_from:
      - ${CONTAINER_REGISTRY_BASE}/varnish
  depends_on:
    - api
  # Comment out this volume in production
  volumes:
    - ./api/docker/varnish/conf:/etc/varnish:ro
  ports:
    - "8081:80"
```

Comme nous l'avons déjà évoqué, elle est très figée et ne permet pas de développer facilement sur plusieurs applications en local. 


### Configuration après Traefik

```yaml
version: '3.2'

services: 
  
  [...]

  cache-proxy:
    image: ${CONTAINER_REGISTRY_BASE}/varnish
    build:
      context: ./api
      dockerfile: Dockerfile.varnish
      cache_from:
        - ${CONTAINER_REGISTRY_BASE}/varnish
    depends_on:
      - api
    # Comment out this volume in production
    volumes:
      - ./api/docker/varnish/conf:/etc/varnish:ro
    #ports:
    #  - "8081:80"
    labels:
      - 'traefik.enable=true'
      - 'traefik.frontend.rule=Host:blog.localhost'
      - 'traefik.docker.network=local_traefik'
      - 'traefik.port=80'
    networks:
      local_traefik:
        aliases:
          - app_demo_traefik_blog_varnish
      blog:
      	aliases:
      	  - varnish
      app_demo_traefik:
        aliases:
       	  - blog
  
[...]
networks:

  local_traefik:
    external:
      name: local_traefik

  app_demo_traefik:
    external:
      name: local_traefik

  blog: ~
```


Dans l'ordre : 
* `labels` : Nous ajoutons tous les labels nécessaire à la prise en compte par traefik. 
	* `enable=true` : on active traefik pour ce container
	* `frontend.rule=Host:blog.localhost` : On ajoute une règle au reverse proxy pour le Host `blog.localhost`
	* `docker.network=local_traefik` : Sur quel network docker ce container est joignable
	* `port` : Le port du container sur lequel traefik doit router les paquets qui correspondent à la règle `frontend.rule`
* `networks`: Tous les networks rejoints par le container. Vous n'êtes pas obligés de faire aussi complexe :-)
	* `local_traefik` : Discussion avec traefik, et routage depuis l'extérieur.
	* `app_demo_traefik` : Discussions des briques de l'application `app_demo_traefik`. Dans le cadre de cette application, notre container varnish est simplement le point d'accès vers le backend `blog`. Son nom est donc simplement `blog`. 
	* `blog`: C'est le réseau de la brique blog. Dans ce réseau, chaque container porte un nom décrivant son service. Ici :  `varnish`.


Tous les containers ont été modifié pour être configuré derrière traefik (ou pour certains, n'être disponible que dans le réseau `blog`)

Un petit `docker-compose up -d` dans le répertoire `blog` vous permettra de lancer l'application :-)
Une fois démarrée, la brique `blog` est détectée par traefik, et les paquets à destination de `blog.localhost` sont redirigés vers le container `varnish`.  

