# Architecture cible et choix de l'hebergement

> Competence evaluee : `C29` — Selectionner une plateforme d'hebergement adaptee aux exigences techniques, economiques, qualitatives et reglementaires.

## 1. Analyse des besoins techniques

L'application OpsTrack Field Service repose sur les composants suivants :

- **Laravel 12 / PHP 8.4** : application principale, API REST et traitement du webhook `hooks.php`
- **MySQL 8** : données transactionnelles (tickets, interventions, utilisateurs, sites, clients)
- **MongoDB 8** : journaux techniques et événements applicatifs
- **Redis 7** : cache applicatif et stockages temporaires
- **Next.js** : microservice `dispatch-dashboard` consommant l'API Laravel
- **API publique tierce** : Open-Meteo pour l'enrichissement de données météo
- **Webhook** : `hooks.php` appelé chaque minute par un système externe

La population cible est estimée à **50-200 utilisateurs actifs** (techniciens terrain, superviseurs, administrateurs) dans un contexte de PME en croissance, avec des pics d'activité liés aux interventions terrain.

## 2. Architecture cible proposee

### 2.1 Diagramme de deploiement

```
Internet
    │
    ▼
┌─────────────────────────────────────────────────┐
│  CloudFront (CDN + WAF)                         │
│  - Distribution des assets statiques            │
│  - Protection applicative (WAF)                 │
│  - Terminaison HTTPS / certificat ACM           │
└────────────────────┬────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────┐
│  VPC  eu-west-3                                 │
│                                                 │
│  ┌──────────────────────────────────────────┐   │
│  │  Subnet public                           │   │
│  │  Application Load Balancer (ALB)         │   │
│  └──────────────┬───────────────────────────┘   │
│                 │                               │
│  ┌──────────────▼───────────────────────────┐   │
│  │  Subnet privé - Couche applicative       │   │
│  │                                          │   │
│  │  Auto Scaling Group                      │   │
│  │  ┌────────────┐  ┌────────────┐          │   │
│  │  │EC2 t3.small│  │EC2 t3.small│          │   │
│  │  │Laravel +   │  │Laravel +   │          │   │
│  │  │Apache      │  │Apache      │          │   │
│  │  └────────────┘  └────────────┘          │   │
│  │                                          │   │
│  │  ┌────────────────────────────────────┐  │   │
│  │  │ ECS Fargate                        │  │   │
│  │  │ Microservice Next.js               │  │   │
│  │  │ dispatch-dashboard                 │  │   │
│  │  └────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────┘   │
│                                                 │
│  ┌──────────────────────────────────────────┐   │
│  │  Subnet privé - Couche données           │   │
│  │                                          │   │
│  │  ┌──────────┐  ┌──────────┐  ┌────────┐  │   │
│  │  │RDS MySQL │  │DocumentDB│  │Elasti- │  │   │
│  │  │Multi-AZ  │  │(MongoDB) │  │Cache   │  │   │
│  │  └──────────┘  └──────────┘  └────────┘  │   │
│  └──────────────────────────────────────────┘   │
│                                                 │
│  ┌──────────────────────────────────────────┐   │
│  │  Services transverses                    │   │
│  │  S3 · CloudWatch · AWS Backup            │   │
│  │  Secrets Manager · ACM                   │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### 2.2 Description des composants

| Composant | Service ou technologie | Dimensionnement | Justification |
| --- | --- | --- | --- |
| Serveurs applicatifs Laravel | EC2 t3.small (Auto Scaling Group) | Min 1 / Max 3 instances | Élasticité selon la charge, coût maîtrisé |
| Microservice Next.js | ECS Fargate | 0,25 vCPU / 0,5 Go RAM | Serverless, scalabilité automatique, pas de gestion serveur |
| Base MySQL | RDS MySQL 8 Multi-AZ | db.t3.small, 20 Go | Haute disponibilité, sauvegardes automatiques, patches managés |
| Base MongoDB | Amazon DocumentDB | db.t3.medium, 10 Go | Compatible MongoDB, managé, sauvegarde intégrée |
| Cache Redis | ElastiCache for Redis | cache.t3.micro, 1 nœud | Managé, haute disponibilité |
| Assets statiques | S3 + CloudFront | 10 Go stockage | Distribution mondiale, faible latence, coût minimal |
| Certificat SSL/TLS | AWS Certificate Manager | — | Certificats gratuits, renouvellement automatique |
| Protection applicative | AWS WAF | 1 WebACL | Filtrage des requêtes malveillantes |
| Supervision | CloudWatch | — | Centralisation logs, métriques, alertes |
| Sauvegardes | AWS Backup | Rétention 7 jours | Politique centralisée |
| Secrets | AWS Secrets Manager | ~10 secrets | Gestion sécurisée des credentials |
| Réseau | VPC subnets publics/privés | — | Isolation des composants, sécurité en profondeur |
| Équilibrage de charge | Application Load Balancer | — | Répartition du trafic, terminaison SSL |

## 3. Choix du fournisseur et des services

### 3.1 Fournisseur retenu

**Amazon Web Services (AWS)**, région `eu-west-3` (Paris, France).

### 3.2 Justification du choix

- **Cohérence avec l'environnement existant** : l'évaluation tourne déjà sur AWS EC2 en région `eu-west-3`, ce qui facilite la migration et réduit les risques
- **Localisation européenne** : la région Paris garantit l'hébergement des données sur le territoire européen, sans transfert hors UE, conforme au RGPD
- **Complétude de la stack** : AWS propose l'ensemble des services managés nécessaires (RDS, DocumentDB, ElastiCache, ECS Fargate, CloudFront)
- **Maturité opérationnelle** : outils de supervision (CloudWatch), de sécurité (IAM, WAF, Security Groups) et de sauvegarde (AWS Backup) couvrant l'ensemble des exigences
- **Modèle de facturation à l'usage** : adapté à une PME en croissance

## 4. Estimation des couts

| Poste de depense | Cout mensuel estime | Cout annuel estime |
| --- | --- | --- |
| EC2 Auto Scaling (1,5 instance t3.small en moyenne) | 35 € | 420 € |
| RDS MySQL Multi-AZ (db.t3.small, 20 Go) | 55 € | 660 € |
| Amazon DocumentDB (db.t3.medium, 10 Go) | 70 € | 840 € |
| ElastiCache Redis (cache.t3.micro) | 20 € | 240 € |
| ECS Fargate (Next.js, 0,25 vCPU / 0,5 Go) | 10 € | 120 € |
| Application Load Balancer | 20 € | 240 € |
| S3 + CloudFront (10 Go, 50 Go transfert) | 5 € | 60 € |
| AWS WAF (1 WebACL) | 10 € | 120 € |
| CloudWatch (logs + métriques) | 10 € | 120 € |
| AWS Backup (RDS + DocumentDB) | 10 € | 120 € |
| Secrets Manager (~10 secrets) | 5 € | 60 € |
| **Total** | **~250 €/mois** | **~3 000 €/an** |

## 5. Elasticite et evolutivite

L'Auto Scaling Group ajuste automatiquement le nombre d'instances EC2 selon la charge CPU (seuil : 70 %). En période creuse, une seule instance suffit ; en pic d'activité, jusqu'à 3 instances sont provisionnées automatiquement.

ECS Fargate scale le microservice Next.js sans intervention manuelle. RDS et ElastiCache peuvent être redimensionnés sans interruption de service via la console AWS.

## 6. Disponibilite et continuite de service

- **Cible de disponibilité** : 99,9 % (environ 8h d'indisponibilité par an)
- **RDS Multi-AZ** : basculement automatique en cas de défaillance de l'instance principale
- **ALB** : distribue le trafic entre les instances EC2 saines et détecte les instances défaillantes
- **ElastiCache** : mode cluster avec réplication pour la tolérance aux pannes
- **AWS Backup** : sauvegardes automatiques quotidiennes avec rétention 7 jours

## 7. Securite et sauvegarde

- **Isolation réseau** : bases de données dans des subnets privés, inaccessibles depuis Internet
- **Secrets** : credentials gérés via AWS Secrets Manager (aucun secret en clair dans `.env`)
- **Chiffrement** : au repos (RDS, S3, DocumentDB) et en transit (TLS 1.2+)
- **WAF** : protection contre les injections SQL, XSS et autres attaques applicatives
- **IAM** : principe du moindre privilège pour chaque service
- **Sauvegardes** : AWS Backup avec politique centralisée, rétention 7 jours, restauration testée

## 8. Conformite et contraintes reglementaires

- **RGPD** : toutes les données hébergées en région `eu-west-3` (Paris), sans transfert hors UE
- **Certifications AWS** : ISO 27001, SOC 2 Type II, HDS — AWS propose des DPA conformes au RGPD
- **Traçabilité** : CloudTrail enregistre toutes les actions sur l'infrastructure ; CloudWatch Logs centralise les logs applicatifs
- **Rétention des données** : politique configurable via AWS Backup selon les exigences métier
- **Localisation** : les logs MongoDB (événements applicatifs) restent dans le périmètre européen
