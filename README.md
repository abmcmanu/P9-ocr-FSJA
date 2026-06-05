<p align="center">
  <img src="./front/src/favicon.png" width="96px" />
</p>

# MicroCRM – Mettez en œuvre l'intégration et le déploiement continu d'une application Full-Stack

Application CRM interne Orion – Back-end Spring Boot 3 · Front-end Angular 17
---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Structure du dépôt](#2-structure-du-dépôt)
3. [Lancer l'application localement](#3-lancer-lapplication-localement)
4. [Conteneurisation Docker](#4-conteneurisation-docker)
5. [Pipeline CI/CD – GitHub Actions](#5-pipeline-cicd--github-actions)
6. [Configuration des secrets GitHub](#6-configuration-des-secrets-github)
7. [Analyse qualité SonarQube Cloud](#7-analyse-qualité-sonarqube-cloud)
8. [Versioning sémantique & releases](#8-versioning-sémantique--releases)
9. [Plan de sécurité](#9-plan-de-sécurité)
10. [Plan de testing périodique](#10-plan-de-testing-périodique)

---

## 1. Vue d'ensemble

MicroCRM est une application interne de gestion de contacts (Personnes et Organisations). Le pipeline CI/CD automatise :

- l'exécution des **tests** (back + front) à chaque push/PR,
- l'**analyse de qualité** via SonarQube Cloud,
- la **publication d'images Docker** sur GitHub Container Registry (GHCR),
- la **création de releases GitHub** versionnées (SemVer).

---

## 2. Structure du dépôt

```
.
├── .github/
│   └── workflows/
│       ├── ci.yml          # Build, tests, SonarQube
│       ├── cd.yml          # Build et push Docker (merge main)
│       └── release.yml     # Création release GitHub (tag vX.Y.Z)
├── back/
│   ├── Dockerfile          # Image back-end (amazoncorretto:17-alpine)
│   └── src/
├── front/
│   ├── Dockerfile          # Image front-end (node:20-alpine + caddy:2-alpine)
│   └── src/
├── elk/                    # Configuration Logstash (pipeline + settings)
├── misc/
│   └── docker/
│       ├── Caddyfile       # Config serveur Caddy (image standalone)
│       └── supervisor.ini  # Config supervisord (image standalone)
├── docker-compose.yml      # Orchestration app (back + front)
├── docker-compose-elk.yml  # Stack ELK (Elasticsearch, Logstash, Kibana)
└── README.md
```

---

## 3. Lancer l'application localement

### Prérequis

| Outil | Version minimale |
|-------|-----------------|
| Java (OpenJDK / Temurin) | 17 |
| Node.js | 20 |
| npm | 10 |
| Google Chrome | récent |
| Docker Desktop | 24+ |

### Règles importantes en local

> Ces points évitent les erreurs les plus courantes avant de lancer quoi que ce soit.

| # | Règle | Détail |
|---|-------|--------|
| 1 | **Permissions `gradlew`** | Après un `git clone`, le wrapper peut ne pas être exécutable : `chmod +x back/gradlew` |
| 2 | **Apple Silicon (ARM64)** | L'image `eclipse-temurin:17-jre-alpine` n'a pas de manifest ARM64. Le `back/Dockerfile` utilise `amazoncorretto:17-alpine` qui supporte les deux architectures. |
| 3 | **Ordre de démarrage avec ELK** | `docker compose up` **avant** `docker compose -f docker-compose-elk.yml up` — le premier crée le réseau `microcrm-net` dont ELK a besoin. |
| 4 | **Profil `elk`** | L'appender Logstash n'est actif que si `SPRING_PROFILES_ACTIVE=elk`. Sans ce profil, aucune tentative de connexion à Logstash n'est faite. |
| 5 | **RAM pour ELK** | La stack ELK nécessite ~4 Go de RAM disponibles (Elasticsearch 1 Go + Logstash 512 Mo + Kibana). |

---

### Back-end

```bash
cd back
chmod +x gradlew          # si nécessaire après un clone
./gradlew build           # compile + tests
java -jar build/libs/microcrm-0.0.1-SNAPSHOT.jar
# → http://localhost:8080
```

### Front-end

```bash
cd front
npm ci
npx ng serve
# → http://localhost:4200
```

### Exécution des tests

**Back-end :**
```bash
cd back
./gradlew test
# Rapport : back/build/reports/tests/test/index.html
```

**Front-end :**
```bash
cd front
CHROME_BIN=/usr/bin/google-chrome npx ng test --watch=false --browsers=ChromeHeadlessNoSandbox
# Rapport de couverture : front/coverage/microcrm/
```

---

## 4. Conteneurisation Docker

### Images disponibles

| Service | Dockerfile | Image de base | Port |
|---------|-----------|---------------|------|
| Back-end | `back/Dockerfile` | `amazoncorretto:17-alpine` (amd64 + arm64) | 8080 |
| Front-end | `front/Dockerfile` | `node:20-alpine` + `caddy:2-alpine` | 80 |

### Docker Compose — sans ELK (mode par défaut)

```bash
# Démarrer les deux services
docker compose up --build

# En arrière-plan
docker compose up -d --build

# Arrêter
docker compose down
```

- Front-end : **http://localhost**
- Back-end (API) : **http://localhost:8080**

### Docker Compose — avec stack ELK

L'observabilité repose sur deux fichiers Compose distincts reliés par un réseau partagé.
**`docker-compose.yml` doit démarrer en premier** pour créer le réseau `microcrm-net`.

```bash
# 1. Démarrer l'application (crée le réseau microcrm-net)
docker compose up -d --build

# 2. Démarrer la stack ELK (rejoint microcrm-net)
docker compose -f docker-compose-elk.yml up -d

# 3. Redémarrer le back avec le profil elk (active l'appender Logstash)
SPRING_PROFILES_ACTIVE=elk docker compose up -d --no-deps back
```

| Interface | URL |
|-----------|-----|
| Kibana | http://localhost:5601 |
| Elasticsearch | http://localhost:9200 |
| Logstash TCP input | localhost:5001 |

```bash
# Arrêter les deux stacks
docker compose down
docker compose -f docker-compose-elk.yml down
```

### Build manuel des images

```bash
# Back-end
docker build -t orion-microcrm-back:latest ./back

# Front-end
docker build -t orion-microcrm-front:latest ./front
```

### Image standalone (legacy – Dockerfile racine)

L'image standalone originale combine les deux services via supervisord :

```bash
docker build --target standalone -t orion-microcrm-standalone:latest .
docker run -it --rm -p 8080:8080 -p 80:80 orion-microcrm-standalone:latest
```

---

## 5. Pipeline CI/CD – GitHub Actions

### Vue d'ensemble des workflows

```
Push / PR
    │
    ▼
┌─────────────────────────────────────────┐
│  ci.yml – Intégration Continue          │
│  • test-back  (Gradle JUnit)            │
│  • test-front (Karma ChromeHeadless)    │
│  • build-back (JAR)                     │
│  • build-front (Angular dist)           │
│  • sonarqube  (SonarQube Cloud)         │
└─────────────────────────────────────────┘
    │ (merge sur main uniquement)
    ▼
┌─────────────────────────────────────────┐
│  cd.yml – Déploiement Continu           │
│  • docker-back  → GHCR                  │
│  • docker-front → GHCR                  │
└─────────────────────────────────────────┘
    │ (tag vX.Y.Z uniquement)
    ▼
┌─────────────────────────────────────────┐
│  release.yml – Release                  │
│  • Build JAR + dist Angular             │
│  • Création GitHub Release              │
│  • Images Docker taguées → GHCR         │
└─────────────────────────────────────────┘
```

### Déclencheurs

| Workflow | Déclencheur |
|----------|-------------|
| `ci.yml` | Push ou PR vers `main` / `develop` |
| `cd.yml` | Push (merge) vers `main` |
| `release.yml` | Push d'un tag `v*.*.*` |

### Commandes importantes

| Commande | Objectif | Défini dans | Moment |
|----------|----------|-------------|--------|
| `./gradlew test` | Tests JUnit back-end | `back/build.gradle` | CI, local |
| `./gradlew build -x test` | Build JAR sans tests | `back/build.gradle` | CI, release |
| `npx ng test --watch=false` | Tests Karma front-end | `front/package.json` | CI, local |
| `npx ng build --configuration=production` | Bundle Angular prod | `front/package.json` | CI, release |
| `./gradlew sonar` | Analyse SonarQube | `back/build.gradle` | CI |
| `docker compose up --build` | Démarrage complet | `docker-compose.yml` | local, CD |

---

## 6. Configuration des secrets GitHub

Rendez-vous dans **Settings → Secrets and variables → Actions** de votre dépôt.

| Secret | Description | Workflow |
|--------|-------------|---------|
| `SONAR_TOKEN` | Token d'authentification SonarQube Cloud | CI |
| `SONAR_PROJECT_KEY` | Clé du projet SonarQube (ex. `orion_microcrm`) | CI |
| `SONAR_ORGANIZATION` | Organisation SonarQube Cloud | CI |
| `GITHUB_TOKEN` | Fourni automatiquement par GitHub Actions | CD / Release |
| `DEPLOY_HOST` | IP/hostname du serveur cible (déploiement SSH optionnel) | CD |
| `DEPLOY_USER` | Utilisateur SSH du serveur cible | CD |
| `DEPLOY_SSH_KEY` | Clé privée SSH (PEM) | CD |

> **Ne jamais stocker de secrets en clair dans les fichiers YAML de workflow.**

### Obtenir un token SonarQube Cloud

1. Connectez-vous sur [sonarcloud.io](https://sonarcloud.io)
2. Mon compte → Security → Generate Token
3. Ajoutez le token comme secret `SONAR_TOKEN` dans GitHub

---

## 7. Analyse qualité SonarQube Cloud

### Ajouter le plugin SonarQube à Gradle

Dans `back/build.gradle`, ajoutez :

```groovy
plugins {
    // ... plugins existants
    id "org.sonarqube" version "5.0.0.4638"
    id 'jacoco'
}

jacoco {
    toolVersion = "0.8.11"
}

test {
    useJUnitPlatform()
    finalizedBy jacocoTestReport
}

jacocoTestReport {
    dependsOn test
    reports {
        xml.required = true
    }
}
```

### Métriques surveillées

| Catégorie | Seuil recommandé |
|-----------|-----------------|
| Couverture de code | ≥ 80 % |
| Duplication | < 3 % |
| Bugs | 0 bloquant/critique |
| Vulnérabilités | 0 haute/critique |
| Code Smells | Note A |
| Security Hotspots | Tous revus |

---

## 8. Versioning sémantique & releases

### Convention SemVer

```
vMAJOR.MINOR.PATCH
  │      │     └── Correctif rétrocompatible
  │      └────── Nouvelle fonctionnalité rétrocompatible
  └──────────── Rupture de compatibilité
```

### Créer une release

```bash
# 1. S'assurer d'être sur main à jour
git checkout main && git pull

# 2. Créer et pousser le tag
git tag v1.0.0
git push origin v1.0.0
# → Le workflow release.yml se déclenche automatiquement
```

### Politique de versioning

- Une **action humaine** (création d'un tag) est requise pour déclencher une release.
- Pas de release candidate automatique à chaque commit.
- Les branches de release ne sont pas créées : `main` est la branche de production.
- Un tag `-rc.N` (ex. `v1.0.0-rc.1`) génère une **pre-release** GitHub automatiquement.

---

## 9. Plan de sécurité

| Pratique | Mise en œuvre |
|----------|--------------|
| Analyse SAST | SonarQube Cloud (OWASP Top 10) sur chaque push |
| Gestion des secrets | GitHub Secrets uniquement, jamais en clair |
| Images minimales | `amazoncorretto:17-alpine`, `caddy:2-alpine` |
| Utilisateur non-root | `adduser appuser` dans le Dockerfile back |
| Audit dépendances npm | `npm audit` recommandé en CI |
| Token avec portée minimale | `GITHUB_TOKEN` scope limité par job |
| Exclusion fichiers sensibles | `.dockerignore` couvre `.git`, `*.env`, `node_modules` |

---

## 10. Plan de testing périodique

| Type de test | Outils | Déclenchement | Objectif |
|-------------|--------|---------------|----------|
| Tests unitaires back | JUnit 5, Spring Boot Test | Push, PR | Validation fonctionnelle |
| Tests intégration back | `@DataJpaTest` | Push, PR | Non-régression couche données |
| Tests unitaires front | Karma, Jasmine | Push, PR | Validation composants Angular |
| Analyse qualité | SonarQube Cloud | Push, PR | Qualité et sécurité du code |
| Tests de build Docker | `docker compose up` | Merge main (CD) | Validité des images |
| Tests programmés | (à implémenter) | Hebdomadaire | Stabilité long terme |

### Ajouter un test hebdomadaire programmé

Dans `ci.yml`, section `on` :

```yaml
on:
  schedule:
    - cron: '0 6 * * 1'  # Chaque lundi à 6h UTC
```

---

*Projet Orion MicroCRM – Pipeline CI/CD · Juin 2026*
