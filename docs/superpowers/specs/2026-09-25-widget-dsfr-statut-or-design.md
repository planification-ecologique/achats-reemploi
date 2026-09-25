# Widget carto — DSFR sidebar, filtre Statut OR, séparation header

Date: 2026-09-25  
Fichier ciblé: `widget_carto.html`  
Statut: approved (design)

## Problème

1. Le filtre **Statut** liste les chaînes brutes Grist (combinaisons séparées par `;`), avec un match exact. L’acheteur ne peut pas filtrer sur un statut atomique sans connaître la combinaison stockée.
2. En haut de la sidebar, le CTA « Se référencer » (acteurs) est mélangé au bloc destiné aux acheteurs publics.
3. L’UI sidebar n’utilise pas le **DSFR**.

## Décisions produit

| Sujet | Choix |
|---|---|
| Filtre Statut | Multi-select (checkboxes), logique **OU** |
| Périmètre DSFR | **Sidebar seule** ; carte Leaflet + fiche détail hors DSFR |
| Header | **Deux blocs empilés** (acheteurs puis acteurs) |
| Implémentation DSFR | CDN `@gouvfr/dsfr` (CSS + JS), classes `fr-*` |

## Design

### 1. Filtre Statut (OR)

- Normaliser `Statut_structure` comme champ multi : split sur `;` → tableau d’atomes (`statutAtoms`).
- Options UI = union des atomes présents dans les données (typiquement 4) :
  - Entreprise conventionnelle
  - Secteur de l'ESS
  - Secteur de l'insertion
  - Secteur du handicap
- UI : `fr-fieldset` + `fr-checkbox` (une case par atome). Pas de `<select>` pour ce champ.
- Match :
  - 0 case cochée → aucun filtre statut
  - ≥1 case cochée → structure retenue si **au moins un** de ses atomes est dans la sélection (OR)
- Liste / détail : afficher les atomes séparés (tags), plus la string concaténée.
- Reset : décoche toutes les cases statut.

### 2. Header — séparation acheteurs / acteurs

Deux blocs empilés, séparés visuellement (bordure ou fond `fr-background-alt--grey`) :

1. **Bloc acheteurs**
   - Logo + `h1` + bouton aide
   - Sous-titre : « Pour les acheteurs publics — SPASER / art. 58 loi AGEC »
2. **Bloc acteurs**
   - Titre court : « Vous êtes un acteur du réemploi ? »
   - CTA `fr-btn` (secondaire) → même `QUESTIONNAIRE_URL` Grist

Le panneau d’aide (`#help`) reste sous ces deux blocs ; comportement inchangé.

### 3. DSFR sidebar

- Charger le CDN DSFR (CSS + JS ; icons si nécessaire pour les composants utilisés).
- Remapper le markup sidebar :
  - Inputs / selects : `fr-input`, `fr-select`
  - Statut : fieldset checkboxes
  - Sections filtres / liste : `fr-accordion` ou `details` stylés DSFR
  - Lignes liste : `fr-card` compact ou équivalent cliquable accessible
  - Barre bas : boutons tertiaires Export / Reset
- Layout flex sidebar (~400px) + carte inchangé.
- **Hors scope DSFR** : `#map`, clusters Leaflet, `#detail` (garde le style actuel).
- **Hors scope fonctionnel** : autres filtres (cat, act, etc.) restent selects mono-valeur ; basemap IGN ; boot Grist/CSV.

## Contraintes

- Un seul fichier modifié pour l’implémentation : `widget_carto.html`.
- Widget embarqué Grist (iframe) : CDN externe acceptable ; pas de bundler npm.
- Accessibilité : focus visible DSFR, labels liés, `aria` checkboxes / liste / aide conservés ou améliorés.
- Données : atomes dérivés dynamiquement des records (pas de liste hardcodée figée — ordre alphabétique FR recommandée).

## Critères d’acceptation

1. Dropdown Statut ne montre plus de combinaisons `;`.
2. Cocher « Secteur de l'ESS » affiche aussi les structures « ESS;insertion » et « ESS;handicap ».
3. Cocher ESS + insertion = union des deux ensembles.
4. Header : bloc acheteurs distinct du bandeau CTA acteurs.
5. Sidebar reconnaissable DSFR (Marianne/tokens/`fr-*`) ; carte inchangée.
6. Export / reset / Grist / CSV / carte continuent de fonctionner.

## Hors scope

- Migration DSFR de la fiche détail carte.
- Multi-select sur catégorie / activité / autres champs.
- Changement du questionnaire de référencement ou du schéma Grist.
