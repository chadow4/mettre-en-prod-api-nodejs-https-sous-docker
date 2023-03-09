<br />
<div align="center">
  <a href="https://i.ibb.co/ScTx2Rh/logo.png">
    <img src="https://jolicode.com/media/original/2013/10/homepage-docker-logo.png" alt="Logo" width="250" height="200">
  </a>

<h3 align="center">Mettre en production une API REST NodeJS avec HTTPS sous docker </h3>

  <p align="center">
   Build, Ship and Run Any App, Anywhere
  </p>
</div>


## Prérequis

Afin de mettre en place la mise en production de notre API, nous allons avoir besoin de certaines choses :
- docker 
- portainer
- une API nodejs de votre choix (Vanilla express, NestJS...).
- une certificat SSL (ici, nous allons utiliser certbot)
- un path déjà défini, là ou nous allons stocker les fichiers de notre API , pour nous : ```/home/docker/api/tutorialAPI```

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

1. Pour que tout fonctionne correctement, vous allez devoir créer à la racine de votre API (dans mon cas /home/docker/api/TutorialAPI) un script de lancement de l'application Node.js dans un fichier shell .sh

```batch
npm install
node ./dist/server.js \
    --tls-cert-path /app/cert.pem \
    --tls-key-path /app/key.pem \
    --port 3000
```
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
Afin de vous faciliter la tâche, nous allons utiliser Portainer, qui est une interface graphique pour la gestion des conteneurs Docker.
Vous allez suivre ces différentes étapes : 

1. Ouvrez Portainer et connectez-vous à votre environnement Docker.
2. Dans le panneau de gauche, cliquez sur "Stacks".
3. Cliquez sur le bouton "Add stack" en haut à droite de l'écran.
4. Donnez un nom à votre stack.
5. Collez le contenu de votre fichier YAML ci-dessous dans le champ "Web editor".
Cliquez sur "Create the stack".

Cela va créer une stack Docker avec les services et les configurations définis dans votre fichier YAML. Vous pouvez également surveiller les logs de votre stack et les paramètres de configuration dans l'interface de Portainer.

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

Ce fichier YAML est un fichier de configuration pour Docker, il contient une liste de services qui sont des conteneurs Docker qui travaillent ensemble pour fournir une application complète.

Le service dans ce fichier s'appelle "tutorial" et est basé sur l'image Docker "node:latest", qui est une image officielle de Node.js à jour. Le service est configuré pour exposer le port 3000 du conteneur et le mapper sur le port 32000 de la machine hôte dans notre cas.

Le service est également configuré pour utiliser un répertoire de travail "/app" dans le conteneur.
Un répertoire local "/home/docker/api/tutorialAPI" est monté sur le répertoire de travail du conteneur avec les deux fichiers de certificats SSL qui sont eux aussi montés dans le conteneur pour permettre la communication sécurisée.

Enfin, la commande "sh /app/start.sh" est exécutée lorsque le conteneur est lancé. Cette commande va lancer notre script défini plus haut et va installer les dépendances nécessaires pour l'application et exécuter l'api avec des options de certificat SSL et de port.


Voilà :)

J'espère que ce tutoriel vous a plu ^^

