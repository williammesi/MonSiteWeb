# Configuration d'un pipeline CI/CD avec GitHub Actions

## En bref :

### Définitions :

- Pipeline CI/CD : Un processus automatisé qui permet de tester, construire et deployer un projet automatique après chaque modification apportées (Après chaque "push" sur un repository par exemple)  
- GitHub Actions : Un outil intégré à GitHub qui permet d'exécuter certaines tâches automatiquement (tests, build, déploiement) en réponse à certains évènements (Push)  
- Serveur web NGINX : Un serveur web configurable sur tout serveur (Linux/Unix & Windows server) permettant d'héberger et gérer plusieurs sites ou applications web

### Objectif :

Lorsqu'une modification sera apportée à notre site web et publiée sur notre repository (Commit + Push), notre pipeline va automatiquement assurer le déploiement de ce site web sur notre serveur web préconfiguré.

### Difficulté :

- Avancé : Préférable d'avoir un peu d'expérience en développement et déploiement web, et expérience en configuration de serveurs sous Linux  
- Durée approximative : Entre 2h et 3h dépendant de la complexité du projet

### Prérequis :

- **Un projet de base (voir MonSiteWeb.zip)**, dans le cadre de cette procédure, nous utiliserons un projet sous Vue.js, mais tout framework basé sur Node.js devrait également fonctionner sans trop de difficultés.  
- **Un repository GitHub** qui contient les fichiers de notre site web  
- **Un serveur prêt à être configuré**, dans le cadre de cette procédure, nous utiliserons un serveur Ubuntu, hébergé sur DigitalOcean. Tout autre hébergeur et distributions (Fedora, Debian ect.) devraient également fonctionner à quelques différences près au niveau des commandes utilisées.

## Étapes :

### 1. Configuration d'un serveur web NGINX

#### 1.1 Installation de NGINX sur Ubuntu :

``` bash
# Cette commande pourrait durer quelques minutes
sudo apt update && sudo apt upgrade -y
# Installation de NGINX : 
sudo apt install nginx -y
# S'assurer que le service de NGINX est actif et "enabled"
sudo systemctl start nginx
sudo systemctl enable nginx
# On autorise NGINX dans le firewall
sudo ufw allow 'Nginx Full'
```

#### 1.2 Une fois complété, testez l'installation de NGINX en ouvrant un navigateur web, et en consultant l'adresse IP de votre serveur web

> Si vous ne connaissez pas l'adresse ip de votre serveur, executez la commande :

``` bash
ip -4 addr show
```

> Par exemple : http://165.227.33.36 (Utilisez votre adresse IP de votre serveur)
> Vous devriez voir une page similaire à celle-ci :

<img src="procedure-media/a088b50993d30151affb7af0030a8ade38ef6976.png"
class="wikilink" alt="Pastedimage20251010143554.png" />
  

#### 1.3 Configuration du serveur NGINX

``` bash
# Création du dossier de notre site web
sudo mkdir -p /var/www/MonSiteWeb
# Donner les droits d'accès au dossier à l'utilisateur connecté
sudo chown $USER:$USER /var/www/MonSiteWeb
# Création du sous-dossier /dist, qui va contenir le build de notre application web
cd /var/www/MonSiteWeb
sudo mkdir dist
# Création du fichier de configuration NGINX pour notre application web
sudo nano /etc/nginx/sites-available/MonSiteWeb
```

À l'intérieur du fichier "MonSiteWeb" copiez cette configuration, et remplacez "{HOST}" par l'adresse IP de votre serveur

``` nginx
server {
    listen 80;
    # Remplacez le {HOST} par votre adresse ip, ou nom de domaine
    server_name {HOST};  
    
    root /var/www/MonSiteWeb/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

Avant de passer à la prochaine étape, activez la configuration du site web, testez votre configuration et redémarrez le serveur NGINX

``` bash
# On crée un symlink afin d'activer la configuration NGINX du site web
sudo ln -s /etc/nginx/sites-available/MonSiteWeb /etc/nginx/sites-enabled/
# Ensuite, on teste la configuration NGINX
sudo nginx -t
# Et finalement, on redémarre le serveur NGINX
sudo systemctl restart nginx
```

### 2. Déploiement manuel de notre application

Avant de configurer le pipeline, assurons nous que notre application web soit fonctionnelle

#### 2.1 Cloner le repository sur notre serveur

Afin de cloner le repository, nous allons devoir générer une paire de clé SSH pour permettre l'authentification avec GitHub. Sur votre serveur exécutez cette commande :

``` bash
# Remplacez le email par le email associé à votre compte GitHub
ssh-keygen -t ed25519 -C "your_email@example.com"
```

Si un "passphrase" vous est demandé, laissez le vide et appuyez sur "Enter"

<figure>

<img src="procedure-media/ad996a387dae0b3fceb6709204c58dcdc5365765.png"
class="wikilink" alt="Pastedimage20251010175127.png" />
<figcaption aria-hidden="true">

Pastedimage20251010175127.png
</figcaption>

</figure>

Ensuite, copiez le chemin vers votre clé publique avec "CTRL+SHIFT+C" et exécutez cette commande :

``` shell
# Remplacez {Votre-Chemin} par le chemin que vous avez copié précedemment
cat {Votre-Chemin}
```

Votre clé publique sera affichée. Copiez la ligne complète (CTRL+SHIFT+C).

Connectez vous à votre compte GitHub, allez dans vos paramètres, puis "SSH and GPG keys"
<img src="procedure-media/0e067486d90117d419eac14281d7d4d094e92e1f.png"
class="wikilink" alt="Pastedimage20251010161034.png" />
Ajoutez une nouvelle clé SSH : <img src="procedure-media/76dbdea477a4f4cdb15e11626e94ba398095fb2e.png"
class="wikilink" alt="Pastedimage20251010161128.png" />
Donnez lui le titre de votre choix, et ensuite copiez votre clé privée dans la zone de texte. Enregistrez la clé et dirigez vous vers votre repository.

Une fois sur votre repository, copiez l'adresse SSH, et redirigez vous sur votre serveur. Une fois sur votre serveur, exécutez les commandes suivantes :

``` shell
cd ~
# Remplacez {ADRESS} par l'adresse SSH que vous avez copié précedemment
git clone {ADRESS}
# Remplacez {MonSiteWeb} par le nom du sous-dossier de votre repository
cd {MonSiteWeb}
```

Ca y est ! Votre repository est cloné sur votre serveur, vous pouvez passer à la prochaine étape !

#### 2.2 Installation de Node.js

Afin de build notre application, nous aurons besoin d'installer Node.js

``` bash
# Configuration du repository
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
# Installation de Node.js 
sudo apt-get install -y nodejs
# Test de l'installation 
node --version
npm --version
```

#### 2.3 Build et déploiement initial de notre application

Afin de build et déployer notre application pour la première fois, simplement exécuter ces commandes :

``` bash
# Installation des modules pré-requis
npm i
# Compilation de notre application web
npm run build
# On copie le build vers le chemin de notre site web NGINX
cp -r dist/* /var/www/MonSiteWeb/dist/
```

Ca y est ! Maintenant, vous pouvez tester que le déploiement a fonctionné en allant sur :
http://{ADRESSEIP_DE_VOTRE_SERVEUR}

> Rappel : Si vous ne connaissez pas l'adresse ip de votre serveur, exécutez cette commande :

``` bash
ip -4 addr show
```

Si tout fonctionne bien, vous êtes prêt à passer à la prochaine étape !

### 3. Configuration des GitHub Actions
#### 3.1 Géneration de clé SSH pour le déploiement
Nous allons maintenant génerer une deuxième clé SSH qui sera responsable d'authentifier les github actions.

À partir de votre serveur executez ces commandes :

``` bash
# Génération de la clé ssh
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/github_actions


# Ajouter la clé aux clés autorisées du serveur
cat ~/.ssh/github_actions.pub >> ~/.ssh/authorized_keys

# Copier la clé privée (Nous en aurons besoins pour configurer les Github Actions)
cat ~/.ssh/github_actions

```

#### 3.2 Configurations des secrets Github Actions
À partir de votre repository, dirigez vous dans "Settings" et puis dans "Secrets and variables" et finalement dans "Actions"
<img src="procedure-media/Pasted image 20251203212500.png"
class="wikilink" alt="Pasted image 20251203212500.png" />


Ajoutez ces 3 secrets :
- SERVER_IP (l'adresse ip publique de votre serveur)
- SERVER_USER (le nom d'utilisateur avec lequel vous avez généré la clé ssh)
- SSH_PRIVATE_KEY (le contenu de la clé privée que nous avons copiée précedemment)
#### 3.3 Configuration des Github Actions

À partir de votre repository, allez dans la sections "Actions" et sélectionnez "New workflow"

![[Pasted image 20251203212908.png]]

Github vous proposera certain modèle préconcus de Github Actions. Sentez vous libre d'explorer toutes les configurations possibles plus tard. Pour l'instant, nous allons créer notre propre workflow en cliquant sur "Set up a workflow yourself"

![[Pasted image 20251203213100.png]]

Renommez le fichier "deploy.yml" et copiez ce script basique de déploiement à l'intérieur : 

``` yaml
name: Deploy Vue App

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        run: npm run build
      
      - name: Deploy to server
        uses: appleboy/scp-action@master
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "dist/*"
          target: "/var/www/MonSiteWeb/dist/"
          strip_components: 1
```

#### 3.4 Testez !!
La configuration de votre environnement de déploiement CI/CD est complet, faites une modification à votre site web, et lorsque votre modification aura été "commit" et "push", le Github Action sera lancé automatiquement et déploiera les changements directement sur votre serveur de production !

Dans un contexte réel de développement, le repository aurait une branche "prod" et seulement les commits sur cette branche déclencherait l'action. Vous pourriez répeter le processus aussi avec une branche "dev-prod" pour un serveur de développement également.
