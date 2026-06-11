# POC Grist — Les acteurs du réemploi pour une commande publique circulaire

Proof of concept pour la **mesure 6 du SPASER de l'État** (cible 6.2.7) : permettre aux acheteurs publics d'identifier les acteurs du réemploi en mesure de répondre à leurs marchés (art. 58 loi AGEC).

Ce dépôt contient les deux livrables du POC :

| Fichier | Rôle |
|---|---|
| `acteurs_reemploi_poc.csv` | Jeu de données (39 structures réelles SPARE, France) prêt à importer dans Grist |
| `widget_carto.html` | Widget carte custom Grist (Leaflet) avec filtres, liste et fiche structure |

## Contenu du jeu de données (39 structures)

Le jeu de données est **circonscrit à la France** (métropole + DROM-COM) et aux **produits visés par l'article 58 de la loi AGEC**. Il ne contient que des **données réelles vérifiées** :

| Source | Lignes | Nature |
|---|---|---|
| `SPARE — vérifié 12/03/2026` | 39 | **Données réelles** (SIRET, adresses), matériel informatique / EEE ciblés achats publics |

> La colonne **`Source`** est conservée pour la traçabilité interne (mais n'est plus exposée comme filtre dans le widget).

### Évolution depuis la V1 (retours DAE)

- **Domaine « Réemploi d'emballages » retiré** (503 structures, internationales, hors périmètre art. 58). Il pourra être réintroduit *dans un second temps* pour élargir à des produits non visés par AGEC.
- **Périmètre France uniquement** (DROM-COM inclus), cohérent avec la démarche du réemploi.
- **Lignes de démo (champs synthétiques) retirées** : seules les 39 structures réelles vérifiées SPARE sont conservées.
- **Colonnes hors PRD supprimées** (issues de la carto emballages) : `Pays`, `Region_source`, `Type_acteur`, `Secteur`, `Offre`, `Type_emballage`, `Materiaux`, `Cible_client`.
- **`Domaine` recalculé en domaine d'activité multi-tag** (`Informatique`, `Mobilier`, `Électroménager`) dérivé des catégories de produits ; une structure sur plusieurs domaines porte plusieurs tags.
- **`Source` conservée pour la traçabilité interne** mais **n'est plus exposée comme filtre** dans le widget.

## ⚠️ Nature des données

Jeu **illustratif** pour le POC, à fiabiliser avant tout usage officiel :

- **SIRET** : **réels** pour les 39 lignes SPARE. À recouper avec SIRENE avant usage officiel.
- **Coordonnées** : géocodées automatiquement (geonamescache + table d'appoint). À vérifier sur la Base Adresse Nationale.
- Le fichier source réel du POC mobilier reste celui transmis par **SPARE**.

## Schéma des données

Colonnes alignées sur le §7.1 du PRD :

| Colonne | Type Grist conseillé | Valeurs |
|---|---|---|
| `Nom_structure` | Texte | — |
| `Statut_structure` | Choix | Secteur de l'insertion / de l'ESS / du handicap / Entreprise conventionnelle |
| `Domaine` | Liste de choix | Informatique / Mobilier / Électroménager (multi-tag, séparés par `;`) |
| `SIRET` | Texte | 14 chiffres |
| `Categorie_produits` | Liste de choix | catégories art. 58 (séparées par `;`) |
| `Activite_structure` | Liste de choix | Réparation / Reconditionnement / Nettoyage ou Lavage / Upcycling / Point de collecte / Distribution (`;`) |
| `Position_chaine` | Choix | Je collecte / Je répare / Je vends / Je collecte et je vends |
| `Adresse`, `Ville` | Texte | — |
| `Departement` | Choix | `NN - Nom` |
| `Zone_intervention` | Choix | Nationale / Régionale - … / Départementale - … |
| `Site_internet` | Texte | URL |
| `Presentation` | Texte | présentation succincte |
| `Certifications` | Liste de choix | reconnaissances : labels, certifications, distinctions (`;`) |
| `Latitude`, `Longitude` | Numérique | WGS84 |
| `Source` | Choix | SPARE (vérifié) / Démo (à vérifier) — *traçabilité interne, non affichée comme filtre* |

> Les champs multi-valeurs utilisent `;` comme séparateur. À l'import, typez ces colonnes en **Liste de choix** dans Grist (Grist découpe automatiquement).

## Import dans Grist

1. Créer un document → **Add New → Import from file** → `acteurs_reemploi_poc.csv`.
2. Vérifier le typage des colonnes (Choix / Liste de choix / Numérique comme ci-dessus).
3. Renommer la table `Acteurs` si besoin (les colId restent ceux des en-têtes).

## Ajouter le widget carte

Le widget lit la table active via l'API Grist (`grist.ready` + `grist.onRecords`).

1. Héberger `widget_carto.html` à une URL publique (ex. GitHub Pages de ce dépôt, ou un hébergement souverain).
2. Dans la vue Grist : **Add New → Add Widget to Page → Custom**.
3. **Sélectionner les données** : dans le panneau de droite, régler « SELECT DATA » sur la table `Acteurs` (sinon le widget ne reçoit aucune ligne → liste à 0).
4. Coller l'URL du widget dans « Custom URL », puis **autoriser l'accès « Read table »** quand Grist le demande.
5. La carte affiche les structures ; filtres, liste et fiche agissent côté widget.

> **Liste à 0 alors que la table est remplie ?** C'est presque toujours (a) la source de données du widget non réglée sur la table, ou (b) l'accès « Read table » non accordé. Le widget affiche un message de diagnostic explicite dans ces cas. La lecture des colonnes est tolérante (casse/accents) et gère les listes de choix Grist.

### Prévisualisation locale (sans Grist)

Ouvert hors Grist, le widget retombe automatiquement sur le CSV servi à côté de lui :

```bash
python3 -m http.server 8000   # puis http://localhost:8000/widget_carto.html
```

## Fonctionnalités du widget (couverture PRD §7)

- **En-tête** : titre « Les acteurs du réemploi pour une commande publique circulaire » et emplacement de **logo** (SVG provisoire à remplacer par le logo officiel).
- **Bouton de référencement** « Vous êtes un acteur du réemploi ? Se référencer » : renvoie vers le questionnaire de présentation des structures (URL à renseigner via `QUESTIONNAIRE_URL` dans le widget).
- **Carte** : fond **Plan IGN v2** (Géoplateforme) — raster, en français, souverain. Clustering des marqueurs et recentrage sur le **barycentre** des résultats.
- **Liste des acteurs** dans le panneau latéral, synchronisée avec les filtres ; un clic centre la carte sur la structure et ouvre sa fiche.
- **Filtres** : Nom, Domaine, Catégorie de produits, Position dans la chaîne du réemploi, Activité, Statut, Zone d'intervention, Département. *(Les filtres « Source » et « Pays » ont été retirés pour alléger l'interface.)*
- **Filtres rapides** : réduits à un seul rang de raccourcis (chips) sur le **Domaine**, en complément des filtres ci-dessus.
- **Fiche structure** au clic sur un marqueur ou un élément de la liste : présentation, puis **site internet juste après la description**, domaine(s), SIRET, catégories, activités, position dans la chaîne du réemploi, ville, département, zone d'intervention et **« Reconnaissances (labels, certifications, distinction) »**.
- Compteur de résultats + réinitialisation.

## Vues natives Grist (complément, sans code)

En plus du widget, on peut configurer côté Grist : une vue **Carte** native (colonnes Latitude/Longitude), une vue **Fiche** (Card) pour la présentation détaillée, et des **filtres de vue** sur les champs ci-dessus — utile comme repli si l'hébergement du widget custom n'est pas possible dans un environnement ministériel restreint.

## Points ouverts à confirmer avec la DAE

- **Questionnaire de référencement** : URL à fournir pour activer le bouton « Se référencer ».
- **Logo** : fournir l'asset officiel pour remplacer le SVG provisoire.
