# Deploiement automatise

> Competence evaluee : `C31` — Mettre en oeuvre un systeme de deploiement automatise respectant les bonnes pratiques DevOps.

## 1. Strategie de deploiement

### 1.1 Vue d'ensemble

Le déploiement de OpsTrack de la qualification vers la production est orchestré via **GitHub Actions**. Le pipeline est déclenché manuellement, exécute des contrôles préalables, transfère le code via SSH/rsync, applique les migrations et valide le déploiement par des smoke tests.

### 1.2 Diagramme du pipeline

```
[Déclenchement manuel]
         │
         ▼
[Contrôles préalables]
  ├── Confirmation "oui" requise
  ├── Health check qualification (HTTP 200)
  ├── composer install
  └── php artisan test
         │
         ▼ (si tous les contrôles passent)
[Déploiement production]
  ├── rsync code → prod (SSH)
  ├── composer install --no-dev
  ├── php artisan migrate --force
  ├── php artisan config:cache / route:cache / view:cache
  └── systemctl reload apache2
         │
         ▼
[Smoke tests]
  ├── HTTPS accessible (HTTP 200)
  ├── API /api/health → {"status":"ok"}
  ├── Redirection HTTP → HTTPS (301/302)
  └── Certificat SSL valide
         │
         ▼
[Notification résultat ✅ / ❌]
```

## 2. Outillage retenu

| Outil | Role dans le pipeline | Justification |
| --- | --- | --- |
| GitHub Actions | Orchestration CI/CD | Natif GitHub, gratuit pour les dépôts publics, intégration directe avec le dépôt |
| SSH / rsync | Transfert du code vers la production | Protocole sécurisé, transfert différentiel (seuls les fichiers modifiés) |
| PHP Artisan | Migrations, cache, optimisations Laravel | Outil natif Laravel, reproductible |
| curl / openssl | Smoke tests post-déploiement | Outils standard, sans dépendance externe |

## 3. Declenchement du deploiement

### 3.1 Mode de declenchement

Le déploiement est déclenché **manuellement** via `workflow_dispatch` sur la branche `main`. L'opérateur doit saisir la confirmation `oui` pour lancer le pipeline. Ce choix garantit un déploiement intentionnel et contrôlé.

### 3.2 Reproductibilite

Le pipeline est entièrement décrit dans `.github/workflows/deploy.yml`. Il peut être relancé à l'identique depuis l'interface GitHub Actions en cliquant sur **Run workflow** et en confirmant avec `oui`. Chaque exécution produit un log complet consultable depuis GitHub.

## 4. Controles prealables au deploiement

| Controle | Description | Critere de passage |
| --- | --- | --- |
| Confirmation manuelle | L'opérateur saisit "oui" | Valeur strictement égale à `oui` |
| Health check qualification | `curl` sur `/api/health` de la qual | HTTP 200 |
| Installation dépendances | `composer install` sans erreur | Exit code 0 |
| Tests unitaires | `php artisan test` | Tous les tests passent |

## 5. Mise a jour de la production

```yaml
- name: Transférer le code en production
  run: |
    rsync -avz --exclude='.git' --exclude='node_modules' --exclude='.env' \
      -e "ssh -i ~/.ssh/ubuntu.pem" \
      ./ ubuntu@35.180.47.223:/var/www/opstrack/

- name: Déploiement Laravel
  run: |
    ssh -i ~/.ssh/ubuntu.pem ubuntu@35.180.47.223 << 'EOF'
      cd /var/www/opstrack
      composer install --no-dev --optimize-autoloader
      php artisan migrate --force
      php artisan config:cache && php artisan route:cache && php artisan view:cache
      php artisan cache:clear
      sudo chown -R www-data:www-data /var/www/opstrack
      sudo chmod -R 775 storage bootstrap/cache
      sudo systemctl reload apache2
    EOF
```

## 6. Verification post-deploiement

### 6.1 Smoke tests

| Test | Commande ou methode | Resultat attendu |
| --- | --- | --- |
| HTTPS accessible | `curl -s -o /dev/null -w "%{http_code}" https://eval-dfs-p-tpl-20263-01.it-students.fr` | `200` |
| API health check | `curl -s https://.../api/health` | `{"status":"ok",...}` |
| Redirection HTTP→HTTPS | `curl -s -o /dev/null -w "%{http_code}" http://eval-dfs-p-tpl-20263-01.it-students.fr` | `301` ou `302` |
| Certificat SSL valide | `openssl s_client -connect ...:443` | Certificat Let's Encrypt valide |

### 6.2 Preuve de deploiement reussi

Le log GitHub Actions constitue la preuve formelle du déploiement. Chaque étape est horodatée et les résultats des smoke tests sont visibles dans les logs de l'étape **Smoke tests**.

## 7. Conduite a tenir en cas d'echec

| Etape en echec | Action |
| --- | --- |
| Contrôles préalables | Corriger le problème sur la qualification, relancer le pipeline |
| Déploiement (rsync/SSH) | Vérifier la connectivité SSH et les logs Apache : `sudo tail -f /var/log/apache2/opstrack_error.log` |
| Migration BDD | Vérifier les logs Laravel : `tail -f /var/www/opstrack/storage/logs/laravel.log` |
| Smoke tests | Rollback vers le commit précédent (voir procédure ci-dessous) |

**Procédure de rollback :**
```bash
ssh -i ubuntu.pem ubuntu@35.180.47.223
cd /var/www/opstrack
git log --oneline -5
git checkout <commit-précédent>
php artisan config:cache && php artisan route:cache
sudo systemctl reload apache2
```

## 8. Scripts et fichiers de configuration

| Fichier | Role |
| --- | --- |
| `.github/workflows/deploy.yml` | Pipeline GitHub Actions complet (contrôles, déploiement, smoke tests) |
| `.env` (production) | Variables d'environnement de production (exclu du dépôt Git) |
| `/etc/apache2/sites-available/opstrack.conf` | Configuration Apache Virtual Host |

**Contenu complet de `.github/workflows/deploy.yml` :**

```yaml
name: Deploy OpsTrack to Production

on:
  workflow_dispatch:
    inputs:
      confirm:
        description: "Confirmer le déploiement en production (oui)"
        required: true
        default: "non"

jobs:
  pre-checks:
    name: Contrôles préalables
    runs-on: ubuntu-latest
    steps:
      - name: Vérifier la confirmation
        run: |
          if [ "${{ github.event.inputs.confirm }}" != "oui" ]; then
            echo "Déploiement annulé."
            exit 1
          fi

      - name: Health check qualification
        run: |
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
            http://eval-dfs-q-tpl-20263-01.it-students.fr/api/health)
          [ "$STATUS" = "200" ] || exit 1

      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: "8.4"
      - run: composer install --no-dev --optimize-autoloader
      - run: php artisan test --env=testing

  deploy:
    name: Déploiement production
    runs-on: ubuntu-latest
    needs: pre-checks
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Configurer SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.PROD_SSH_KEY }}" > ~/.ssh/ubuntu.pem
          chmod 400 ~/.ssh/ubuntu.pem
          ssh-keyscan -H 35.180.47.223 >> ~/.ssh/known_hosts

      - name: Transférer le code
        run: |
          rsync -avz --exclude='.git' --exclude='node_modules' --exclude='.env' \
            -e "ssh -i ~/.ssh/ubuntu.pem" \
            ./ ubuntu@35.180.47.223:/var/www/opstrack/

      - name: Déployer Laravel
        run: |
          ssh -i ~/.ssh/ubuntu.pem ubuntu@35.180.47.223 << 'EOF'
            set -e
            cd /var/www/opstrack
            composer install --no-dev --optimize-autoloader
            php artisan migrate --force
            php artisan config:cache && php artisan route:cache && php artisan view:cache
            php artisan cache:clear
            sudo chown -R www-data:www-data /var/www/opstrack
            sudo chmod -R 775 storage bootstrap/cache
            sudo systemctl reload apache2
          EOF

  smoke-tests:
    name: Smoke tests
    runs-on: ubuntu-latest
    needs: deploy
    steps:
      - name: Test HTTPS
        run: |
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
            https://eval-dfs-p-tpl-20263-01.it-students.fr)
          [ "$STATUS" = "200" ] || exit 1

      - name: Test API health
        run: |
          curl -sf https://eval-dfs-p-tpl-20263-01.it-students.fr/api/health \
            | grep '"status":"ok"' || exit 1

      - name: Test certificat SSL
        run: |
          echo | openssl s_client \
            -connect eval-dfs-p-tpl-20263-01.it-students.fr:443 2>/dev/null \
            | openssl x509 -noout -dates
```
