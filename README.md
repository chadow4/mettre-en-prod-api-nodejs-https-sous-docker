<br />
<div align="center">
    <img src="https://jolicode.com/media/original/2013/10/homepage-docker-logo.png" alt="Logo" width="250" height="200">
</div>

<h3 align="center">Mettre en production une API REST NodeJS avec HTTPS avec Docker (interface Portainer) </h3>

  <p align="center">
   Build, Ship and Run Any App, Anywhere
  </p>
</div>


> Avant de commencer ce tutoriel, il est important de noter que la mise en production d'une API peut se faire de différentes manières et que celle-ci n'est pas l'unique solution. En effet, il existe plusieurs approches pour déployer une API, chacune ayant ses avantages et inconvénients en fonction des besoins et des contraintes de l'application.


## Prérequis

Afin de mettre en place la mise en production de notre API, nous aurons besoin des éléments suivants :
- un serveur (VPS, dédié, etc.)
- Docker 
- Portainer
- une API NodeJS de votre choix (par exemple Vanilla express, NestJS...)
- un certificat SSL (nous utiliserons certbot pour le générer)
- un chemin pré-défini où nous stockerons les fichiers de notre API. Pour notre exemple, ce chemin est : ```/home/docker/api/tutorialAPI```

## Mise en place
### 1. Générer un certificat SSL avec Certbot:

1. Installer Certbot ```sudo apt install certbot``` (Ubuntu & debian) 

2. Exécuter la commande ```sudo certbot certonly --standalone -d example.com``` en remplaçant **example.com** par votre nom de domaine.
Suivre les instructions de Certbot pour générer le certificat et la clé privée.
A la fin du processus, si tout c'est bien passé, vous retrouverez votre certificat et votre clé privée dans : 
```
  /etc/letsencrypt/live/example.com/fullchain.pem:/app/cert.pem
  /etc/letsencrypt/live/example.com/privkey.pem:/app/key.pem
```

### 2. Préparer le terrain coté fichier avant d'initialiser un container docker 

1. Pour que tout fonctionne correctement, vous devez créer à la racine de votre API (dans mon cas, /home/docker/api/TutorialAPI) un script de lancement de l'application Node.js dans un fichier shell (.sh) comme suit :

```batch
npm install
node fichier.js \
    --tls-cert-path /app/cert.pem \
    --tls-key-path /app/key.pem \
    --port 3000
```
Ce script, nommé "start.sh", permet d'automatiser le lancement de l'application Node.js en installant les dépendances nécessaires et en configurant les options de communication sécurisée avec un certificat SSL. Il lance ensuite l'application en écoutant sur le port 3000. Ce fichier est placé à la racine de l'API (/home/docker/api/TutorialAPI dans mon cas) pour faciliter la gestion de l'application lors de son déploiement.

2. Vous aller devoir ajouter la configuration SSL à votre fichier principal de votre API:

#### Avec Express vanilla : 
dans votre index.js

```js
const fs = require('fs');
const https = require('https');
const express = require('express');

const app = express();

app.use(express.json());
app.use(express.urlencoded({extended: true}));

const sslOptions = {
  cert: fs.readFileSync('/app/cert.pem'),
  key: fs.readFileSync('/app/key.pem')
};

const server = https.createServer(sslOptions, app);

const PORT = process.env.PORT || 8080;
server.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

#### Avec NestJS :

dans votre main.ts
```ts
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import * as fs from "fs";

async function bootstrap() {
  const httpsOptions = {
    cert: fs.readFileSync('/app/cert.pem'),
    key: fs.readFileSync('/app/key.pem'),
  };
  const app = await NestFactory.create(AppModule, { httpsOptions });
  app.enableCors();
  await app.listen(3000);
}

bootstrap();
```

### 3. Utiliser le certificat dans Docker avec Node.js

Pour vous faciliter la tâche, nous allons utiliser Portainer, une interface graphique pour la gestion des conteneurs Docker. Voici les différentes étapes à suivre :

1. Ouvrez Portainer et connectez-vous à votre environnement Docker.
2. Dans le panneau de gauche, cliquez sur "Stacks".
3. Cliquez sur le bouton "Add stack" en haut à droite de l'écran.
4. Donnez un nom à votre stack.
5. Collez le contenu de votre fichier YAML ci-dessous dans le champ "Web editor".
6. Cliquez sur "Create the stack".

Ceci va créer une stack Docker avec les services et les configurations définis dans votre fichier YAML. Vous pouvez également surveiller les logs de votre stack et les paramètres de configuration dans l'interface de Portainer.

```yaml
version: '3'
services:
  tutorial:
    image: node:latest 
    ports:
      - "32000:3000"
    working_dir: /app 
    volumes:
      - /home/docker/api/tutorialAPI:/app
      - /etc/letsencrypt/live/example.com/fullchain.pem:/app/cert.pem
      - /etc/letsencrypt/live/example.com/privkey.pem:/app/key.pem
    command: sh /app/start.sh
```

Le fichier YAML est un fichier de configuration pour Docker. Il contient une liste de services qui sont des conteneurs Docker travaillant ensemble pour fournir une application complète.

Dans ce fichier, le service s'appelle "tutorial" et est basé sur l'image Docker "node:latest", qui est une image officielle de Node.js à jour. Le service est configuré pour exposer le port 3000 du conteneur et le mapper sur le port 32000 de la machine hôte.

De plus, le service est configuré pour utiliser le répertoire de travail "/app" dans le conteneur. Un répertoire local "/home/docker/api/tutorialAPI" est monté sur le répertoire de travail du conteneur, et les deux fichiers de certificats SSL sont également montés dans le conteneur pour permettre une communication sécurisée.

Enfin, lors du lancement du conteneur, la commande "sh /app/start.sh" est exécutée. Cette commande lancera le script défini précédemment[^1], installera les dépendances nécessaires à l'application et exécutera l'API avec des options de certificat SSL et de port.

Vous pouvez dès à présent lancer votre container Docker et vérifier si votre api fonctionne bien en allant pour notre cas sur https://example.com:32000

### 4. Bonus - Connecter son API à sa base de données de son serveur

Lorsque nous sommes en phase de développement, nous utilisons généralement l'adresse "localhost" pour accéder à notre base de données. 

En revanche, lors de la création d'une stack Docker, Docker crée automatiquement un réseau interne pour les conteneurs de la stack. Ce réseau est appelé "network overlay" et il est utilisé pour connecter les différents conteneurs de la stack ensemble. Chaque conteneur dans la stack obtient une adresse IP unique sur ce réseau. La stack possède également une passerelle IPv4 qui est l'adresse IP utilisée comme point de sortie pour les paquets de données sortant du réseau interne de la stack. C'est l'adresse IP que les conteneurs de la stack utilisent pour communiquer avec des réseaux externes ou le reste de votre machine.

- Si vous avez une base de données tournant dans un autre conteneur Docker, vous devrez préciser l'adresse IP de ce conteneur à la place de "localhost" dans votre API.
- Si vous avez une base de données tournant directement sur votre serveur, vous devrez remplacer "localhost" par l'adresse IP de la passerelle de votre stack :
  -  1. Dans le panneau de gauche, cliquez sur "Network".
  -  2. Récuperez l'ip passerelle de votre stack : ```IPV4 IPAM Gateway```

Voilà, j'espère que cela vous sera utile. N'hésitez pas à me poser d'autres questions si nécessaire.
