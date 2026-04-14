# Resonance Backend

Guide de mise en production de l'API Symfony de **Résonance** avec Docker.

## Prérequis

- Docker et Docker Compose installés sur le VPS.
- Git pour cloner le projet.
- Les ports 8080 (API) et 80 (Web) ouverts dans le pare-feu du VPS.

## Installation

### 1. Cloner le projet

```bash
git clone <URL_DU_REPO> resonance-backend
cd resonance-backend
```

### 2. Configurer l'environnement de production

Créer et configurer le fichier `.env.local` :

```bash
cp .env .env.local
nano .env.local
```

Vérifier que `DATABASE_URL` correspond aux identifiants définis dans `compose.yaml` :

```dotenv
DATABASE_URL="postgresql://!ChangeUser!:!ChangePassword!@database:5432/resonance?serverVersion=16&charset=utf8"
APP_ENV=prod
```

### 3. Lancer les conteneurs

```bash
docker compose up -d --build
```

### 4. Configuration interne de PHP

Une fois les conteneurs démarrés, installer les outils nécessaires à l'intérieur du conteneur `resonance_api` :

```bash
# A. Installer Composer
docker exec -it resonance_api sh -c "curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer"

# B. Installer les extensions PHP (Postgres)
docker exec -it resonance_api sh -c "apt-get update && apt-get install -y libpq-dev && docker-php-ext-install pdo pdo_pgsql"

# C. Redémarrer pour appliquer les extensions
docker compose restart app
```

## Initialisation de l'application

### Dépendances PHP

```bash
docker exec -it resonance_api composer install --no-dev
```

### Base de données et migrations

```bash
docker exec -it resonance_api php bin/console 
doctrine:migrations:migrate
```

Charger les données de test (fixtures).

```bash
docker exec -it resonance_api php bin/console doctrine:fixtures:load
```

### Sécurité JWT

Générer les clés de chiffrement pour l'authentification :

```bash
docker exec -it resonance_api php bin/console lexik:jwt:generate-keypair --overwrite
```

## Commandes utiles

| Action | Commande |
| --- | --- |
| Arrêter les services | `docker compose stop` |
| Tout supprimer, volumes inclus | `docker compose down -v` |
| Vider le cache Symfony | `docker exec -it resonance_api php bin/console cache:clear` |

## Troubleshooting

- Erreur 500 JWT : souvent due à un problème de droits sur les clés. Relancer `lexik:jwt:generate-keypair`.
- Erreur 504 Timeout : vérifier que le service `database` est bien `Healthy` avec `docker ps`.
- Erreur 403/405 (CORS) : vérifier la configuration dans `config/packages/nelmio_cors.yaml` et s'assurer que l'URL du frontend est autorisée.

## Accès

- L'API est exposée par défaut sur le port 8080.
- Swagger : http://<IP_VPS>:8080/api