# Projet fil rouge - Partie 1 : mettre en ligne un site

Hugo BERTON - B2 Info

Contexte : publier le site d'une petite entreprise (la "Boulangerie du Cours") avec nginx.
Tout est fait sur mon Mac avec Docker Desktop.

```
$ docker version --format 'Client: {{.Client.Version}} / Serveur: {{.Server.Version}} ({{.Server.Os}}/{{.Server.Arch}})'
Client: 29.8.1 / Serveur: 29.8.1 (linux/arm64)
```

---

## Étape 1 - Vérifier l'environnement

Conteneur alpine jetable (`--rm` le supprime tout seul à la fin) :

```
$ docker run --rm alpine:3.20 uname -r
7.0.14-linuxkit
```

Sur l'hôte :

```
$ uname -r
25.5.0
$ uname -s
Darwin
```

**Constat :** les deux résultats sont différents. Dans le conteneur on a un noyau Linux (`linuxkit`), alors que
sur l'hôte c'est le noyau de macOS (Darwin 25.5.0). Un conteneur n'a pas son propre noyau, il utilise celui de
la machine qui le fait tourner. Sur Mac, Docker Desktop lance une petite VM Linux (linuxkit) et les conteneurs
partagent le noyau de cette VM. Sur une machine Linux on aurait eu exactement le même `uname -r` des deux côtés.

---

## Étape 2 - Explorer un conteneur

```
$ docker run -it --name test ubuntu:24.04 bash
root@63f7df9d2d30:/# cat /etc/os-release | head -2
PRETTY_NAME="Ubuntu 24.04.5 LTS"
NAME="Ubuntu"
root@63f7df9d2d30:/#        <- Ctrl+P puis Ctrl+Q : on se détache sans arrêter le conteneur
```

```
$ docker ps
CONTAINER ID   IMAGE          COMMAND   CREATED                  STATUS                  PORTS     NAMES
63f7df9d2d30   ubuntu:24.04   "bash"    Less than a second ago   Up Less than a second             test
```

Le conteneur tourne toujours (`Up`) même si on n'est plus dedans.

```
$ docker attach test
root@63f7df9d2d30:/# echo de retour dans test
de retour dans test
root@63f7df9d2d30:/# exit       <- Ctrl+D
```

```
$ docker ps -a --filter name=^test$
CONTAINER ID   IMAGE          COMMAND   CREATED          STATUS                              PORTS     NAMES
63f7df9d2d30   ubuntu:24.04   "bash"    31 seconds ago   Exited (0) Less than a second ago             test
```

**Constat :** Ctrl+P Ctrl+Q détache le terminal mais laisse le conteneur tourner. Ctrl+D ferme bash, et comme
bash est le processus principal du conteneur, le conteneur s'arrête (`Exited (0)`). Il n'est pas supprimé : il
apparaît encore avec `docker ps -a`.

---

## Étape 3 - Lancer le serveur web

```
$ docker run -d --name site -p 8080:80 -e ENV=dev nginx:1.27
8f577f1ebd6943dc03bfab705c4ea6f2ca1d0ba39ba60240b2c3e688ccac4731
$ docker ps
CONTAINER ID   IMAGE        COMMAND                  CREATED         STATUS         PORTS                                     NAMES
8f577f1ebd69   nginx:1.27   "/docker-entrypoint.…"   2 seconds ago   Up 2 seconds   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   site
$ curl -s http://localhost:8080 | grep title
<title>Welcome to nginx!</title>
```

- `-d` : en arrière-plan
- `--name site` : nom du conteneur
- `-p 8080:80` : le port 8080 de mon Mac redirige vers le port 80 du conteneur
- `-e ENV=dev` : variable d'environnement

Sur http://localhost:8080 on voit la page par défaut "Welcome to nginx!".

---

## Étape 4 - Publier la page

J'ai écrit `index.html` (page de la boulangerie) dans `projet-site/` sur l'hôte, puis :

```
$ docker cp index.html site:/usr/share/nginx/html/index.html
$ docker cp site:/etc/nginx/nginx.conf ./nginx.conf
$ ls
index.html
nginx.conf
$ curl -s http://localhost:8080 | grep title
  <title>Boulangerie du Cours - Aix-en-Provence</title>
```

`docker cp` marche dans les deux sens : hôte -> conteneur pour la page, conteneur -> hôte pour récupérer
`nginx.conf`. La page de la boulangerie s'affiche maintenant sur http://localhost:8080 (pas besoin de
redémarrer nginx, il sert directement le nouveau fichier).

---

## Étape 5 - Observer

Logs (après avoir ouvert la page dans Chrome) :

```
$ docker logs site
...
192.168.65.1 - - [09/Oct/2026:11:45:46 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.7.1" "-"
192.168.65.1 - - [09/Oct/2026:11:45:46 +0000] "GET / HTTP/1.1" 200 2283 "-" "curl/8.7.1" "-"
192.168.65.1 - - [09/Oct/2026:11:45:53 +0000] "GET / HTTP/1.1" 200 2283 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/153.0.0.0 Safari/537.36" "-"
2026/10/09 11:45:54 [error] 29#29: *4 open() "/usr/share/nginx/html/favicon.ico" failed (2: No such file or directory), client: 192.168.65.1, server: localhost, request: "GET /favicon.ico HTTP/1.1", host: "localhost:8080", referrer: "http://localhost:8080/"
192.168.65.1 - - [09/Oct/2026:11:45:54 +0000] "GET /favicon.ico HTTP/1.1" 404 555 "http://localhost:8080/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/153.0.0.0 Safari/537.36" "-"
```

On retrouve mes requêtes : les deux `curl` (615 octets = page par défaut, 2283 octets = ma page) et celle de
Chrome (user agent Mozilla/Chrome). Le navigateur demande aussi `/favicon.ico`, qui n'existe pas, d'où le 404.
L'IP 192.168.65.1 est celle de la VM Docker Desktop, pas directement celle de mon Mac.

IP du conteneur et variable ENV :

```
$ docker inspect -f '{{.NetworkSettings.IPAddress}}' site
template parsing error: template: :1:18: executing "" at <.NetworkSettings.IPAddress>: map has no entry for key "IPAddress"
```

Avec ma version de Docker (29.8) ce champ n'existe plus, il faut passer par les réseaux du conteneur :

```
$ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' site
172.17.0.2
$ docker inspect -f '{{.Config.Env}}' site
[ENV=dev PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin NGINX_VERSION=1.27.5 NJS_VERSION=0.8.10 NJS_RELEASE=1~bookworm PKG_RELEASE=1~bookworm DYNPKG_RELEASE=1~bookworm]
```

Le conteneur a l'IP 172.17.0.2 sur le réseau bridge par défaut, et la variable `ENV=dev` est bien là (avec les
variables déjà définies par l'image nginx).

Fichiers servis :

```
$ docker exec site ls -l /usr/share/nginx/html
total 8
-rw-r--r-- 1 root root     497 Apr 16  2025 50x.html
-rw-r--r-- 1  501 dialout 2283 Oct  9 11:44 index.html
```

Mon `index.html` a remplacé celui de nginx (2283 octets). Il appartient à l'UID 501 : `docker cp` a gardé le
propriétaire du fichier sur mon Mac.

---

## Étape 6 - Arrêter et recréer

```
$ docker stop site
site
$ docker rm site
site
$ docker run -d --name site -p 8080:80 -e ENV=dev nginx:1.27
bdcebc70ed1e351c4798114fa67de6d50e1574dbe884feec2db1d61656aa4392
$ curl -s http://localhost:8080 | grep title
<title>Welcome to nginx!</title>
$ docker exec site ls -l /usr/share/nginx/html
total 8
-rw-r--r-- 1 root root 497 Apr 16  2025 50x.html
-rw-r--r-- 1 root root 615 Apr 16  2025 index.html
```

**Ma page n'est plus là**, on revient à "Welcome to nginx!".

**Pourquoi :** un conteneur = l'image (en lecture seule) + une couche en écriture qui lui est propre. Le
`docker cp` a écrit ma page dans cette couche du conteneur, pas dans l'image `nginx:1.27`. Quand on fait
`docker rm`, la couche du conteneur est supprimée avec lui. Le nouveau conteneur repart de l'image d'origine,
donc de la page par défaut.

Pour que la page reste il faudrait soit la mettre dans une image (Dockerfile avec `COPY`), soit la monter depuis
l'hôte (volume / bind mount). C'est ce qui est fait dans la partie 2.

---

## Résumé des commandes utilisées

| Commande | Rôle |
|---|---|
| `docker version` / `docker info` | infos sur le client et le démon |
| `docker run` | créer et démarrer un conteneur |
| `docker ps [-a]` | lister les conteneurs |
| `docker attach` | se rattacher au terminal d'un conteneur |
| `docker logs` | voir stdout / stderr d'un conteneur |
| `docker inspect` | voir la configuration détaillée |
| `docker exec` | lancer une commande dans un conteneur qui tourne |
| `docker cp` | copier des fichiers hôte <-> conteneur |
| `docker stop` / `docker rm` | arrêter / supprimer un conteneur |
