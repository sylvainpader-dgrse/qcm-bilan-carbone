# Design — QCM Bilan Carbone® Enrichi

**Date :** 2026-03-31
**Fichier cible :** `QCM CARBONE.html` (fichier HTML unique, vanilla JS)
**Approche :** Enhancement progressive — aucune dépendance externe, fichier partageable en un clic

---

## 1. Sources de données

Toutes les questions (existantes et nouvelles) sont issues exclusivement de :
- **QCM existant** : 67 questions MCQ + 4 questions d'ordre (vérifiées et validées)
- **Guide méthodologique BC® v9** (DOCX, 295 000 caractères) : contexte climatique, standards comparatifs, incertitude v9, BEGES-R, parcours de transition, CSRD

Règle stricte : aucun contenu inventé ou extrapolé hors de ces deux sources.

---

## 2. Types de questions

| Code | Nom | Description |
|------|-----|-------------|
| `mcq` | QCM 1 réponse | 4 options, 1 bonne réponse (existant) |
| `ord` | Ordre | Drag-and-drop vertical pour remettre en ordre (existant) |
| `tf` | Vrai / Faux | 2 boutons, 1 bonne réponse |
| `multi` | Multi-réponses | Cases à cocher, toutes les bonnes cases doivent être cochées |
| `match` | Association | 2 colonnes, cliquer gauche puis droite pour relier par paires |
| `blank` | Texte à trous | `"Le BC® est né en [____]"` — champ texte, validation insensible à la casse, synonymes acceptés |

### Comportement feedback commun à tous les types
- **Si faux (mode Étude)** : explication affichée automatiquement, bandeau rouge
- **Si juste (mode Étude)** : bandeau vert, bouton discret "Voir l'explication" disponible
- **Mode Examen** : aucune explication ni feedback pendant le quiz — tout révélé à la fin

---

## 3. Nouvelles questions (≈40)

Thèmes non encore couverts dans le QCM existant, tirés du DOCX :

### 3.1 Contexte climatique & objectifs cadres
- Accord de Paris : objectif principal (+2°C / efforts +1.5°C)
- GIEC AR6 : nombre d'États membres (195), conclusions clés
- SNBC : objectif neutralité carbone France (2050), division par 6 des émissions vs 1990
- Green Deal européen : objectif -55% d'ici 2030 vs 1990
- Réduction nécessaire à l'échelle mondiale pour limiter à +1.5° (84%)

### 3.2 Comptabilité carbone — concepts fondamentaux
- Différence approche inventaire (cadastrale) vs approche empreinte (responsabilité)
- 4 échelles de comptabilité carbone : territoire, individu, produit, organisation
- Atténuation vs adaptation : définitions et différences
- Rôle de la comptabilité carbone dans la triple comptabilité

### 3.3 Standards comparatifs
- GHG Protocol : ne demande PAS d'amortir les immobilisations (≠ BC® et BEGES-R)
- ISO 14064-1 : intègre les "suppressions de GES" (≠ GHG-P)
- GHG Protocol : seul standard demandant les 2 approches (location-based ET market-based)
- GHG Protocol : n'intègre pas les "déplacements de visiteurs" (≠ BC®)
- BC® vs autres standards : seul standard intégrant mobilisation + plan de transition comme exigences

### 3.4 BEGES-R
- Organisations soumises au BEGES-R (>500 salariés en France, collectivités >50 000 hab.)
- Précaution périmètre temporel BC® pour répondre au BEGES-R (année complète)
- Périmètre organisationnel pour BEGES-R (personne morale = SIREN)

### 3.5 Incertitude v9
- Double approche v9 : estimation qualitative ET quantitative (nouveauté vs v8)
- Limites mathématiques v8 résolues en v9 (alignement avec meilleures pratiques des BDD)

### 3.6 Parcours de transition
- 4 étapes du parcours : Analyser → Construire → Mettre en œuvre → Évaluer
- Le parcours n'est pas linéaire — logique de cycle de progression
- Deux approches complémentaires : atténuation (org → climat) et adaptation (climat → org)

---

## 4. Modes de jeu

Sélectionnables dans l'écran de filtre (avant démarrage) :

### Mode Étude (défaut)
- Feedback immédiat après chaque réponse
- Explication auto si faux, bouton discret si juste
- Timer visible mais non bloquant

### Mode Examen
- Aucun feedback pendant le quiz
- Score et corrections révélés uniquement à l'écran de résultats
- Timer visible

### Timer
- Chrono global configurable (défaut : 45 min pour l'ensemble des questions sélectionnées)
- Affiché en haut du quiz, s'arrête quand on arrive aux résultats
- Non bloquant : ne coupe pas le quiz si temps écoulé, affiche simplement "Temps dépassé"

---

## 5. Persistance (localStorage)

Clé : `qcm_bc_v2`

Structure sauvegardée :
```json
{
  "starred": ["id1", "id2"],         // questions marquées ⭐
  "attempts": { "id1": 3, "id2": 1 }, // nb de fois vues
  "successes": { "id1": 1 },          // nb de fois réussies
  "lastSession": { "score": 42, "total": 67, "date": "2026-03-31" }
}
```

### Filtre "À revoir" dans l'écran de démarrage
- Affiche uniquement les questions ⭐ marquées
- Si aucune question marquée : bouton grisé avec tooltip "Marque des questions pour les revoir"

### Mémorisation espacée
- Bouton ⭐ disponible sur chaque question (après réponse) et en mode review
- Dans l'écran de résultats : section "Questions à revoir" listant les questions ratées
- Compteur de passages affiché sur le bouton ⭐ (`⭐ 3`)

---

## 6. Architecture technique

### Structures de données nouvelles (dans le JS)

```js
// Vrai/Faux
{ id: "TF1", type: "tf", cat: "Historique", q: "...", a: true, e: "..." }

// Multi-réponses
{ id: "M1", type: "multi", cat: "...", q: "...", o: ["A","B","C","D"], a: [0,2], e: "..." }
// a = indices des bonnes réponses (tableau)

// Association
{ id: "MA1", type: "match", cat: "...", q: "...",
  pairs: [ {l: "Initial", r: "4 ans max"}, {l: "Standard", r: "2 ans"}, {l: "Avancé", r: "1 an"} ],
  e: "..." }

// Texte à trous
{ id: "B1", type: "blank", cat: "...", q: "La philosophie du BC® est « [____] pour agir ».",
  a: ["compter"], synonyms: [], e: "..." }
```

### Nouvelles fonctions d'état
- `confirmMulti()` : vérifie que tous les indices cochés = tableau `a` exactement
- `submitMatch(qid)` : vérifie toutes les paires reliées
- `submitBlank(qid, value)` : normalise et compare à `a` + `synonyms`
- `toggleStar(id)` : bascule ⭐ + persiste dans localStorage
- `loadLS() / saveLS()` : lecture/écriture localStorage

### Modifications à buildQuiz()
- Switch sur `q.type` pour dispatcher vers le bon renderer
- `buildTF(q)`, `buildMulti(q)`, `buildMatch(q)`, `buildBlank(q)`

### Modifications à buildFilter()
- Ajout sélecteur Mode (Étude / Examen)
- Ajout toggle Timer (on/off + durée)
- Ajout bouton "À revoir (⭐ N)" si N > 0

### Modifications à buildResults()
- Section "Questions à revoir" (questions ratées + suggestions à ⭐)
- Historique dernière session

---

## 7. Ce qui ne change pas

- Design visuel (couleurs, typographie, dark theme, layout card)
- Filtres par catégorie
- Mode review post-résultats
- Dépôt OCCF anonymisé (hors scope)
- Format fichier unique HTML

---

## 8. Critères de succès

- Toutes les nouvelles questions sont factuellement correctes (vérifiables dans le DOCX)
- Les 5 types de questions fonctionnent sur mobile (touch) et desktop (drag/click)
- Le localStorage persiste entre sessions sans erreur
- Le fichier final reste un HTML unique partageable
- Mode Examen : aucune fuite d'information pendant le quiz
