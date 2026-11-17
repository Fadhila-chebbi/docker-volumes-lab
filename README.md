\# TP Docker – Volumes et Persistance



\## 1. Objectif du TP



Ce TP a pour objectif de comprendre les différents mécanismes de stockage dans Docker et de vérifier la persistance des données.



\## 2. Environnement de travail



\- Système d'exploitation : Windows

\- Docker Desktop

\- PostgreSQL 16 Alpine

\- Nginx 1.27 Alpine

\- Alpine 3.20

\- Terminal : PowerShell



\## 3. Étape 1 – Perte des données sans volume nommé



Création d'un conteneur PostgreSQL sans volume nommé.



Création d'une table `notes` et insertion de la valeur `premiere note`.



Après suppression et recréation du conteneur, la requête SQL retourne :



```text

ERROR: relation "notes" does not exist

```



\*\*Résultat :\*\* les données ne sont pas retrouvées dans le nouveau conteneur.



\## 4. Étape 2 – Persistance avec un volume nommé



Création du volume nommé :



```powershell

docker volume create pgdata

```



Lancement de PostgreSQL avec le volume :



```powershell

docker run -d --name db-lab `

\-e POSTGRES\_USER=etudiant `

\-e POSTGRES\_PASSWORD=motdepasse-tp `

\-e POSTGRES\_DB=cours `

\-v pgdata:/var/lib/postgresql/data `

postgres:16-alpine

```



Après suppression et recréation du conteneur avec le même volume, la requête SQL retourne :



```text

&#x20;id |     texte

\----+---------------

&#x20; 1 | premiere note

```



\*\*Résultat :\*\* les données sont conservées grâce au volume nommé `pgdata`.



\## 5. Étape 3 – Inspection du volume



Commande utilisée :



```powershell

docker volume inspect pgdata

```



Mountpoint :



```text

/var/lib/docker/volumes/pgdata/\_data

```



Avec Docker Desktop, ce chemin appartient à l'environnement Linux interne de Docker.



\## 6. Étape 4 – Bind Mount avec Nginx



Création d'une page HTML dans le dossier local `site`.



Lancement de Nginx avec un Bind Mount en lecture seule :



```powershell

docker run -d --name web-lab -p 8083:80 -v "${PWD}/site:/usr/share/nginx/html:ro" nginx:1.27-alpine

```



Test d'accès :



```powershell

curl.exe http://localhost:8083

```



Modification du fichier HTML sur Windows : la nouvelle version apparaît immédiatement dans Nginx, sans redémarrage du conteneur.



Test d'écriture :



```powershell

docker exec web-lab sh -c "echo test > /usr/share/nginx/html/test.txt"

```



Résultat :



```text

Read-only file system

```



\*\*Résultat :\*\* le Bind Mount permet le partage des fichiers entre l'hôte et le conteneur. L'option `:ro` interdit l'écriture depuis le conteneur.



\## 7. Étape 6 – Stockage temporaire avec tmpfs



Création du conteneur :



```powershell

docker run -d --name temp-lab --tmpfs /cache alpine:3.20 sleep 3600

```



Création et lecture d'un fichier :



```powershell

docker exec temp-lab sh -c "echo donnees > /cache/f.txt"

docker exec temp-lab cat /cache/f.txt

```



Redémarrage du conteneur :



```powershell

docker restart temp-lab

```



Vérification du contenu :



```powershell

docker exec temp-lab ls -l /cache

```



Résultat :



```

total 0

```



\*\*Résultat :\*\* les données stockées dans tmpfs disparaissent après le redémarrage du conteneur.





\## 8. TP 5.2 – Sauvegarde et Restauration d'un volume Docker



\### 8.1 Préparation des données



Utilisation du volume `pgdata` avec PostgreSQL.



Insertion de trois lignes dans la table `notes` :



```powershell

docker exec db-lab psql -U etudiant -d cours -c "INSERT INTO notes (texte) VALUES ('deuxieme note'), ('troisieme note');"

```



Vérification :



```text

&#x20;count

\-------

&#x20;    3

```



\### 8.2 Arrêt du conteneur PostgreSQL



```powershell

docker stop db-lab

```



\### 8.3 Création de la sauvegarde



Création d'une archive compressée du volume `pgdata` :



```powershell

docker run --rm -v pgdata:/data:ro -v "${PWD}/sauvegardes:/backup" alpine:3.20 tar czf /backup/pgdata.tar.gz -C /data .

```



Vérification de l'archive :



```powershell

Get-Item .\\sauvegardes\\pgdata.tar.gz | Select-Object Name, Length

```



\*\*Résultat :\*\* le fichier `pgdata.tar.gz` a été créé avec succès.



\### 8.4 Simulation de perte des données



Suppression du conteneur :



```powershell

docker rm db-lab

```



Suppression du volume original :



```powershell

docker volume rm pgdata

```



\*\*Résultat :\*\* le volume original a été supprimé. La sauvegarde reste disponible dans le dossier `sauvegardes`.



\### 8.5 Restauration dans un nouveau volume



Création du volume de restauration :



```powershell

docker volume create pgdata-restaure

```



Extraction de l'archive dans le nouveau volume :



```powershell

docker run --rm -v pgdata-restaure:/data -v "${PWD}/sauvegardes:/backup" alpine:3.20 tar xzf /backup/pgdata.tar.gz -C /data

```



Lancement du nouveau conteneur PostgreSQL :



```powershell

docker run -d --name db-lab `

\-e POSTGRES\_USER=etudiant `

\-e POSTGRES\_PASSWORD=motdepasse-tp `

\-e POSTGRES\_DB=cours `

\-v pgdata-restaure:/var/lib/postgresql/data `

postgres:16-alpine

```



Vérification des journaux :



```text

PostgreSQL Database directory appears to contain a database; Skipping initialization

database system is ready to accept connections

```



\*\*Résultat :\*\* PostgreSQL a retrouvé la base de données existante sans effectuer une nouvelle initialisation.



\### 8.6 Vérification de l'intégrité des données



Vérification du nombre de lignes :



```powershell

docker exec db-lab psql -U etudiant -d cours -c "SELECT count(\*) FROM notes;"

```



Résultat :



```text

&#x20;count

\-------

&#x20;    3

```



Vérification du contenu :



```text

&#x20;id |     texte

\----+----------------

&#x20; 1 | premiere note

&#x20;34 | deuxieme note

&#x20;35 | troisieme note

```







