# Changelog

Toutes les modifications notables apportees pendant l'epreuve sont documentees dans ce fichier.

Le format s'inspire de [Keep a Changelog](https://keepachangelog.com/).

## [Session du 28 avril 2026]

### Ajoute

- Script de supervision `opstrack-healthcheck.sh` planifié via cron toutes les 5 minutes
- Scripts de sauvegarde MySQL et MongoDB planifiés quotidiennement à 2h
- Virtual Host Apache avec HTTPS et en-têtes de sécurité sur la production
- Certificat Let's Encrypt via Certbot sur le domaine `eval-dfs-p-tpl-20263-01.it-students.fr`
- Pipeline GitHub Actions `.github/workflows/deploy.yml` pour le déploiement automatisé
- Fichier `.gitignore` pour le microservice Next.js (exclusion `node_modules/` et `.next/`)

### Modifie

- `DashboardController.php` : durée du cache KPIs réduite de 30 à 1 minute
- `TicketController.php` : regroupement des conditions de recherche dans une closure
- `TicketController.php` : ajout de `Cache::forget('dashboard.kpis')` après mise à jour d'un ticket
- `WebhookController.php` : application du statut reçu dans le payload au lieu de forcer `scheduled`
- `microservices/dispatch-dashboard/lib/api.ts` : correction de la clé de réponse (`payload.data` au lieu de `payload.items`)
- `.env` production : `APP_DEBUG=false`, `APP_ENV=production`

### Corrige

- Cache des KPIs du tableau de bord non invalidé lors des modifications de tickets
- Recherche de tickets combinée aux filtres de priorité retournant des résultats incohérents
- Webhook ignorant le statut reçu et forçant systématiquement `scheduled`
- Microservice Next.js affichant toujours une liste vide

### Securite

- Élimination d'une vulnérabilité d'injection SQL dans la recherche de tickets (`orWhereRaw` → `orWhere`)
- Validation stricte du champ `status` dans le webhook (`in:new,scheduled,in_progress,resolved,closed`)
- `APP_DEBUG=false` imposé en production
