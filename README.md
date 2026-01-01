# 🚪 Hall - Orchestrateur d'entrée Auditio

Gateway intelligente multi-domaines pour l'orchestration de serveurs avec réveil à la demande (WoL) et gestion des projets de testing.

## Vue d'ensemble

Hall est une application Flask qui fournit :

- Une **gateway web** pour accéder à plusieurs domaines/projets
- Un **système de réveil à la demande** (WoL) pour réveiller les serveurs inactifs
- Une **page d'attente** affichée pendant le démarrage du service
- Un **tableau de bord d'administration** pour la gestion et le monitoring
- Un **système de gestion des projets de testing** avec authentification
- Un **reverse proxy** (Traefik) pour la gestion des certificats SSL et du routage

## Structure du projet

```md
hall/
├── docker-compose.yml                  # Orchestration des services (Flask, Traefik, WoL API)
├── Dockerfile.flask                    # Image pour l'application Flask Gateway
├── Dockerfile.traefik                  # Image Traefik personnalisée
├── .env.exemple                        # Variables d'environnement (à copier en .env)
├── hall-service.sh                     # Script de gestion du service systemd
├── test-https.sh                       # Script de test HTTPS
├── SERVICE.md                          # Documentation du service systemd
├── CERTIFICATES.md                     # Documentation gestion des certificats
├── WOL_CHECKLIST.md                    # Checklist de configuration WoL
├── wol_persistant.sh                   # Script pour rendre le WoL persistant
├── app/                                # Application Flask
│   ├── app.py                          # Point d'entrée Flask (factory pattern)
│   ├── api_bp.py                       # Blueprint routes API et pages
│   ├── admin_bp.py                     # Blueprint pour /admin
│   ├── testing_bp.py                   # Blueprint pour gestion testing
│   ├── config.py                       # Chargement configuration domains.json
│   ├── database.py                     # Gestion SQLite (logs, activité)
│   ├── functions.py                    # Fonctions utilitaires
│   ├── wol.py                          # Logique WoL et vérifications réseau
│   ├── logging_utils.py                # Configuration du logging
│   ├── requirements.txt                # Dépendances Python (Flask, gunicorn, requests, httpx)
│   ├── static/
│   │   ├── css/                        # Styles (admin, base, waiting, testing)
│   │   └── js/                         # Scripts frontend (admin, waiting)
│   └── templates/                      # Templates HTML
│       ├── index.html                  # Page d'accueil
│       ├── waiting.html                # Page d'attente (WoL en cours)
│       ├── testing_login.html          # Login pour projets testing
│       ├── admin_login.html            # Login admin
│       ├── admin.html                  # Dashboard admin
│       ├── admin_testing.html          # Gestion des projets testing
│       └── admin_testing_form.html     # Formulaire testing
├── config/
│   ├── domains.json                    # Configuration des domaines/serveurs
│   └── domains.schema.json             # Schéma JSON pour validation
├── traefik/                            # Configuration Traefik (reverse proxy)
│   ├── traefik.yml                     # Configuration Traefik
│   ├── dynamic/
│   │   └── routes.yml                  # Routes dynamiques
│   └── acme/                           # Stockage certificats Let's Encrypt
├── wol-dedicated/                      # API WoL séparée (conteneur dédié)
│   ├── Dockerfile                      # Image pour API WoL
│   └── wol_api.py                      # API WoL (port 5001)
├── log/                                # Répertoire des logs
│   ├── erp/
│   └── testing/
└── (BD SQLite auto-créée)
```

## Démarrage rapide

### 1. Configuration initiale

```bash
# Copier et configurer les variables d'environnement
cp .env.exemple .env
# Éditer .env avec vos paramètres (admin, WoL, etc.)
```

### 2. Configuration des domaines

Éditer `config/domains.json` pour définir vos serveurs/domaines avec :

- IP serveur
- Adresse MAC (pour WoL)
- URL de redirection
- Health check
- Politiques de réveil

### 3. Lancer les services

```bash
docker-compose up -d
```

### 4. Vérifier (optionnel)

```bash
# Via le script de service
./hall-service.sh status

# Ou manuellement
curl http://localhost
curl http://localhost:8080/dashboard/
```

## Services Docker

| Service | Port | Rôle |
| - | - | - |
| **Flask Gateway** | 5000 | Application web principale |
| **Traefik** | 80, 443, 8080 | Reverse proxy, SSL/TLS, dashboard |
| **WoL API** | 5001 | API Wake-on-LAN (conteneur dédié) |
| **SQLite** | N/A | Base de données des logs et activités |

## Routes principales

### Pages web

- `GET /` → Page d'accueil ou sélection domaine
- `GET /<domain>` → Page d'attente (si serveur inactif)
- `GET /admin/login` → Login administrateur
- `GET /admin` → Dashboard admin (avec gestion testing, logs, etc.)
- `GET /testing/login` → Login pour projets testing
- `GET /testing/<project>` → Page projet testing

### API

- `GET /api/status/<domain>` → État détaillé (serveur en ligne, service prêt, etc.)
- `POST /api/wake/<domain>` → Déclenche WoL + redirige si prêt
- `POST /api/testing/<project>/wake` → WoL pour projet testing
- `GET /api/health` → Health check application

## Architectures des dépendances

### Application Flask

- **Framework** : Flask 3.1.2
- **Serveur** : Gunicorn 23.0.0
- **Requêtes HTTP** : requests, httpx
- **Configuration** : python-dotenv

### Infrastructure

- **Reverse proxy** : Traefik (certificats SSL auto, routage HTTP(S))
- **Base de données** : SQLite (persistance des logs et activité)
- **Orchestration** : Docker Compose

### WoL

- API WoL dédiée dans `wol-dedicated/` (isolation réseau/sécurité)
- Envoie paquets Wake-on-LAN via UDP broadcast
- Appels via requests avec authentification (X-API-KEY)
- Nécessaire pour faire le pont entre les réseaux

## Configuration avancée

### Gestion du service systemd

Voir [SERVICE.md](SERVICE.md) pour :

- Démarrage automatique au boot
- Commandes systemctl
- Gestion via script `hall-service.sh`
- Dépannage

### Certificats SSL

Voir [CERTIFICATES.md](CERTIFICATES.md) pour :

- Configuration Let's Encrypt
- Renouvellement automatique
- Domaines multiples

### Configuration WoL

Voir [WOL_CHECKLIST.md](WOL_CHECKLIST.md) pour :

- Test WoL
- Configuration MAC adresses
- Politique de réveil automatique
- Scripts persistants

## Fonctionnalités principales

### 1. Multi-domaines avec configuration flexible

- Configuration centralisée dans `config/domains.json`
- Support multiple serveurs/projets
- Policies de réveil configurables

### 2. Page d'attente intelligente

- Affichée si serveur offline
- Polling automatique du statut (configurable)
- Redirection transparente quand serveur prêt

### 3. Wake-on-LAN (WoL)

- Réveil automatique ou manuel
- Vérification de l'IP et health check après réveil
- Logs détaillés par domaine
- API WoL dédiée dans conteneur séparé

### 4. Dashboard administrateur

- Logs détaillés par domaine
- Activité en temps réel
- Test manuels (ping, WoL)
- Gestion des projets de testing

### 5. Projets de testing

- Création/modification/suppression via admin
- Authentification par token
- Logs d'accès séparés
- Interface dédiée

### 6. Reverse proxy Traefik

- Certificats SSL automatiques (Let's Encrypt)
- Routage multi-domaine
- Dashboard Traefik (port 8080)
- Configuration dynamique

## Logging et monitoring

- **Logs application** : fichiers dans `log/`
- **Base de données** : SQLite stocke les logs et l'activité par domaine
- **Logs systemd** : `journalctl -u hall-auditio.service` (si service systemd)
- **Dashboard Traefik** : `http://localhost:8080/dashboard/`

## Dépannage rapide

| Problème | Solution |
| - | - |
| Serveur ne se réveille pas | Vérifier MAC, IP broadcast, configuration WoL |
| Certificats SSL non renouvelés | Vérifier logs Traefik, permissions acme/ |
| Page d'attente ne redirige pas | Vérifier health check URL, configuration domains.json |
| Admin inaccessible | Vérifier ADMIN_PASSWORD env var |

Voir [SERVICE.md](SERVICE.md) et [CERTIFICATES.md](CERTIFICATES.md) pour plus de détails.
