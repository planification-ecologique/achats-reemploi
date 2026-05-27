# POC Grist — Cartographie des acteurs du réemploi

Proof of concept pour la **mesure 6 du SPASER de l'État** (cible 6.2.7) : permettre aux acheteurs publics d'identifier les acteurs du réemploi en mesure de répondre à leurs marchés (art. 58 loi AGEC).

Ce dépôt contient les deux livrables du POC :

| Fichier | Rôle |
|---|---|
| `acteurs_reemploi_poc.csv` | Jeu de données (589 structures) prêt à importer dans Grist |
| `widget_carto.html` | Widget carte custom Grist (Leaflet) avec filtres, liste et fiche structure |

## Contenu du jeu de données (589 structures, 2 domaines)

Le CSV regroupe deux jeux distingués par la colonne **`Domaine`** :

| Domaine | Lignes | Origine | Périmètre |
|---|---|---|---|
| `Mobilier & informatique` | 86 | Réseaux du réemploi (Envie, Ecodair, Valdelia/ressourceries, Adopte un Bureau, Tricycle, RNRR…) | France, art. 58 AGEC |
| `Réemploi d'emballages` | 503 | Cartographie acteurs réemploi emballages B2B | International (16 pays) |

## ⚠️ Nature des données

Jeu **illustratif** pour le POC, à fiabiliser avant tout usage officiel :

- **SIRET** : synthétiques (format Luhn valide mais fictifs) pour les 86 lignes mobilier+info ; **vides** pour les 503 lignes emballages. À compléter via SIRENE.
- **Coordonnées** : géocodées automatiquement (geonamescache + table d'appoint). 307 lignes au niveau ville, 150 via table d'appoint (exonymes, communes, régions), 46 repliées sur le **centroïde du pays** (petites communes non résolues, à affiner). À vérifier sur la Base Adresse Nationale.
- **Données source emballages** conservées telles quelles : certaines structures taguées « France » sont en réalité des filiales nord-américaines (`… (Canada)`/`(USA)`) — géocodées à leur ville réelle mais le champ `Pays` reflète la source.
- Le fichier source réel du POC mobilier reste celui transmis par **SPARE**.

## Schéma des données

Colonnes alignées sur le §7.1 du PRD, plus deux colonnes géo pour la carte :

| Colonne | Type Grist conseillé | Valeurs |
|---|---|---|
| `Nom_structure` | Texte | — |
| `Statut_structure` | Choix | Secteur de l'insertion / de l'ESS / du handicap / Entreprise conventionnelle |
| `SIRET` | Texte | 14 chiffres |
| `Categorie_produits` | Liste de choix | catégories art. 58 (séparées par `;`) |
| `Activite_structure` | Liste de choix | Réparation / Reconditionnement / Nettoyage ou Lavage / Upcycling / Point de collecte / Distribution (`;`) |
| `Position_chaine` | Choix | Je collecte / Je répare / Je vends / Je collecte et je vends |
| `Adresse`, `Ville` | Texte | — |
| `Departement` | Choix | `NN - Nom` |
| `Zone_intervention` | Choix | Nationale / Régionale - … / Départementale - … |
| `Site_internet` | Texte | URL |
| `Presentation` | Texte | présentation succincte |
| `Certifications` | Liste de choix | labels (`;`) |
| `Latitude`, `Longitude` | Numérique | WGS84 |

Colonnes ajoutées par la fusion (renseignées surtout pour le domaine emballages) :

| Colonne | Type Grist conseillé | Valeurs |
|---|---|---|
| `Domaine` | Choix | Mobilier & informatique / Réemploi d'emballages |
| `Pays` | Choix | France, Belgique, Allemagne… (16 pays) |
| `Region_source` | Texte | région d'origine de la source |
| `Type_acteur` | Choix | Opérateur / Fabricant / Pooler / Lavage / Reconditionneur… |
| `Secteur` | Choix | Logistique / Boissons / Restauration / Industrie… |
| `Offre`, `Type_emballage`, `Materiaux`, `Cible_client` | Texte/Choix | dimensions de la source emballages |

> Les champs multi-valeurs utilisent `;` comme séparateur. À l'import, typez ces colonnes en **Liste de choix** dans Grist (Grist découpe automatiquement). Les colonnes vides pour un domaine donné (ex. `Departement` côté emballages) sont normales.

## Import dans Grist

1. Créer un document → **Add New → Import from file** → `acteurs_reemploi_poc.csv`.
2. Vérifier le typage des colonnes (Choix / Liste de choix / Numérique comme ci-dessus).
3. Renommer la table `Acteurs` si besoin (les colId restent ceux des en-têtes).

## Ajouter le widget carte

Le widget lit la table active via l'API Grist (`grist.ready` + `grist.onRecords`).

1. Héberger `widget_carto.html` à une URL publique (ex. GitHub Pages de ce dépôt, ou un hébergement souverain).
2. Dans la vue Grist : **Add New → Add Widget to Page → Custom**.
3. Coller l'URL du widget, choisir la table `Acteurs`, accès **Read table**.
4. La carte affiche les structures ; les 7 filtres et les filtres rapides agissent côté widget.

### Prévisualisation locale (sans Grist)

Ouvert hors Grist, le widget retombe automatiquement sur le CSV servi à côté de lui :

```bash
python3 -m http.server 8000   # puis http://localhost:8000/widget_carto.html
```

## Fonctionnalités du widget (couverture PRD §7)

- **Carte** (fond IGN / Géoplateforme, souverain) avec clustering des marqueurs ; recentrage automatique sur les résultats filtrés (zoom mondial inclus).
- **Liste des acteurs** dans le panneau latéral, synchronisée avec les filtres ; un clic centre la carte sur la structure et ouvre sa fiche.
- **Filtres** : Nom, Domaine, Pays, Catégorie de produits, Position dans la chaîne, Activité, Statut, Zone d'intervention, Département.
- **Filtres rapides** (chips) sur Domaine, Position et Statut.
- **Fiche structure** au clic sur un marqueur ou un élément de la liste — affiche aussi, pour le domaine emballages, le type d'acteur, le secteur, la cible client et les matériaux.
- Compteur de résultats + réinitialisation.

## Vues natives Grist (complément, sans code)

En plus du widget, on peut configurer côté Grist : une vue **Carte** native (colonnes Latitude/Longitude), une vue **Fiche** (Card) pour la présentation détaillée, et des **filtres de vue** sur les 7 champs — utile comme repli si l'hébergement du widget custom n'est pas possible dans un environnement ministériel restreint.
