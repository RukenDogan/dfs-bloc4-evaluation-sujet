# Journal de securite

Ce document recense les failles de securite identifiees pendant l'epreuve, leur evaluation et les mesures correctives appliquees.

## Faille 1

| Champ | Description |
| --- | --- |
| Date de detection | 28 avril 2026 |
| Composant concerne | `app/Http/Controllers/Api/TicketController.php` |
| Description de la faille | Injection SQL potentielle via `orWhereRaw` — le paramètre `$search` était interpolé directement dans une requête SQL brute sans utiliser les requêtes préparées |
| Severite estimee | `Haute` |
| Impact potentiel | Extraction non autorisée de données, contournement de l'authentification, corruption de la base de données |
| Mesure corrective appliquee | Remplacement de `orWhereRaw("reference like '%{$search}%'")` par `orWhere('reference', 'like', "%{$search}%")` — utilisation des requêtes préparées PDO via Eloquent |
| Statut | `Corrige` |
| Preuve de correction | Test avec payload `search='; DROP TABLE tickets; --` → HTTP 200 avec liste vide, aucune erreur SQL |

## Faille 2

| Champ | Description |
| --- | --- |
| Date de detection | 28 avril 2026 |
| Composant concerne | `app/Http/Controllers/WebhookController.php` |
| Description de la faille | Validation insuffisante du champ `status` dans le webhook — toute valeur de type `string` était acceptée, sans contrainte sur les valeurs autorisées |
| Severite estimee | `Moyenne` |
| Impact potentiel | Corruption des statuts de tickets avec des valeurs arbitraires, comportement imprévisible de l'application |
| Mesure corrective appliquee | Ajout de la règle de validation `in:new,scheduled,in_progress,resolved,closed` sur le champ `status` |
| Statut | `Corrige` |
| Preuve de correction | Test avec `status: "INVALIDE"` → HTTP 422 Unprocessable Entity |

## Faille 3

| Champ | Description |
| --- | --- |
| Date de detection | 28 avril 2026 |
| Composant concerne | `.env` de production |
| Description de la faille | `APP_DEBUG=true` actif en production — en cas d'erreur applicative, Laravel expose les traces complètes, les variables d'environnement et les credentials |
| Severite estimee | `Haute` |
| Impact potentiel | Fuite de credentials (mots de passe BDD, tokens API), exposition de la structure interne et des chemins serveur |
| Mesure corrective appliquee | `APP_DEBUG=false` dans le `.env` de production |
| Statut | `Corrige` |
| Preuve de correction | Déclenchement volontaire d'une erreur → page d'erreur générique sans informations sensibles |
