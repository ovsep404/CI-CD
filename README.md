# Guide de déploiement CI/CD avec GitHub Actions sur Planethoster

Ce guide détaille les étapes nécessaires pour mettre en place un simple flux de travail CI/CD en utilisant GitHub Actions pour déployer une application sur un serveur Planethoster lorsque des changements sont poussés sur la branche `main`.

Voici la page simple que j'ai déployée sur mon serveur Planethoster : [http://cicd.ovsep-simonian.mds-montpellier.yt/](http://cicd.ovsep-simonian.mds-montpellier.yt/).

## Prérequis

- Un compte GitHub.
- Un serveur sur Planethoster avec accès SSH.

## Configuration des clés SSH

1. **Générez une paire de clés SSH** si vous n'en avez pas déjà :
   - Sur votre terminal local (Git Bash si vous êtes sur Windows), exécutez `ssh-keygen` et suivez les instructions pour créer votre paire de clés. 
   - Assurez-vous de ne pas définir de passphrase pour faciliter l'automatisation.

2. **Connectez-vous à votre espace client Planethoster**.

3. **Ajoutez votre clé SSH publique** :
   - Pour récupérer votre clé publique, ouvrez votre terminal et tapez :
     - Sur Linux : `cat ~/.ssh/<nom du clé>`
     - Sur Windows (dans PowerShell) : `Get-Content C:\Users\yourusername\.ssh\<nom du clé>`
     - Sur Windows (dans Git Bash) : `cat /c/Users/yourusername/.ssh/<nom du clé>`
   - Copiez le contenu affiché, c'est votre clé publique SSH.
   - Naviguez vers l'onglet sur Planethoster `Fichiers` puis `Clés SSH`.
   - Ajoutez la clé publique que vous avez générée.

## Configuration du domaine

1. **Ajoutez un domaine à votre espace Planethoster** :
   - Naviguez vers l'onglet `Domaines`.
   - Naviguez vers l'onglet `Gestion des domaines`.
   - Cliquez sur `Ajouter`.
   - Sélectionnez `Sous-Domaines`. ou sur `Domaine` si vous souhaitez ajouter un domaine.
   - Ajoutez votre sous-domaine dans le champ approprié.
   - Cliquez sur `Créer`.
   - Ensuite Validez l'option SSL/TLS pour sécuriser votre sous-domaine.

## Configuration du projet GitHub

1. **Créez un nouveau repository GitHub** ou accédez à un repository existant.

2. **Ajoutez des secrets pour la configuration** :
   - Allez dans `Settings (de ton repository) > Security > Secrets and variables > Actions secrets`.
   - Ajoutez les secrets suivants :
     - `DEPLOY_PATH` : Chemin absolu vers le dossier du projet sur le serveur.   
       ```markdown
       Exemple: /home/username/myproject
       ```
     - `DEPLOY_SERVER` : Adresse IP ou domaine de votre serveur Planethoster.    
       ```markdown
       Exemple: 186.222.186.287
       ```
     - `DEPLOY_USER` : Nom d'utilisateur pour la connexion SSH.   
       ```markdown
       Exemple: username
       ```
     - `SSH_PRIVATE_KEY` : Contenu de votre clé privée SSH.   
       ```markdown
       Exemple: commence par -----BEGIN OPENSSH PRIVATE KEY----- et finit par -----END OPENSSH PRIVATE KEY-----
       ```
      ![Secrets](./images/secrets.png) 
 
## Création du fichier de workflow GitHub Actions

1. **Créez un dossier pour les workflows** si ce n'est pas déjà fait :
   - Dans votre projet, créez le dossier `.github/workflows`.

2. **Ajoutez le fichier `deploy.yml`** :
   - Copiez le contenu suivant dans ce fichier pour définir le processus de déploiement :

```yaml
name: Deploy  # Nom du workflow

on:  # Déclencheur du workflow
  push:  # Déclenché lors d'un push
    branches:  # Sur la branche spécifiée
      - main  # Branche 'main'
    paths:  # Pour tous les chemins
      - '**'  # Correspond à tous les fichiers

jobs:  # Tâches à exécuter
  deploy:  # Tâche de déploiement
    runs-on: ubuntu-latest  # Exécute la tâche sur la dernière version d'Ubuntu

    steps:  # Étapes de la tâche
      - name: Checkout code  # Étape de récupération du code
        uses: actions/checkout@v2  # Utilise l'action de checkout de GitHub

      - name: Setup SSH and rsync  # Étape de configuration de SSH et rsync
        env:  # Variables d'environnement
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}  # Clé privée SSH
          DEPLOY_USER: ${{ secrets.DEPLOY_USER }}  # Utilisateur pour le déploiement
          DEPLOY_SERVER: ${{ secrets.DEPLOY_SERVER }}  # Serveur de déploiement
          DEPLOY_PATH: ${{ secrets.DEPLOY_PATH }}  # Chemin de déploiement sur le serveur
        run: |  # Commandes à exécuter
          mkdir -p ~/.ssh  # Crée le dossier .ssh si nécessaire
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa  # Crée le fichier id_rsa avec la clé privée SSH
          chmod 600 ~/.ssh/id_rsa  # Modifie les permissions du fichier pour qu'il soit lisible uniquement par l'utilisateur
          ssh-keyscan -p 5022 ${{ secrets.DEPLOY_SERVER }} >> ~/.ssh/known_hosts  # Ajoute le serveur aux hôtes connus de SSH

      - name: Test SSH connection  # Étape de test de la connexion SSH
        run: ssh -vvv -p 5022 -o StrictHostKeyChecking=no ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_SERVER }} "echo 'Connection successful'" || true  # Tente de se connecter au serveur et affiche un message en cas de succès

      - name: Deploy to server  # Étape de déploiement sur le serveur
        run: rsync -avz --exclude='.env' -e 'ssh -p 5022' --delete . ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_SERVER }}:${{ secrets.DEPLOY_PATH }}  # Utilise rsync pour synchroniser les fichiers avec le serveur, en excluant les fichiers .env

```

## Déploiement

Le processus de déploiement est automatisé grâce à GitHub Actions. Toutes les étapes nécessaires pour déployer l'application sont définies dans le fichier `.github/workflows/main.yml`.

Pour suivre le processus de déploiement :

1. Allez dans l'onglet `Actions` de votre dépôt GitHub.
2. Sélectionnez le workflow de déploiement dans la liste.
3. Vous pouvez voir le détail de chaque exécution du workflow, y compris les logs de chaque étape.

Cela vous permet de suivre le processus de déploiement et de diagnostiquer d'éventuels problèmes.


## En résumé:

En fait, le serveur et ton GitHub communiquent via la clé privée et la clé publique.

1. Tu génères une paire de clés SSH sur ton ordinateur : une clé publique et une clé privée.
2. la clé publique est ajoutée à ton compte GitHub et au serveur PlanetHoster.
3. La clé privée est gardée secrète sur ton ordinateur et dans les paramètres de sécurité de ton dépôt GitHub
4. Quand GitHub veut envoyer du code au serveur PlanetHoster, il utilise la clé privée pour prouver son identité.
5. PlanetHoster vérifie cette identité avec la clé publique que tu as ajoutée.

En résumé :
Clé publique sur GitHub et PlanetHoster.
Clé privée utilisée par GitHub pour s'authentifier.
Cette combinaison de clés assure que les communications entre GitHub et PlanetHoster sont sécurisées et autorisées.