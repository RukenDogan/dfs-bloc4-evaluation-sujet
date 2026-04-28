# Base de connaissances — Note de passation

Ce document est destine a un pair charge de reprendre la maintenance de l'application. Il doit permettre de comprendre le fonctionnement, les points d'attention et les procedures essentielles sans zone d'ombre majeure.

## 1. Presentation de l'application

OpsTrack Field Service est une application de gestion des interventions terrain destinée aux équipes de techniciens et leurs superviseurs. Elle permet de créer et suivre des tickets d'intervention, affecter des techniciens, recevoir des mises à jour externes via webhook et superviser les interventions via un tableau de bord secondaire.

## 2. Architecture technique

### 2.1 Composants principaux

| Composant | Technologie | Role |
| --- | --- | --- |
| Application principale | Laravel 12 / PHP 8.4 | Interface web, API REST, traitement webhook |
| Base de données relationnelle | MySQL 8 | Tickets, interventions, utilisateurs, sites |
| Base de données NoSQL | MongoDB 8 | Journaux techniques et événements applicatifs |
| Cache | Redis 7 | Cache applicatif et stockages temporaires |
| Tableau de bord secondaire | Next.js | Microservice `dispatch-dashboard` |
| Serveur web | Apache 2.4 | Exposition HTTP/HTTPS |
| API externe | Open-Meteo | Enrichissement météo |

### 2.2 Schema d'architecture

```
Internet
    │
    ▼
Apache (HTTPS :443 / HTTP :80 → redirect)
    │
    ├── Laravel (/ et /api/*)
    │     ├── MySQL (données métier)
    │     ├── MongoDB (journaux)
    │     └── Redis (cache)
    │
    ├── hooks.php (webhook, HTTP Basic)
    │
    └── Next.js dispatch-dashboard (port 3000)
          └── API Laravel /api/v1/tickets
```

## 3. Points d'attention connus

- **Cache Redis** : les KPIs du dashboard sont mis en cache sous la clé `dashboard.kpis`. En cas d'affichage incorrect, vider le cache : `php artisan cache:clear`
- **Webhook** : appelé chaque minute par un système externe. Le statut reçu doit correspondre à `new`, `scheduled`, `in_progress`, `resolved` ou `closed`. Toute autre valeur retourne HTTP 422.
- **MongoDB** : si MongoDB est indisponible, l'application reste fonctionnelle — l'`EventLogService` log un avertissement et continue sans bloquer.
- **Microservice Next.js** : nécessite les variables `LARAVEL_API_BASE_URL` et `LARAVEL_API_TOKEN` dans `.env.local`. Sans token valide, toutes les requêtes retournent HTTP 401.
- **Monoserveur production** : pas de redondance. Une panne entraîne une indisponibilité totale.

## 4. Procedures operationnelles

### 4.1 Deploiement

```bash
# Déploiement manuel depuis la qualification
ssh -i ubuntu.pem ubuntu@35.180.47.223
cd /var/www/opstrack
git pull origin main
composer install --no-dev --optimize-autoloader
php artisan migrate --force
php artisan config:cache && php artisan route:cache && php artisan view:cache
sudo systemctl reload apache2
```

### 4.2 Sauvegarde et restauration

```bash
# Sauvegarde MySQL (automatique à 2h via cron)
mysqldump -u root -p0000 opstrack | gzip > /var/backups/opstrack/mysql/opstrack_$(date +%Y%m%d).sql.gz

# Restauration MySQL
gunzip < /var/backups/opstrack/mysql/opstrack_YYYYMMDD.sql.gz | mysql -u root -p0000 opstrack
```

### 4.3 Supervision et alertes

```bash
# Vérification manuelle de l'état des services
sudo systemctl status apache2 mysql redis-server mongod

# Health check API
curl -s https://eval-dfs-p-tpl-20263-01.it-students.fr/api/health

# Logs en temps réel
tail -f /var/www/opstrack/storage/logs/laravel.log
sudo tail -f /var/log/apache2/opstrack_error.log
```

Le script `/usr/local/bin/opstrack-healthcheck.sh` s'exécute toutes les 5 minutes et log dans `/var/log/opstrack-health.log`.

### 4.4 Acces et secrets

| Ressource | Accès |
| --- | --- |
| Qualification (SSH) | `ssh -i ubuntu.pem ubuntu@51.45.7.148` |
| Production (SSH) | `ssh -i ubuntu.pem ubuntu@35.180.47.223` |
| phpMyAdmin (qual uniquement) | `http://eval-dfs-q-tpl-20263-01.it-students.fr/phpmyadmin` |
| Secrets applicatifs | Fichier `.env` sur chaque machine (non versionné) |

## 5. Bugs et failles corriges pendant l'epreuve

| # | Type | Fichier | Description |
| --- | --- | --- | --- |
| 1 | Bug | `DashboardController.php` | Cache KPIs non invalidé — compteurs incorrects |
| 2 | Bug + Sécurité | `TicketController.php` | Recherche + filtre incohérent / injection SQL |
| 3 | Bug | `TicketController.php` | Cache non invalidé après update ticket |
| 4 | Bug | `WebhookController.php` | Webhook ignorait le statut reçu |
| 5 | Sécurité | `WebhookController.php` | Statut webhook non validé |
| 6 | Bug | `dispatch-dashboard/lib/api.ts` | Microservice retournait toujours une liste vide |
| 7 | Sécurité | `.env` production | APP_DEBUG=true en production |

## 6. Ameliorations recommandees

- Migrer les credentials vers AWS Secrets Manager
- Activer l'authentification MongoDB
- Mettre en place un WAF (AWS WAF ou ModSecurity)
- Ajouter un rate limiting sur les endpoints API publics
- Migrer vers une architecture multi-instances avec ALB (cf. livrable 01)
- Renforcer les mots de passe MySQL et webhook
- Mettre en place des tests automatisés (PHPUnit) sur les cas critiques

## 7. Contacts et ressources

| Ressource | Valeur |
| --- | --- |
| Dépôt application | `https://github.com/itakademy/dfs-bloc4-evaluation-app` |
| Dépôt livrables | Fork candidat sur GitHub |
| Documentation API | `livrables/05_documentation_maintenance/documentation_api.md` |
| API Open-Meteo | `https://api.open-meteo.com/v1/forecast` |
