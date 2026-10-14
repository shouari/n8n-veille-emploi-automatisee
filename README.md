# Veille emploi automatisée (n8n)

Workflow n8n qui va chercher chaque jour des offres d'emploi pertinentes selon un profil défini, élimine les doublons par rapport à un historique réel, et envoie un résumé par courriel — sans jamais avoir à ouvrir un site d'offres d'emploi manuellement.

Construit en autodidacte, pour apprendre n8n sur un cas d'usage réel plutôt que sur un tutoriel. Je le partage tel quel, avec les erreurs qu'on a dû corriger en cours de route — ça devrait vous faire gagner du temps.

## Ce que ça fait

1. Chaque jour à une heure fixe, lance plusieurs recherches d'offres ciblées (une liste de requêtes que vous définissez)
2. Filtre et note chaque offre selon des mots-clés, une liste de villes et un seuil de salaire
3. Compare les résultats à un Google Sheet existant — seules les offres **jamais vues** passent
4. Écrit les nouvelles offres dans le Sheet (avec une colonne de statut : non lue / soumise / ignorée)
5. Envoie un résumé par courriel, seulement s'il y a du nouveau ce jour-là

## Architecture

```
Schedule Trigger (quotidien)
        │
        ▼
Code Python — génère la liste de requêtes
        │
        ▼
Loop Over Items ──► HTTP Request (recherche d'offres) ──► Code Python (scoring) ──┐
        │  ▲                                                                      │
        │  └──────────────────────── retour boucle ───────────────────────────────┘
        │
        └──(fin de boucle)──┬──► Compare Datasets (candidats vs historique)
                             └──► Google Sheets — lecture de l'historique
                                              │
                      Compare Datasets ──(nouveautés seulement)──┬──► Google Sheets — écriture
                                                                   └──► Code Python (HTML) ──► If ──► Gmail
```

La boucle de collecte et la phase de consolidation sont volontairement séparées. Les combiner cause un blocage circulaire silencieux — c'est l'erreur la plus piégeuse qu'on a rencontrée en le construisant, détaillée plus bas.

## Prérequis

| Élément | Pourquoi |
|---|---|
| Une instance n8n (cloud ou self-hosted) | Fait tourner le workflow |
| Un compte RapidAPI avec une souscription à l'API JSearch | Source des offres d'emploi |
| Un compte Google (Sheets + Gmail activés) | Stockage et notification |

Ce README ne couvre pas l'installation de n8n lui-même — suivez la [documentation officielle n8n](https://docs.n8n.io/) pour ça.

**Important si vous êtes self-hosted :** les nœuds de code de ce workflow sont en Python natif. Depuis n8n 2.x, ça nécessite un *external task runner* configuré séparément — sans ça, vous aurez l'erreur `Python runner unavailable: Python 3 is missing from this system`. Voir [la documentation n8n sur les task runners](https://docs.n8n.io/hosting/configuration/task-runners/#setting-up-external-mode). Sur n8n cloud, ce n'est pas un problème — c'est déjà configuré.

## Installation

### 1. Importer le workflow

Dans n8n : **Workflows → Import from File**, sélectionnez `veille-emploi-n8n.json`.

### 2. Créer le Google Sheet

Créez un Google Sheet avec un onglet contenant exactement ces en-têtes en ligne 1 :

| A | B | C | D | E | F | G | H | I |
|---|---|---|---|---|---|---|---|---|
| job_id | title | company | location | salaire | score | url | date_ajout | application_status |

### 3. Obtenir une clé API JSearch

Sur [RapidAPI, page JSearch](https://rapidapi.com/letscrape-6bRBa3QguO5/api/jsearch), abonnez-vous au plan **BASIC** (gratuit, 200 requêtes/mois, aucune carte requise).

### 4. Configurer les credentials dans n8n

| Nœud | Credential à créer |
|---|---|
| HTTP Request | Aucune credential n8n — collez votre clé RapidAPI directement dans l'en-tête `x-rapidapi-key` du nœud |
| Get row(s) in sheet / Append row in sheet | Credential Google Sheets (OAuth2) |
| Send a message | Credential Gmail (OAuth2) |

Pour les deux credentials Google, vous devrez créer un projet dans Google Cloud Console, activer les APIs Sheets/Drive/Gmail, et configurer un écran de consentement OAuth. C'est l'étape la plus longue — comptez 20-30 minutes la première fois.

### 5. Remplacer les champs placeholder

Dans le JSON importé, remplacez :
- `REMPLACER_PAR_VOTRE_CLE_RAPIDAPI` → votre clé RapidAPI
- `REMPLACER_PAR_URL_DE_VOTRE_GOOGLE_SHEET` (3 occurrences) → l'URL de votre Sheet
- `REMPLACER_PAR_VOTRE_EMAIL` → votre adresse courriel

### 6. Publier le workflow

Une fois testé manuellement (bouton *Execute workflow*), activez-le pour qu'il tourne sur l'horaire défini dans le Schedule Trigger.

## Personnalisation

Tout se règle dans deux nœuds Code, sans toucher au reste du workflow.

**Nœud "Liste Queries"** — la liste des recherches à effectuer chaque jour :
```python
queries = [
    "votre poste cible Votre ville",
    "autre intitulé Votre région",
]
```

**Nœud "Scoring"** — les critères de pertinence :
- `MOTS_CLES_TITRE_SENIOR` / `MOTS_CLES_FORTS` : ce qui fait monter le score
- `MOTS_EXCLUSION` : mots qui rejettent une offre d'office (ex. "junior" si vous cherchez un poste senior)
- `DOMAINES_EXCLUS` : sites à exclure (ex. plateformes à candidature payante)
- `VILLES_CIBLES` et `SALAIRE_MIN` : vos critères géographiques et salariaux

## Limites connues

- **Quota gratuit JSearch** : 200 requêtes/mois. Avec 5 requêtes/jour, ça laisse peu de marge pour des tests manuels en plus de l'exécution planifiée — réduisez la liste de requêtes ou passez à un plan payant si vous testez beaucoup.
- **Jeton Google en mode "Test"** : si votre app OAuth reste en mode Testing dans Google Cloud Console, l'accès expire tous les 7 jours et il faut reconnecter la credential manuellement. Passez le statut de publication à **"In production"** dans Google Auth Platform → Audience pour éviter ça — pas besoin de compléter la vérification complète de Google pour un usage personnel.
- **Le champ salaire est souvent vide** : la plupart des offres canadiennes retournées par l'API n'indiquent pas de salaire minimum. Le critère salaire dans le scoring se déclenche rarement — ne comptez pas dessus comme filtre principal.

## Dépannage rapide

| Symptôme | Cause probable |
|---|---|
| `Python runner unavailable` | Task runner externe non configuré (self-hosted uniquement) |
| `name '_input' is not defined` | Code écrit pour l'ancienne syntaxe Pyodide — utilisez `_items` et `item["json"]` |
| La boucle s'arrête après 1-2 requêtes sans erreur | Une connexion referme la boucle via un nœud qui dépend de la sortie "done" — vérifiez que rien ne reboucle vers *Loop Over Items* sauf le nœud Scoring |
| Offres dupliquées malgré le dédoublonnage | L'identifiant utilisé pour comparer n'est pas stable — ce workflow utilise l'URL de l'offre comme clé, pas l'ID retourné par l'API |
| `redirect_uri_mismatch` sur Google | L'URI de redirection affiché par n8n doit être copié-collé (jamais retapé) dans Google Cloud Console |

## Licence

Faites-en ce que vous voulez. Si vous l'améliorez, un retour est apprécié mais pas requis.
