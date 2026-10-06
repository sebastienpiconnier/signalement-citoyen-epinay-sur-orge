# Page de signalement citoyen : Épinay-sur-Orge

Formulaire web permettant aux habitants d'Épinay-sur-Orge (Essonne) de signaler un problème sur l'espace public (voirie, éclairage, propreté, espaces verts…), avec géolocalisation sur carte et photos. Chaque signalement est transmis par e-mail au service compétent grâce à un workflow **n8n**, et le déclarant reçoit un accusé de réception.

> Les adresses de webhook, identifiants de messagerie et adresses de routage de ce dépôt sont **anonymisés**. Voir [Configuration](#configuration).

## Fonctionnement

```
index.html ──POST JSON──▶ webhook n8n ──▶ validation et routage ──▶ API Gmail ──▶ service compétent
                                                               └──▶ accusé de réception au déclarant
                                                               └──▶ alerte administrateur en cas d'échec
```

Aucune base de données : l'e-mail envoyé au service est le seul enregistrement du signalement.

## Fonctionnalités

### Page (`index.html`)

- **Parcours guidé** : catégorie, puis motif, localisation, description, photos et coordonnées.
- **9 catégories** avec motifs détaillés et renvois automatiques vers le bon interlocuteur quand la commune n'est pas compétente (fibre optique, GRDF, Enedis, conflits de voisinage, urbanisme…).
- **Localisation** : géolocalisation de l'appareil, clic sur la carte (Leaflet, fonds IGN et OpenStreetMap) ou saisie d'adresse avec autocomplétion via le service de géocodage de l'IGN (Géoplateforme). Recherche limitée au territoire de la commune.
- **Photos** : jusqu'à 5, redimensionnées et compressées dans le navigateur avant l'envoi.
- **Anti-robot** : champ piège invisible et délai minimal avant l'envoi.
- **Repli par e-mail** si l'envoi échoue.
- Page autonome : HTML, CSS et JavaScript natifs, sans framework ni étape de build.

### Workflow n8n (`workflow-n8n.json`)

| Nœud | Rôle |
|---|---|
| Réception du signalement | Webhook POST |
| Table de routage | **Seul nœud à configurer** : mode maquette/production, adresses par catégorie |
| Validation et routage | Nettoyage des champs, contrôles, référence `ESO-AAMMJJ-HHMMSS`, choix du destinataire |
| Construction du message | Assemblage MIME manuel avec un nombre variable de photos jointes |
| Envoi au service (API Gmail) | Envoi via `gmail/v1/users/me/messages/send` |
| Accusé de réception (Gmail) | Confirmation au déclarant avec sa référence |
| Alerte administrateur (Gmail) | Prévient l'administrateur si l'envoi au service échoue |
| Réponses | Retour JSON à la page (succès, rejet, échec) |

**Mode maquette** : tous les e-mails (service, accusé de réception, alerte) sont redirigés vers une boîte de test, avec un préfixe `[MAQUETTE]` et l'indication du destinataire réel.

## Configuration

### 1. Workflow n8n

1. Importer `workflow-n8n.json` dans n8n.
2. Associer les nœuds Gmail et HTTP Request à votre identifiant **Gmail OAuth2** (remplace `VOTRE_ID_CREDENTIAL`).
3. Dans le nœud *Réception du signalement*, définir le chemin du webhook (remplace `votre-chemin-webhook`).
4. Dans le nœud *Table de routage*, remplacer les adresses `@example.org` par les vraies boîtes (test, archivage, administrateur, services), puis passer `mode` à `production` une fois les tests terminés.
5. Avant la mise en ligne publique, restreindre `allowedOrigins` (actuellement `*`) au domaine du site.

### 2. Page

Dans le bloc `CONFIG` du script de `index.html` :

| Clé | À renseigner |
|---|---|
| `WEBHOOK` | URL de production du webhook n8n (remplace `https://VOTRE-INSTANCE-N8N.example/...`) |
| `URL_FIBRE` | Lien vers le formulaire de la Cellule Fibre Optique (remplace `https://VOTRE-FORMULAIRE-FIBRE.example`) |
| `LOGO` | Chemin du logo blanc (`assets/logo-epinay-negatif.webp`, non fourni : un logo vectoriel intégré prend le relais) |
| `PAYLOAD_MAX_MO` | À aligner sur la taille maximale de requête de votre instance n8n |

Les adresses de contact publiques de la mairie affichées dans la page et dans l'accusé de réception sont conservées. Plusieurs mentions `[à confirmer]` restent à valider avec les services (délais de traitement, permanences, adresses de service).

## Structure

```
.
├── index.html             # Page de signalement complète
├── workflow-n8n.json      # Workflow n8n anonymisé
├── favicon.ico, favicon-*.png, apple-icon-*.png, android-icon-192x192.png, ms-icon-*.png
└── README.md
```

## Lancer en local

```bash
python3 -m http.server 8000
# puis http://localhost:8000
```

Sans webhook configuré, l'envoi échoue et la page propose le repli par e-mail.

## Données personnelles

Le formulaire collecte un e-mail (obligatoire), un nom et un téléphone (facultatifs), une position et des photos, avec consentement explicite. Rien n'est stocké par le workflow : les données ne transitent que par les e-mails envoyés. Penser à compléter la page de politique de confidentialité (`URL_CONFIDENTIALITE`).

## Crédits

Conception et développement : Sébastien Piconnier, conseiller municipal délégué à l'Innovation et aux Systèmes d'information.
