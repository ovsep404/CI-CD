# Guide de déploiement CI/CD avec GitHub Actions sur Planethoster

Ce guide détaille les étapes nécessaires pour mettre en place un flux de travail CI/CD en utilisant GitHub Actions pour déployer une application sur un serveur Planethoster lorsque des changements sont poussés sur la branche `main`.

## Prérequis

- Un compte GitHub.
- Un serveur sur Planethoster avec accès SSH.
- Connaissance de base de Git et GitHub.

## Configuration des clés SSH

1. **Générez une paire de clés SSH** si vous n'en avez pas déjà :
   - Sur votre terminal local, exécutez `ssh-keygen` et suivez les instructions pour créer votre paire de clés. 
   - Assurez-vous de ne pas définir de passphrase pour faciliter l'automatisation.

2. **Connectez-vous à votre espace client Planethoster**.

3. **Ajoutez votre clé SSH publique** :
   - Naviguez vers l'onglet `Fichiers` puis `Clés SSH`.
   - Ajoutez la clé publique que vous avez générée.

## Configuration du domaine

1. **Ajoutez un domaine à votre espace Planethoster** :
   - Allez dans l'onglet `Domaines`.
   - Ensuite `Sous-Domaines` .
   - Et ajouter votre sous-domaine .
   - Ensuite valdier SSL/TLS .

## Configuration du projet GitHub

1. **Créez un nouveau repository GitHub** ou accédez à un repository existant.

2. **Ajoutez des secrets pour la configuration** :
   - Allez dans `Settings > Secrets and variables > Actions secrets`.
   - Ajoutez les secrets suivants :
     - `DEPLOY_PATH` : Chemin absolu vers le dossier du projet sur le serveur.   exp: /home/username/myproject
     - `DEPLOY_SERVER` : Adresse IP ou domaine de votre serveur Planethoster.    exp: 186.222.186.287
     - `DEPLOY_USER` : Nom d'utilisateur pour la connexion SSH.   exp: username
     - `SSH_PRIVATE_KEY` : Contenu de votre clé privée SSH.   exp: commance par -----BEGIN OPENSSH PRIVATE KEY----- et fini par -----END OPENSSH PRIVATE KEY-----

 

## Création du fichier de workflow GitHub Actions

1. **Créez un dossier pour les workflows** si ce n'est pas déjà fait :
   - Dans votre projet, créez le dossier `.github/workflows`.

2. **Ajoutez le fichier `deploy.yml`** :
   - Copiez le contenu suivant dans ce fichier pour définir le processus de déploiement :

```yaml
name: Deploy

on:
  push:
    branches:
      - main
    paths:
      - '**'

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Setup SSH and rsync
        env:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
          DEPLOY_USER: ${{ secrets.DEPLOY_USER }}
          DEPLOY_SERVER: ${{ secrets.DEPLOY_SERVER }}
          DEPLOY_PATH: ${{ secrets.DEPLOY_PATH }}
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan -p 5022 ${{ secrets.DEPLOY_SERVER }} >> ~/.ssh/known_hosts

      - name: Test SSH connection
        run: ssh -vvv -p 5022 -o StrictHostKeyChecking=no ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_SERVER }} "echo 'Connection successful'" || true

      - name: Deploy to server
        run: rsync -avz --exclude='.env' -e 'ssh -p 5022' --delete . ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_SERVER }}:${{ secrets.DEPLOY_PATH }}
