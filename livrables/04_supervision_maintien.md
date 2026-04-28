# Supervision, journalisation, sauvegarde et maintenance corrective

> Competence evaluee : `C32` — Mettre en oeuvre un systeme de supervision pour detecter, diagnostiquer et corriger bugs, incidents et failles.

## 1. Journalisation

### 1.1 Services journalises

| Service | Emplacement des journaux | Niveau de detail |
| --- | --- | --- |
| Apache (accès) | `/var/log/apache2/opstrack_access.log` | Toutes les requêtes HTTP |
| Apache (erreurs) | `/var/log/apache2/opstrack_error.log` | Erreurs applicatives et serveur |
| Laravel | `/var/www/opstrack/storage/logs/laravel.log` | `debug` (qual) / `error` (prod) |
| MongoDB (events) | Base `opstrack_logs`, collection `event_logs` | Événements API et webhook |
| Système | `journalctl -u apache2/mysql/mongod/redis-server` | Logs systemd des services |

### 1.2 Configuration de la journalisation

Laravel est configuré avec `LOG_CHANNEL=stack` et `LOG_LEVEL=error` en production. Les événements applicatifs (appels API, webhook) sont journalisés dans MongoDB via l'`EventLogService` :

```bash
# Consulter les logs Laravel en temps réel
tail -f /var/www/opstrack/storage/logs/laravel.log

# Consulter les événements MongoDB
mongosh opstrack_logs --eval \
  "db.event_logs.find().sort({recorded_at:-1}).limit(20).pretty()"

# Consulter les logs Apache
sudo tail -f /var/log/apache2/opstrack_error.log
```

## 2. Outils et configurations d'audit

- **Logs Apache** : analyse des codes HTTP (4xx, 5xx) pour détecter les erreurs et tentatives d'intrusion
- **Logs Laravel** : détection des exceptions et erreurs applicatives
- **MongoDB EventLog** : traçabilité des opérations API et webhook pour le diagnostic inter-services
- **`php artisan tinker`** : inspection de l'état de la base de données en temps réel
- **`redis-cli monitor`** : surveillance des opérations Redis en temps réel
- **`systemctl status`** : état des services système

## 3. Supervision et alertes

### 3.1 Sondes mises en place

| Sonde | Cible | Seuil ou condition | Action en cas d'alerte |
| --- | --- | --- | --- |
| Health check API | `GET /api/health` | HTTP ≠ 200 | Redémarrer Apache, vérifier les logs |
| État des services | `apache2`, `mysql`, `mongod`, `redis-server` | Service arrêté | Redémarrer le service, investiguer |
| Espace disque | Partition `/` | > 85 % | Nettoyer les logs, purger les caches |
| Certificat SSL | Port 443 | Expiration < 30 jours | Renouveler via `certbot renew` |

### 3.2 Mecanisme d'alerte

Script de supervision planifié via cron toutes les 5 minutes :

```bash
# /usr/local/bin/opstrack-healthcheck.sh
#!/bin/bash
ERRORS=0
for SERVICE in apache2 mysql redis-server mongod; do
    systemctl is-active --quiet "$SERVICE" || {
        echo "ALERTE : $SERVICE arrêté"
        ERRORS=$((ERRORS+1))
    }
done
STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
    https://eval-dfs-p-tpl-20263-01.it-students.fr/api/health)
[ "$STATUS" = "200" ] || echo "ALERTE : API inaccessible (HTTP $STATUS)"
DISK=$(df / | awk 'NR==2{print $5}' | tr -d '%')
[ "$DISK" -lt 85 ] || echo "ALERTE : Disque à $DISK%"
exit $ERRORS

# Planification cron
echo "*/5 * * * * ubuntu /usr/local/bin/opstrack-healthcheck.sh \
    >> /var/log/opstrack-health.log 2>&1" | sudo tee /etc/cron.d/opstrack-health
```

## 4. Strategie de sauvegarde et restauration

### 4.1 Elements sauvegardes

| Element | Methode | Frequence | Retention |
| --- | --- | --- | --- |
| Base MySQL (`opstrack`) | `mysqldump` + gzip | Quotidienne à 2h | 7 jours |
| Base MongoDB (`opstrack_logs`) | `mongodump` + tar.gz | Quotidienne à 2h30 | 7 jours |
| Code applicatif | Dépôt Git | À chaque commit | Indéfinie |

Scripts de sauvegarde planifiés :
```bash
# MySQL
0 2 * * * ubuntu mysqldump -u root -p0000 opstrack | gzip \
    > /var/backups/opstrack/mysql/opstrack_$(date +%Y%m%d).sql.gz

# MongoDB
30 2 * * * ubuntu mongodump --db opstrack_logs \
    --archive=/var/backups/opstrack/mongo/opstrack_logs_$(date +%Y%m%d).gz --gzip
```

### 4.2 Procedure de restauration

```bash
# Restauration MySQL
gunzip < /var/backups/opstrack/mysql/opstrack_YYYYMMDD.sql.gz \
    | mysql -u root -p0000 opstrack

# Restauration MongoDB
mongorestore --db opstrack_logs \
    --archive=/var/backups/opstrack/mongo/opstrack_logs_YYYYMMDD.gz --gzip
```

> La procédure de restauration a été validée sur l'environnement de qualification. En production, une restauration nécessite une fenêtre de maintenance.

## 5. Diagnostic et correction du bug technique

### 5.1 Symptome observe

Le tableau de bord principal affiche des compteurs (tickets ouverts, critiques, planifiés) qui ne correspondent plus à l'état réel des tickets en base de données. De plus, une recherche de tickets combinée à un filtre de priorité retourne des résultats incohérents incluant des tickets de toutes priorités.

### 5.2 Demarche de diagnostic

1. Analyse du `DashboardController.php` : identification d'un commentaire explicite signalant un défaut intentionnel sur le cache
2. Analyse du `TicketController.php` méthode `index()` : identification de l'utilisation de `orWhereRaw` sans regroupement des conditions
3. Analyse de la méthode `update()` : constat que la mise à jour d'un ticket n'invalide pas le cache du dashboard

### 5.3 Cause racine identifiee

**Bug 1 — Cache KPIs :** Les KPIs sont mis en cache 30 minutes (`Cache::remember('dashboard.kpis', now()->addMinutes(30), ...)`) sans jamais être invalidés lors des modifications de tickets, provoquant une dérive des compteurs.

**Bug 2 — Recherche + filtre :** L'utilisation de `orWhereRaw` sans regroupement génère une requête SQL mal formée :
```sql
-- Générée (incorrecte)
WHERE title LIKE '%x%' OR reference LIKE '%x%' AND priority = 'critical'
-- Attendue
WHERE (title LIKE '%x%' OR reference LIKE '%x%') AND priority = 'critical'
```

**Bug 3 — Cache non invalidé après update :** La méthode `update()` du `TicketController` sauvegarde le ticket sans invalider le cache `dashboard.kpis`.

### 5.4 Correctif applique

**Correction Bug 1 & 3 — `DashboardController.php` et `TicketController.php` :**
```php
// DashboardController.php : cache réduit à 1 minute
'kpis' => Cache::remember('dashboard.kpis', now()->addMinutes(1), function(): array { ... })

// TicketController.php : invalidation après update
$ticket->save();
Cache::forget('dashboard.kpis');
```

**Correction Bug 2 — `TicketController.php` méthode `index()` :**
```php
// Avant (incorrect + risque injection SQL)
$query->where('title', 'like', "%{$search}%")
    ->orWhereRaw("reference like '%{$search}%'");

// Après (correct + sécurisé)
$query->where(function($q) use ($search) {
    $q->where('title', 'like', "%{$search}%")
      ->orWhere('reference', 'like', "%{$search}%");
});
```

### 5.5 Verification apres correction

```bash
# Vider le cache pour forcer la mise à jour
php artisan cache:clear

# Tester la recherche avec filtre priorité
curl -H "Authorization: Bearer <token>" \
  "https://eval-dfs-p-tpl-20263-01.it-students.fr/api/v1/tickets?search=panne&priority=critical"
# Résultat attendu : uniquement des tickets critiques contenant "panne"
```

## 6. Diagnostic et correction de la faille de securite

### 6.1 Faille identifiee

Deux failles identifiées : une injection SQL potentielle dans la recherche de tickets, et une validation insuffisante du statut dans le webhook.

### 6.2 Demarche de diagnostic

1. Analyse de `TicketController.php` : `orWhereRaw` interpole directement `$search` dans du SQL brut
2. Analyse de `WebhookController.php` : le champ `status` est validé comme `string` uniquement, sans restriction sur les valeurs acceptées
3. Analyse du `.env` de qualification : `APP_DEBUG=true` exposait les traces d'erreur

### 6.3 Evaluation du risque

**Injection SQL** : sévérité **Haute** — un attaquant contrôlant le paramètre `search` pourrait extraire des données sensibles, contourner l'authentification ou corrompre la base de données.

**Validation webhook** : sévérité **Moyenne** — une valeur arbitraire dans `status` peut corrompre les données ou provoquer un comportement imprévisible.

**APP_DEBUG en production** : sévérité **Haute** — expose les credentials, chemins serveur et structure interne en cas d'erreur.

### 6.4 Mesure corrective appliquee

**Faille 1 — Injection SQL :**
```php
// Remplacement de orWhereRaw par orWhere (requêtes préparées PDO)
->orWhere('reference', 'like', "%{$search}%");
```

**Faille 2 — Validation webhook :**
```php
// Ajout de la règle in: sur le champ status
'status' => ['required', 'string', 'in:new,scheduled,in_progress,resolved,closed'],
```

**Faille 3 — APP_DEBUG :**
```env
APP_DEBUG=false
```

### 6.5 Verification apres correction

```bash
# Test injection SQL (doit retourner des résultats normaux, pas d'erreur SQL)
curl -H "Authorization: Bearer <token>" \
  "https://.../api/v1/tickets?search='; DROP TABLE tickets; --"
# Résultat attendu : liste vide ou tickets normaux, pas d'erreur 500

# Test webhook avec statut invalide
curl -u user:password -X POST https://.../hooks.php \
  -H "Content-Type: application/json" \
  -d '{"ticket_reference":"INC-001","status":"INVALIDE","summary":"test"}'
# Résultat attendu : HTTP 422 Unprocessable Entity
```

## 7. Autres observations

**Bug 4 — Webhook ignore le statut reçu :**
Le webhook forçait systématiquement le statut `scheduled` au lieu d'utiliser la valeur reçue dans le payload. Correction : `$ticket->update(['status' => $payload['status']])`.

**Bug 5 — Microservice Next.js retourne toujours une liste vide :**
Le fichier `lib/api.ts` utilisait `payload.items` alors que l'API Laravel retourne `payload.data`. Correction : `return payload.data ?? []`.
