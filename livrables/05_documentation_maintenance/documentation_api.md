# Documentation d'API

> Si la documentation d'API n'est pas pertinente dans le contexte de l'epreuve, remplacer ce contenu par une note de justification.

## 1. Vue d'ensemble de l'API

| Champ | Valeur |
| --- | --- |
| URL de base | `https://eval-dfs-p-tpl-20263-01.it-students.fr/api` |
| Format | `JSON` |
| Authentification | Bearer Token (`Authorization: Bearer <token>`) ou `X-Api-Token: <token>` |

## 2. Endpoints disponibles

| Methode | Endpoint | Description | Authentification requise |
| --- | --- | --- | --- |
| GET | `/api/health` | Santé de l'application | Non |
| GET | `/api/v1/tickets` | Liste paginée des tickets | Oui |
| POST | `/api/v1/tickets` | Créer un ticket | Oui |
| GET | `/api/v1/tickets/{id}` | Détail d'un ticket | Oui |
| PUT/PATCH | `/api/v1/tickets/{id}` | Mettre à jour un ticket | Oui |
| GET | `/api/v1/technicians` | Liste des techniciens | Oui |
| GET | `/api/v1/external/weather` | Données météo pour un site | Oui |
| POST | `/hooks.php` | Webhook externe | HTTP Basic (`user:password`) |

## 3. Exemples de requetes et reponses

**GET /api/health**
```bash
curl https://eval-dfs-p-tpl-20263-01.it-students.fr/api/health
```
```json
{
  "status": "ok",
  "service": "OpsTrack",
  "timestamp": "2026-04-28T10:00:00+02:00"
}
```

**GET /api/v1/tickets** (avec filtres)
```bash
curl -H "Authorization: Bearer <token>" \
  "https://eval-dfs-p-tpl-20263-01.it-students.fr/api/v1/tickets?priority=critical&search=panne&per_page=5"
```
```json
{
  "data": [
    {
      "id": 1,
      "reference": "INC-001234",
      "title": "Panne équipement site A",
      "priority": "critical",
      "status": "in_progress",
      "site": { "id": 1, "name": "Site A", "city": "Paris" },
      "assigned_to": { "id": 2, "name": "Jean Dupont" }
    }
  ],
  "meta": { "current_page": 1, "per_page": 5, "total": 3 }
}
```

**POST /api/v1/tickets**
```bash
curl -X POST -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"site_id":1,"opened_by_user_id":1,"title":"Panne","description":"...","priority":"high"}' \
  https://eval-dfs-p-tpl-20263-01.it-students.fr/api/v1/tickets
```

**POST /hooks.php** (webhook)
```bash
curl -u user:password -X POST \
  -H "Content-Type: application/json" \
  -d '{"ticket_reference":"INC-001234","status":"resolved","summary":"Intervention terminée"}' \
  https://eval-dfs-p-tpl-20263-01.it-students.fr/hooks.php
```
```json
{ "message": "Webhook processed.", "intervention_id": 42 }
```

## 4. Codes d'erreur

| Code | Signification |
| --- | --- |
| `200` | Succès |
| `401` | Token manquant ou invalide |
| `404` | Ressource introuvable |
| `422` | Données de la requête invalides (détail dans `errors`) |
| `500` | Erreur serveur interne |
