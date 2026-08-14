# Plan d'Implémentation — Candidature Automatique Freelance

> Basé sur le code existant dans dan-claude-skills (core/services/job_scanner/)
> Objectif : 100% auto, zéro validation, reporting quotidien
> Intégration pipeline leads Postgres + LinkedIn Jobs

---

## 1. Architecture Actuelle (dan-claude-skills)

```
┌─────────────┐    ┌──────────────┐    ┌──────────┐    ┌───────────┐
│ FreeWork    │    │ Freelance-   │    │ Freelance │    │  Upwork   │
│ (scrape)    │    │ Info (scrape)│    │ map (scp) │    │ (GraphQL) │
└──────┬──────┘    └──────┬───────┘    └─────┬────┘    └─────┬─────┘
       │                  │                  │               │
       └──────────────────┴──────────────────┴───────────────┘
                              │
                     ┌────────▼────────┐
                     │   JobScanner    │
                     │  (orchestrator) │
                     └────────┬────────┘
                              │
                     ┌────────▼────────┐
                     │      Score      │
                     │  (adéquation)   │
                     └────────┬────────┘
                              │
                     ┌────────▼────────┐
                     │     Drafting    │
                     │  (génération)   │
                     └────────┬────────┘
                              │
                     ┌────────▼────────┐
                     │   Submission    │
                     │  (assisté)      │
                     └────────────────┘
```

## 2. Composants à Ajouter

### 2.1 Browser Automation (Playwright)
- **Outil :** Playwright (Python) — compatible avec le VPS, sans tête
- **Rôle :** Remplir et soumettre les formulaires de candidature
- **Sites cibles :** FreeWork, Freelance-Info, Freelancermap
- **Hors scope :** Upwork (API, pas de formulaire — manuel)

### 2.2 LinkedIn Jobs (Jina Reader + Agent-Reach)
- **Outil :** Jina Reader (déjà fonctionnel via Agent-Reach)
- **Rôle :** Scraper les offres LinkedIn Jobs correspondant au profil
- **Format :** Convertir en RawOffer (même modèle pivot)

### 2.3 Pipeline Leads (Postgres)
- **Table :** hermes.leads (déjà existante)
- **Nouveau statut :** `candidate` (candidature envoyée)
- **Mapping :** Mission freelance → Lead dans la pipeline

### 2.4 Cron de Candidature
- **Fréquence :** 1x/jour (9h Paris)
- **Actions :** Scan → Score → Draft → Submit → Report
- **Mode :** Full auto, pas de validation

### 2.5 Reporting Quotidien
- **Canal :** Telegram (via cron, delivery origin)
- **Contenu :** Nouvelles missions, candidatures envoyées, réponses reçues

---

## 3. Architecture Cible

```
┌─────────────┐    ┌──────────────┐    ┌──────────┐    ┌───────────┐    ┌──────────────┐
│ FreeWork    │    │ Freelance-   │    │ Freelance │    │  Upwork   │    │ LinkedIn     │
│ (scrape)    │    │ Info (scrape)│    │ map (scp) │    │ (GraphQL) │    │ Jobs (Reader)│
└──────┬──────┘    └──────┬───────┘    └─────┬────┘    └─────┬─────┘    └──────┬───────┘
       │                  │                  │               │                 │
       └──────────────────┴──────────────────┴───────────────┴─────────────────┘
                              │
                     ┌────────▼────────┐
                     │   JobScanner    │
                     └────────┬────────┘
                              │
                     ┌────────▼────────┐
                     │      Score      │
                     │  (seuil > 60%)  │
                     └────────┬────────┘
                              │
                     ┌────────▼────────┐
                     │     Drafting    │
                     └────────┬────────┘
                              │
                    ┌─────────▼─────────┐
                    │  Browser Auto     │
                    │  (Playwright)     │
                    │  → Submit form    │
                    └─────────┬─────────┘
                              │
                    ┌─────────▼─────────┐
                    │  Pipeline Leads   │
                    │  (hermes.leads)   │
                    │  status=applied   │
                    └─────────┬─────────┘
                              │
                    ┌─────────▼─────────┐
                    │  Reporting        │
                    │  (Telegram)       │
                    └───────────────────┘
```

## 4. Dépendances

| Dépendance | Rôle | Installation |
|------------|------|--------------|
| `playwright` | Browser automation | `pip install playwright && playwright install chromium` |
| `psycopg2` | Connexion Postgres | `pip install psycopg2-binary` |
| `httpx` | HTTP client | `pip install httpx` |
| `Agent-Reach` | LinkedIn Jobs | Déjà installé |
| `Jina Reader` | Web scraping | Déjà fonctionnel (r.jina.ai) |

## 5. Structure des Fichiers

```
/data/agents/hermes/workspace/auto-freelance/
├── scanner.py              # Wrapper JobScanner → pipeline
├── submitter.py            # Browser automation (Playwright)
├── linkedin_source.py      # LinkedIn Jobs via Jina Reader
├── pipeline.py             # Intégration Postgres (hermes.leads)
├── reporter.py             # Génération rapport Telegram
├── config.yaml             # Configuration (seuils, plateformes)
├── cron.sh                 # Script cron quotidien
└── requirements.txt        # Dépendances Python
```

## 6. Pipeline Postgres (hermes.leads)

Mapping des champs :

| RawOffer → | hermes.leads |
|-------------|-------------|
| source → | source (e.g. 'freework') |
| title → | position |
| company → | company |
| url → | notes (stockée) |
| daily_rate → | notes (stockée) |
| skills → | tags |
| external_id → | notes (source_id) |
| — | status = 'candidate' |

Nouveau statut : `candidate` = candidature envoyée automatiquement.

## 7. Cron Quotidien

```yaml
# /data/agents/hermes/cron/auto-freelance.yaml
name: Candidature automatique freelance
schedule: 0 7 * * *  # 9h Paris
script: /data/agents/hermes/workspace/auto-freelance/cron.sh
deliver: origin
```

Le cron :
1. Active le venv Python
2. Lance `python3 scanner.py` → scan toutes les sources
3. Score les missions → garde > 60%
4. `python3 submitter.py` → soumet via Playwright
5. `python3 pipeline.py` → écrit dans hermes.leads
6. `python3 reporter.py` → rapport Telegram

## 8. Plan d'Implémentation

### Phase 1 — Infrastructure (1h)
1. Installer Playwright : `pip install playwright && playwright install chromium`
2. Créer le dossier `/data/agents/hermes/workspace/auto-freelance/`
3. Tester la connexion Postgres (déjà OK via docker exec postgres)
4. Tester Jina Reader sur LinkedIn Jobs (déjà OK)

### Phase 2 — Submitter (2h)
1. Inspecter le formulaire FreeWork → cartographier les champs
2. Écrire le submitter Playwright pour FreeWork
3. Inspecter Freelance-Info → cartographier
4. Écrire le submitter pour Freelance-Info
5. Inspecter Freelancermap → cartographier
6. Écrire le submitter pour Freelancermap
7. Tester chaque soumission en mode headless

### Phase 3 — LinkedIn Jobs (1h)
1. Écrire le source LinkedIn Jobs (via Jina Reader)
2. Convertir les résultats en RawOffer
3. Intégrer dans le JobScanner

### Phase 4 — Pipeline (30min)
1. Écrire pipeline.py → insertion dans hermes.leads
2. Mapping des champs
3. Gestion des doublons (ON CONFLICT)

### Phase 5 — Cron + Reporting (30min)
1. Écrire cron.sh
2. Écrire reporter.py
3. Tester le cron
4. Vérifier la livraison Telegram

---

## 9. Risques et Mitigations

| Risque | Mitigation |
|--------|------------|
| Blocage CAPTCHA | Rate limiting + rotation user-agent |
| Évolution des formulaires | Tests réguliers + reporting d'erreur |
| Doublons de candidature | Déduplication via external_id |
| Upwork suspendu | Manuel seulement (API ne permet pas) |
| LinkedIn Jobs rate limit | Jina Reader gère les retries |

## 10. Métriques de Suivi

- Missions scannées / jour
- Candidatures envoyées / jour
- Taux de succès soumission (submit réussi / tenté)
- Leads générés (entrées dans hermes.leads)
- Réponses reçues (via email/linkedin)