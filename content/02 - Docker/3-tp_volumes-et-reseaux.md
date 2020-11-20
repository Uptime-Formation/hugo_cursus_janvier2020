---
title: TP 3 - Volumes et réseaux
weight: 35
---

<!--
#TODO
# change network name to moby-network and add a schematics for clarity
# Add explanation on redis functionning (in RAM db => create a dump.rdb file only used when restarted) : https://redis.io/topics/persistence
# Redis need to restart to update from file stored in volume.
-->

## Portainer

- Pour visualiser aisément notre environnement docker au fur et à mesure de nos TPs nous allons charger une interface web d'administration docker appelée `portainer` et qui s'installe elle-même avec Docker.

```bash
docker volume create portainer_data
docker run --detach --name portainer \
    -p 9000:9000 \
    -v portainer_data:/data \
    -v /var/run/docker.sock:/var/run/docker.sock \
    portainer/portainer-ce
```

Si vous aviez déjà créé le conteneur Portainer, vous pouvez le relancer en faisant `docker start portainer`

- Remarque sur la commande précédente : pour que Portainer puisse fonctionner et contrôler Docker lui-même depuis l'intérieur du conteneur il est nécessaire de lui donner accès au socket de l'API Docker de l'hôte grâce au paramètre `--mount` ci-dessus.

- Visitez ensuite la page [http://localhost:9000](http://localhost:9000) pour accéder à l'interface.
- Créez votre user admin avec le formulaire.
- Explorez l'interface de Portainer.
- Créez un conteneur

## Docker networking

Pour expérimenter avec le réseau, nous allons lancer une petite application nodejs d'exemple (moby-counter) qui fonctionne avec une file (_queue_) redis (comme une base de données mais pour stocker des paires clé/valeur simples).

Récupérons les images depuis Docker Hub:

- `docker image pull redis:alpine`
- `docker image pull russmckendrick/moby-counter`

- Lancez la commande `ip a | tee /tmp/interfaces_avant.txt` pour lister vos interfaces réseau et les écrire dans le fichier

Pour connecter les deux applications créons un réseau manuellement:

- `docker network create moby-network`

Docker implémente ces réseaux virtuels en créant des interfaces. Lancez la commande `ip a | tee /tmp/interfaces_apres.txt` et comparez (`diff /tmp/interfaces_avant.txt /tmp/interfaces_apres.txt`). Qu'est-ce qui a changé ?

Maintenant, lançons les deux applications en utilisant notre réseau :

- `docker run -d --name redis --network <réseau> redis:alpine`
- `docker run -d --name moby-counter --network <réseau> -p 80:80 russmckendrick/moby-counter`

- Visitez la page de notre application. Qu'en pensez vous ? Moby est le nom de la mascotte Docker 🐳 😊. Faites un motif reconnaissable en cliquant.

Comment notre application se connecte-t-elle au conteneur redis ? Elle utilise ces instructions JS dans son fichier `server.js`:

```javascript
var port = opts.redis_port || process.env.USE_REDIS_PORT || 6379;
var host = opts.redis_host || process.env.USE_REDIS_HOST || "redis";
```

En résumé par défaut, notre application se connecte sur l'hôte `redis` avec le port `6379`

Explorons un peu notre réseau Docker.

- Exécutez (`docker exec`) la commande `ping -c 3 redis` à l'intérieur de notre conteneur applicatif (`moby-counter` donc). Quelle est l'adresse IP affichée ?

```bash
docker exec moby-counter ping -c3 redis
```

- De même, affichez le contenu des fichiers `/etc/hosts` du conteneur (c'est la commande `cat` couplée avec `docker exec`). Nous constatons que Docker a automatiquement configuré l'IP externe **du conteneur dans lequel on est** avec l'identifiant du conteneur. De même, affichez `/etc/resolv.conf` : le résolveur DNS a été configuré par Docker. C'est comme ça que le conteneur connaît l'adresse IP de `redis`. Pour s'en assurer, interrogeons le serveur DNS de notre réseau `moby-network` en lançant la commande `nslookup redis 127.0.0.11` toujours grâce à `docker exec` :
  `docker exec moby-counter nslookup redis 127.0.0.11`

- Créez un deuxième réseau `moby-network2`
- Créez une deuxième instance de l'application dans ce réseau : `docker run -d --name moby-counter2 --network moby-network2 -p 9090:80 russmckendrick/moby-counter`
- Lorsque vous pingez `redis` depuis cette nouvelle instance `moby-counter2`, qu'obtenez-vous ? Pourquoi ?

Vous ne pouvez pas avoir deux conteneurs avec les mêmes noms, comme nous l'avons déjà découvert.
Par contre, notre deuxième réseau fonctionne complètement isolé de notre premier réseau, ce qui signifie que nous pouvons toujours utiliser le nom de domaine `redis`. Pour ce faire, nous devons spécifier l'option `--network-alias` :

- Créons un deuxième redis avec le même domaine: `docker run -d --name redis2 --network moby-network2 --network-alias redis redis:alpine`

- Lorsque vous pingez `redis` depuis cette nouvelle instance de l'application, quelle IP obtenez-vous ?

- Récupérez comme auparavant l'adresse IP du nameserver local pour `moby-counter2`.

- Puis lancez `nslookup redis <nameserver_ip>` dans le conteneur `moby-counter2` pour tester la résolution de DNS.

- Vous pouvez retrouver la configuration du réseau et les conteneurs qui lui sont reliés avec `docker network inspect moby-network2`.
  Notez la section IPAM (IP Address Management).

- Arrêtons nos conteneurs : `docker stop moby-counter2 redis2`.

- Pour faire rapidement le ménage des conteneurs arrêtés lancez `docker container prune`.

- De même `docker network prune` permet de faire le ménage des réseaux qui ne sont plus utilisés par aucun conteneur.

## Volumes Docker

Pour ne pas interférer avec la deuxième partie du TP :

- Stoppez tous les conteneurs redis et moby-counter avec `docker stop` ou avec Portainer.
- Supprimez les conteneurs arrêtés avec `docker container prune`
- Lancez `docker volume prune` pour faire le ménage de volume éventuellement créés dans les TPs précédent
<!-- - Lancez `docker volume ls` pour vérifier qu'aucun volume n'est créé (sauf `portainer_data` si vous utilisez encore Portainer) sinon supprimez-les avec `docker volume rm --force <id_volume>` -->
- Lancez aussi `docker network prune` pour nettoyer les réseaux inutilisés

Passons à l'exploration des volumes:

- Recréez le réseau `moby-network` et les conteneurs `redis` et `moby-counter` à l'intérieur :

```bash
docker network create moby-network
docker container run -d --name redis --network moby-network redis
docker container run -d --name moby-counter --network moby-network -p 8000:80 russmckendrick/moby-counter
```

- Visitez votre application dans le navigateur. Faites un motif reconnaissable en cliquant.

- supprimez le conteneur `redis` : `docker rm --force redis`

- Visitez votre application dans le navigateur. Elle est maintenant déconnectée de son backend.

- Recréez le conteneur `redis` dans le réseau `moby-network`

- Rechargez la page. Que s'est-il passé ?

- Supprimez le conteneur `redis` avec `docker rm --force redis`
- Lancez `docker volume prune` pour faire le ménage et supprimer les volumes non utilisés

### Récupérer un volume d'un conteneur supprimé

- Faites de nouveau un motif reconnaissable dans l'application web

- Puis, **supprimez à nouveau** le conteneur redis.

- Avons-nous vraiment perdu les données de notre conteneur précédent ? Non !
  Le Dockerfile pour l'image officielle Redis ressemble à ça :

```Dockerfile
FROM alpine:3.5

RUN addgroup -S redis && adduser -S -G redis redis
RUN apk add --no-cache 'su-exec>=0.2'
ENV REDIS_VERSION 3.0.7
ENV REDIS_DOWNLOAD_URL http://download.redis.io/releases/redis-3.0.7.tar.gz
ENV REDIS_DOWNLOAD_SHA e56b4b7e033ae8dbf311f9191cf6fdf3ae974d1c
RUN set -x \
    && apk add --no-cache --virtual .build-deps \
        gcc \
        linux-headers \
        make \
        musl-dev \
        tar \
    && wget -O redis.tar.gz "$REDIS_DOWNLOAD_URL" \
    && echo "$REDIS_DOWNLOAD_SHA *redis.tar.gz" | sha1sum -c - \
    && mkdir -p /usr/src/redis \
    && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
    && rm redis.tar.gz \
    && make -C /usr/src/redis \
    && make -C /usr/src/redis install \
    && rm -r /usr/src/redis \
    && apk del .build-deps

RUN mkdir /data && chown redis:redis /data
VOLUME /data
WORKDIR /data
COPY docker-entrypoint.sh /usr/local/bin/
RUN ln -s usr/local/bin/docker-entrypoint.sh /entrypoint.sh # backwards compat
ENTRYPOINT ["docker-entrypoint.sh"]
EXPOSE 6379
CMD [ "redis-server" ]
```

Notez que, vers la fin du fichier, il y a deux instructions VOLUME et WORKDIR ; cela signifie que lorque notre conteneur a été lancé, un volume "caché" a effectivement été créé par Docker.

Beaucoup de conteneurs Docker sont des applications *stateful*, c'est-à-dire qui stockent des données. Automatiquement ces conteneurs créent des volument anonymes en arrière plan qu'il faut ensuite supprimer manuellement (avec rm ou prune).

- Inspectez la liste des volumes (par exemple avec Portainer) pour retrouver l'identifiant du volume caché. Normalement il devrait y avoir un volume `portainer_data` (si vous utilisez Portainer) et un volume anonyme avec un hash.

- Créez un nouveau conteneur redis en le rattachant au volume redis "caché" que vous avez retrouvé (en copiant l'id du volume anonyme) :
  `docker container run -d --name redis -v <volume_id>:/data --network moby-network redis:alpine`

- Visitez la page de l'application. Normalement un motif de logos _moby_ d'une précédente session devrait s'afficher (après un délai)

- Affichez le contenu du volume avec la commande : `docker container exec redis ls -lhat /data`

### Bind mounting

Finalement, nous allons écraser ce volume anonyme par le nôtre.

La bonne façon de créer des volumes consiste à les créer manuellement (volumes nommés) : `docker volume create redis_data`.

- Supprimez l'ancien conteneur `redis` puis créez un nouveau conteneur attaché à ce volume nommé : `docker container run -d --name redis -v redis_data:/data --network moby-network redis:alpine`

Lorsqu'un répertoire hôte spécifique est utilisé dans un volume (la syntaxe `-v HOST_DIR:CONTAINER_DIR`), elle est souvent appelée **bind mounting**.
C'est quelque peu trompeur, car tous les volumes sont techniquement "bind mounted". La différence, c'est que le point de montage est explicite plutôt que caché dans un répertoire géré par Docker.

- Lancez `docker volume inspect redis_data`.

<!-- ### Deux conteneurs Redis sur un seul volume

- Créez un réseau `moby-network2` et ajoutez un deuxième conteneur `redis2` qui va partager les même données que le premier :
  - situé à l'intérieur du nouveau réseau (`moby-network2`) comme à la partie précédent.
  - utilisant l'option `--network-alias redis` pour pouvoir être joignable par `moby-counter2` (que nous n'avons pas encore créé).
  - partageant le volume de données du premier (cf. cours)
      - monté en read-only (`:ro` après le paramètre de la question précédente)

{{% expand "Indice :" %}}
`docker run -v redis_data:/data -d --name redis2 --network moby-network2 --network-alias redis redis:alpine`
{{% /expand %}}

Le read-only est nécessaire pour que les deux Redis n'écrivent pas de façon contradictoire dans la base de valeurs.

- Ajoutez une deuxième instance de l'application dans le deuxième réseau connectée à ce nouveau Redis.

- Visitez la deuxième application : vous devriez voir également le motif de moby apparaître. -->

### Supprimer les volumes et réseaux

- Pour nettoyer tout ce travail, arrêtez d'abord les différents conteneurs `redis` et `moby-counter`.

- Lancez la fonction `prune` pour les conteneurs d'abord, puis pour les réseaux, et enfin pour les volumes.

Comme les réseaux et volumes n'étaient plus attachés à des conteneurs en fonctionnement, ils ont été supprimés.

**_Généralement, il faut faire beaucoup plus attention au prune de volumes (données à perdre) qu'au `prune` de conteneurs (rien à perdre car immutable et en général dans le registry)._**

### Facultatif : utiliser `VOLUME` avec `microblog`

- Rendez-vous dans votre repértoire home en tapant `cd`.
- S'il existe, renommez le dossier `microblog` précédemment créé en `microblog_tp2` (via l'explorateur de fichiers ou via `mv`).
- Puis, clonez une version déjà dockerisée de l'application `microblog` en lançant `git clone https://github.com/uptime-formation/microblog`.
- Après être entré·e dans le repo `microblog` grâce à `cd microblog`, récupérez le contenu de la branche Git `docker` en faisant `git checkout docker`.
- Lire le `Dockerfile` de l'application `microblog`.


La ligne de code Python qui nous permet de déterminer comment l'app utilise le volume du Dockerfile est la suivante :

`config.py` :

```python
SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or \
 'sqlite:///' + os.path.join(basedir, 'data/app.db')

```

Un volume apparaît comme un dossier à l'intérieur du conteneur, la variable d'environnement `DATABASE_URL`, qui indique à l'app où est la base de données, doit donc indiquer un fichier présent dans le dossier monté.

- À l'aide de la lecture de ce `Dockerfile`, utiliser un volume nommé pour stocker la base de données SQLite de l'application.
- Vérifier que le volume nommé est bien utilisé en branchant un deuxième conteneur `microblog` utilisant le même volume nommé.

{{% expand "Indice :" %}}

Dans le conteneur, le chemin du dossier à monter contenant la base de données est :
`/home/microblog/data`

{{% /expand %}}

<!-- Marquer solution -->

### Facultatif : `microblog` avec MySQL

Lire le `Dockerfile` de l'application `microblog` à l'adresse `https://github.com/uptime-formation/microblog` (branche `docker`) du TP précédent pour le lancer dans le même réseau qu'un conteneur `mysql` lancé avec les bonnes options de configuration.


{{% expand "Indice 1 :" %}}

La ligne du `Dockerfile` qui nous intéresse est la suivante :

```Dockerfile
ENV DATABASE_URL=mysql+mysqlconnector://microblog:${MYSQL_PASSWORD}@db/microblog
```

Il faut donc remplacer la variable `DATABASE_URL` au lancement.

{{% /expand %}}

{{% expand "Indice 2 :" %}}

Il va falloir configurer des options de démarrage pour le conteneur `mysql`, à lire sur le [Docker Hub](https://hub.docker.com/).

{{% /expand %}}
