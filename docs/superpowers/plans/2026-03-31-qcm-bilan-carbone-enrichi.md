# QCM Bilan Carbone® Enrichi — Plan d'implémentation

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enrichir le fichier HTML unique du QCM Bilan Carbone® avec 4 nouveaux types de questions, ~40 nouvelles questions issues du DOCX, un système de persistance localStorage avec mémorisation espacée, et des modes Étude/Examen avec timer.

**Architecture:** Enhancement progressive d'un fichier HTML vanilla JS unique. Nouvelles questions dans des tableaux séparés (`TF`, `MULTI`, `MATCH`, `BLANK`) fusionnés dans `start()`. Nouveaux renderers dans des fonctions `buildTF/Multi/Match/Blank()`. State enrichi pour les nouveaux types et le localStorage.

**Tech Stack:** HTML5, CSS3, vanilla JS (ES6), localStorage API — aucune dépendance externe.

**Fichier source de référence :** `C:\Users\sylva\OneDrive\Bureau\QCM CARBONE.docx` (guide méthodologique BC® v9)
**Fichier existant :** code HTML fourni dans la conversation (67 MCQ + 4 ORD)
**Fichier de sortie :** `C:\Users\sylva\OneDrive\Bureau\q\QCM-CARBONE-v2.html`

---

## Structure des fichiers

| Fichier | Action | Responsabilité |
|---------|--------|----------------|
| `QCM-CARBONE-v2.html` | Créer | Fichier HTML unique complet — tout le CSS, les données et le JS |

---

## Task 1 : Fichier de base + système de types

**Fichier :** `C:\Users\sylva\OneDrive\Bureau\q\QCM-CARBONE-v2.html`

- [ ] **Étape 1 : Copier le HTML existant comme base**

  Créer `QCM-CARBONE-v2.html` avec le contenu complet du HTML existant (67 MCQ + 4 ORD), puis ajouter le champ `type:"mcq"` à toutes les questions du tableau `Q` et `type:"ord"` à toutes les questions de `ORD`.

  Dans le tableau `Q`, chaque objet devient :
  ```js
  {id:1, type:"mcq", cat:"Historique", q:"...", o:[...], a:1, e:"..."},
  ```

  Dans le tableau `ORD`, chaque objet devient :
  ```js
  {id:"O1", type:"ord", cat:"...", q:"...", items:[...], correct:[...], e:"..."},
  ```

- [ ] **Étape 2 : Ajouter les tableaux de données vides pour les nouveaux types**

  Après la déclaration de `ORD`, ajouter :
  ```js
  const TF = [];    // Vrai/Faux — à remplir en Task 2
  const MULTI = []; // Multi-réponses — à remplir en Task 3
  const MATCH = []; // Associations — à remplir en Task 4
  const BLANK = []; // Texte à trous — à remplir en Task 5
  ```

- [ ] **Étape 3 : Enrichir le state initial**

  Remplacer la déclaration de `state` par :
  ```js
  let state = {
    mode: "menu",      // menu | filter | quiz | results | review
    quizMode: "study", // study | exam
    timerActive: false,
    timerDuration: 45 * 60, // secondes
    timerElapsed: 0,
    timerInterval: null,
    selCats: new Set(CATS),
    inclOrd: true,
    inclTF: true,
    inclMulti: true,
    inclMatch: true,
    inclBlank: true,
    filterStarred: false,
    qs: [], cur: 0, sel: null, conf: false, score: 0, ans: [], showExp: false,
    revIdx: 0,
    dragItems: {}, dragIdx: null, dragDone: {}, dragOk: {},
    matchSel: {},    // { qid: { leftIdx: null, pairs: {leftIdx: rightIdx} } }
    matchDone: {},   // { qid: bool }
    matchOk: {},     // { qid: bool }
    blankVal: {},    // { qid: string }
    blankDone: {},   // { qid: bool }
    blankOk: {},     // { qid: bool }
    ls: loadLS(),    // données persistantes
  };
  ```

- [ ] **Étape 4 : Ajouter les fonctions localStorage**

  Avant la déclaration de `state`, ajouter :
  ```js
  const LS_KEY = "qcm_bc_v2";

  function loadLS() {
    try {
      const raw = localStorage.getItem(LS_KEY);
      return raw ? JSON.parse(raw) : { starred: [], attempts: {}, successes: {}, lastSession: null };
    } catch { return { starred: [], attempts: {}, successes: {}, lastSession: null }; }
  }

  function saveLS(ls) {
    try { localStorage.setItem(LS_KEY, JSON.stringify(ls)); } catch {}
  }

  function toggleStar(id) {
    const ls = state.ls;
    const idx = ls.starred.indexOf(id);
    if (idx === -1) ls.starred.push(id);
    else ls.starred.splice(idx, 1);
    saveLS(ls);
    setState({ ls });
  }

  function isStarred(id) { return state.ls.starred.includes(String(id)); }
  ```

- [ ] **Étape 5 : Mettre à jour la fonction `start()`**

  Remplacer la fonction `start()` existante par :
  ```js
  function start() {
    const cats = state.selCats;
    const starred = state.filterStarred ? new Set(state.ls.starred.map(String)) : null;

    const filterQ = (arr) => {
      let res = arr.filter(q => cats.has(q.cat));
      if (starred) res = res.filter(q => starred.has(String(q.id)));
      return shuf(res);
    };

    const mcq = filterQ(Q);
    const ord = state.inclOrd ? filterQ(ORD) : [];
    const tf  = state.inclTF  ? filterQ(TF)  : [];
    const multi = state.inclMulti ? filterQ(MULTI) : [];
    const match = state.inclMatch ? filterQ(MATCH) : [];
    const blank = state.inclBlank ? filterQ(BLANK) : [];
    const all = shuf([...mcq, ...ord, ...tf, ...multi, ...match, ...blank]);

    const di = {}, dd = {}, dok = {};
    all.filter(q => q.type === "ord").forEach(q => {
      di[q.id] = shuf(q.items); dd[q.id] = false; dok[q.id] = false;
    });

    const ms = {}, md = {}, mok = {};
    all.filter(q => q.type === "match").forEach(q => {
      ms[q.id] = { leftIdx: null, pairs: {} };
      md[q.id] = false; mok[q.id] = false;
    });

    const bv = {}, bd = {}, bok = {};
    all.filter(q => q.type === "blank").forEach(q => {
      bv[q.id] = ""; bd[q.id] = false; bok[q.id] = false;
    });

    // Démarrer le timer si activé
    if (state.timerInterval) clearInterval(state.timerInterval);
    let interval = null;
    if (state.timerActive) {
      interval = setInterval(() => {
        setState(s => ({ timerElapsed: s.timerElapsed + 1 }));
      }, 1000);
    }

    setState({
      mode: "quiz", qs: all, cur: 0, sel: null, conf: false,
      score: 0, ans: [], showExp: false,
      dragItems: di, dragDone: dd, dragOk: dok,
      matchSel: ms, matchDone: md, matchOk: mok,
      blankVal: bv, blankDone: bd, blankOk: bok,
      timerElapsed: 0, timerInterval: interval,
    });
  }
  ```

- [ ] **Étape 6 : Mettre à jour `next()` pour enregistrer les tentatives**

  Remplacer `next()` par :
  ```js
  function next() {
    if (state.cur + 1 >= state.qs.length) {
      if (state.timerInterval) clearInterval(state.timerInterval);
      // Sauvegarder la session
      const ls = state.ls;
      ls.lastSession = { score: state.score, total: state.qs.length, date: new Date().toISOString().slice(0,10) };
      saveLS(ls);
      setState({ mode: "results", timerInterval: null });
    } else {
      setState({ cur: state.cur + 1, sel: null, conf: false, showExp: false });
    }
  }
  ```

- [ ] **Étape 7 : Ouvrir dans le navigateur et vérifier**

  Ouvrir `QCM-CARBONE-v2.html` dans Chrome/Firefox. Le menu doit s'afficher normalement. Lancer un quiz → les 67 MCQ + 4 ORD fonctionnent. Console : aucune erreur.

- [ ] **Étape 8 : Commit**
  ```bash
  cd "C:/Users/sylva/OneDrive/Bureau/q"
  git add QCM-CARBONE-v2.html
  git commit -m "feat: base v2 avec système de types, state enrichi et localStorage"
  ```

---

## Task 2 : Renderer Vrai/Faux + 12 questions TF

**Fichier :** `QCM-CARBONE-v2.html`

- [ ] **Étape 1 : Ajouter les 12 questions TF dans le tableau `TF`**

  Remplacer `const TF = [];` par :
  ```js
  const TF = [
    {id:"TF1",type:"tf",cat:"Historique",
      q:"Le Bilan Carbone® a été officiellement formalisé par l'ADEME en 2004.",
      a:true,
      e:"Exact. 2004 = formalisation officielle par l'ADEME avec Jean-Marc Jancovici (Manicore). 2002 = travaux préparatoires. 2011 = création de l'ABC."},
    {id:"TF2",type:"tf",cat:"Historique",
      q:"La marque Bilan Carbone® est déposée uniquement en France.",
      a:false,
      e:"FAUX. La marque® est déposée à l'INPI en France ET dans toute l'Union Européenne."},
    {id:"TF3",type:"tf",cat:"Objectifs & Principes",
      q:"La démarche Bilan Carbone® est obligatoire pour les entreprises de plus de 500 salariés.",
      a:false,
      e:"FAUX. Le BC® est entièrement VOLONTAIRE. C'est le BEGES réglementaire qui est obligatoire pour les organisations de +500 salariés. Le BC® peut y répondre mais reste une démarche volontaire."},
    {id:"TF4",type:"tf",cat:"Objectifs & Principes",
      q:"La comptabilité carbone est une mesure scientifiquement rigoureuse des émissions de GES.",
      a:false,
      e:"FAUX. La comptabilité carbone est un CALCUL, basé sur la science, mais incertain par nature. Elle n'est pas rigoureusement scientifique mais suffisante pour passer à l'action."},
    {id:"TF5",type:"tf",cat:"Les 7 Étapes",
      q:"La démarche Bilan Carbone® v9 est structurée en 7 étapes.",
      a:true,
      e:"Exact. Les 7 étapes : (1) Cadrage, (2) Périmètre, (3) Mobilisation, (4) Comptabilisation, (5) Plan de transition, (6) Synthèse et restitution, (7) Évaluation."},
    {id:"TF6",type:"tf",cat:"Les 7 Étapes",
      q:"L'évaluation (étape 7) est obligatoire pour tous les niveaux de maturité.",
      a:false,
      e:"FAUX. L'évaluation (étape 7) est FACULTATIVE et volontaire. Le BC® doit pouvoir être évalué à tout moment, mais l'audit n'est pas une obligation."},
    {id:"TF7",type:"tf",cat:"Niveaux de Maturité",
      q:"Une organisation peut directement viser le niveau Avancé sans commencer par le niveau Initial.",
      a:true,
      e:"Exact. L'organisation peut viser directement le niveau correspondant à ses objectifs et sa maturité. Un questionnaire de maturité aide à trouver le bon point de départ."},
    {id:"TF8",type:"tf",cat:"Niveaux de Maturité",
      q:"Au niveau Avancé, un seul critère manquant est toléré.",
      a:true,
      e:"Exact. Au niveau Avancé, un seul critère manquant est toléré. Cela reflète la logique d'amélioration continue : progresser, pas atteindre la perfection absolue."},
    {id:"TF9",type:"tf",cat:"Comptabilisation",
      q:"L'unité standard pour exprimer les émissions dans un BC® est la tonne de CO₂ (tCO₂).",
      a:false,
      e:"FAUX. L'unité est la TONNE CO₂ ÉQUIVALENT (tCO₂e). Elle agrège différents GES selon leur PRG-100. tCO₂ sans 'e' = uniquement le CO₂, ce n'est pas l'unité standard."},
    {id:"TF10",type:"tf",cat:"Comptabilisation",
      q:"Le SO₂ est un gaz à effet de serre comptabilisé obligatoirement dans un Bilan Carbone®.",
      a:false,
      e:"FAUX. Le SO₂ est un polluant atmosphérique (pluies acides) mais PAS un GES au sens du BC®. Les GES obligatoires sont : CO₂, CH₄, N₂O, SF₆, HFC, PFC et NF₃."},
    {id:"TF11",type:"tf",cat:"Périmètre",
      q:"Le périmètre temporel classique d'un Bilan Carbone® est d'une année complète.",
      a:true,
      e:"Exact. Le périmètre temporel classique = 1 an, pour refléter l'activité dans sa globalité. Pour répondre au BEGES-R, le périmètre doit obligatoirement être une année complète."},
    {id:"TF12",type:"tf",cat:"Plan de Transition",
      q:"Le plan de transition est obligatoire uniquement pour le niveau Avancé.",
      a:false,
      e:"FAUX. Le plan de transition est OBLIGATOIRE À TOUS LES NIVEAUX (Initial, Standard, Avancé). Sans plan de transition, un BC® est incomplet selon la v9."},
  ];
  ```

- [ ] **Étape 2 : Ajouter la fonction `buildTF()`**

  Après la fonction `buildQuiz()`, ajouter :
  ```js
  function buildTF(q) {
    const { conf, sel, showExp, quizMode } = state;
    const isStudy = quizMode === "study";
    const children = [p({ class: "q-text" }, q.q)];

    const tfRow = div({ style: { display: "flex", gap: "10px", marginBottom: "16px" } }, [
      btn({
        class: `opt${sel === 1 ? " selected" : ""}${conf ? " locked" : ""}${conf && q.a === true ? " correct" : ""}${conf && sel === 1 && q.a !== true ? " wrong" : ""}`,
        style: { flex: "1", justifyContent: "center", fontSize: "16px", fontWeight: "700" },
        onclick: conf ? null : () => setState({ sel: 1 }),
      }, [span({ class: "letter" }, "V"), "Vrai", conf && q.a === true ? span({ class: "tick" }, "✓") : null]),
      btn({
        class: `opt${sel === 0 ? " selected" : ""}${conf ? " locked" : ""}${conf && q.a === false ? " correct" : ""}${conf && sel === 0 && q.a !== false ? " wrong" : ""}`,
        style: { flex: "1", justifyContent: "center", fontSize: "16px", fontWeight: "700" },
        onclick: conf ? null : () => setState({ sel: 0 }),
      }, [span({ class: "letter" }, "F"), "Faux", conf && q.a === false ? span({ class: "tick" }, "✓") : null]),
    ]);
    children.push(tfRow);

    if (conf) {
      const ok = (sel === 1 && q.a === true) || (sel === 0 && q.a === false);
      children.push(div({ class: `feedback ${ok ? "ok" : "ko"}` }, [
        span({ style: { fontSize: "18px" } }, ok ? "✅" : "❌"),
        span({ style: { fontWeight: "600", fontSize: "14px" } }, ok ? "Bonne réponse !" : `Incorrect — Réponse : ${q.a ? "Vrai" : "Faux"}`),
        buildStarBtn(q.id),
      ]));
      if (isStudy || showExp) children.push(div({ class: "exp-box" }, q.e));
      if (!isStudy && !showExp) children.push(btn({ class: "exp-btn", onclick: () => setState({ showExp: true }) }, "▼ Voir l'explication"));
    }

    children.push(div({ class: "actions" }, [
      !conf
        ? btn({ class: "btn", style: { opacity: sel === null ? 0.4 : 1 }, onclick: sel !== null ? confirmTF : null }, "Valider →")
        : btn({ class: "btn", onclick: next }, state.cur + 1 >= state.qs.length ? "Voir les résultats →" : "Question suivante →"),
    ]));
    return div({}, children);
  }

  function confirmTF() {
    const q = state.qs[state.cur];
    const ok = (state.sel === 1 && q.a === true) || (state.sel === 0 && q.a === false);
    setState(s => ({ conf: true, score: ok ? s.score + 1 : s.score, ans: [...s.ans, { q, sel: s.sel, ok }] }));
  }
  ```

- [ ] **Étape 3 : Ajouter la fonction helper `buildStarBtn()`**

  Après les fonctions TF, ajouter :
  ```js
  function buildStarBtn(id) {
    const starred = isStarred(id);
    return btn({
      class: "btn-sm",
      style: { marginLeft: "auto", background: "none", border: "none", fontSize: "16px", cursor: "pointer", padding: "0 4px" },
      onclick: () => toggleStar(id),
      title: starred ? "Retirer des favoris" : "Ajouter aux favoris",
    }, starred ? "⭐" : "☆");
  }
  ```

- [ ] **Étape 4 : Connecter `buildTF` dans `buildQuiz()`**

  Dans `buildQuiz()`, trouver le bloc `if(isO){...} else {...}` et le remplacer par un switch :
  ```js
  if (curQ.type === "ord") {
    // ... (code ord existant, inchangé)
  } else if (curQ.type === "tf") {
    children.push(buildTF(curQ));
  } else if (curQ.type === "multi") {
    children.push(buildMulti(curQ)); // sera ajouté en Task 3
  } else if (curQ.type === "match") {
    children.push(buildMatch(curQ)); // sera ajouté en Task 4
  } else if (curQ.type === "blank") {
    children.push(buildBlank(curQ)); // sera ajouté en Task 5
  } else {
    // mcq (code existant)
  }
  ```

- [ ] **Étape 5 : Ajouter les boutons ⭐ dans les feedbacks MCQ existants**

  Dans le code MCQ, trouver les deux lignes `children.push(div({class:\`feedback...)` et ajouter `buildStarBtn(curQ.id)` dans le feedback div (après le span du message).

- [ ] **Étape 6 : Vérifier en navigateur**

  Ouvrir le fichier, démarrer un quiz avec filtres par défaut. Des questions V/F doivent apparaître. Tester : cliquer Vrai sur une question dont la réponse est Faux → bandeau rouge, explication visible. Tester le bouton ⭐.

- [ ] **Étape 7 : Commit**
  ```bash
  git add QCM-CARBONE-v2.html
  git commit -m "feat: type Vrai/Faux + 12 questions TF + bouton étoile"
  ```

---

## Task 3 : Renderer Multi-réponses + 8 questions MULTI

**Fichier :** `QCM-CARBONE-v2.html`

- [ ] **Étape 1 : Ajouter les 8 questions dans `MULTI`**

  Remplacer `const MULTI = [];` par :
  ```js
  const MULTI = [
    {id:"M1",type:"multi",cat:"Historique",
      q:"Quels éléments désignent officiellement le Bilan Carbone® ? (plusieurs réponses correctes)",
      o:["Une méthode de comptabilisation des GES","Une marque déposée à l'INPI","Un label RSE attribué par l'État","Des outils et formations habilités par l'ABC"],
      a:[0,1,3],
      e:"Le BC® désigne 4 éléments : (1) méthode développée par l'ABC, (2) outils, (3) formations par organismes habilités, (4) marque® déposée. Un label RSE d'État ne fait pas partie de sa définition."},
    {id:"M2",type:"multi",cat:"Objectifs & Principes",
      q:"Quels sont les 5 principes de reporting du Bilan Carbone® ? (plusieurs réponses correctes)",
      o:["Cohérence","Exactitude","Significativité","Compensabilité","Évaluation","Transparence"],
      a:[0,1,2,4,5],
      e:"Les 5 principes : Cohérence, Exactitude, Significativité, Évaluation, Transparence. La 'Compensabilité' n'en fait pas partie — le BC® est orienté réduction, pas compensation."},
    {id:"M3",type:"multi",cat:"Objectifs & Principes",
      q:"Quels sont les 3 grands objectifs indissociables du Bilan Carbone® ?",
      o:["Mobiliser les parties prenantes","Compenser les émissions résiduelles","Comptabiliser les émissions directes et indirectes","Élaborer un plan de transition","Certifier la démarche auprès d'un tiers"],
      a:[0,2,3],
      e:"Les 3 objectifs INDISSOCIABLES : (1) MOBILISER les PP internes et externes, (2) COMPTABILISER les émissions directes et indirectes, (3) Élaborer un PLAN DE TRANSITION ambitieux, pilotable, opérationnel et cohérent."},
    {id:"M4",type:"multi",cat:"Comptabilisation",
      q:"Quels GES doivent obligatoirement être comptabilisés dans un Bilan Carbone® ?",
      o:["CO₂","CH₄","N₂O","SO₂","SF₆","HFC","PFC","NF₃"],
      a:[0,1,2,4,5,6,7],
      e:"GES obligatoires : CO₂, CH₄, N₂O, SF₆, HFC, PFC et NF₃. Le NF₃ est inclus dans le BC® bien qu'absent du Protocole de Kyoto. Le SO₂ est un polluant atmosphérique, pas un GES au sens du BC®."},
    {id:"M5",type:"multi",cat:"Périmètre",
      q:"Quels types de flux doivent être cartographiés dans un Bilan Carbone® ?",
      o:["Flux d'énergie","Flux financiers","Fluides","Matières premières","Personnes (déplacements)","Produits (biens et services)","Déchets"],
      a:[0,2,3,4,5,6],
      e:"Tous les flux physiques entrants, internes et sortants : énergie, fluides, matières premières, personnes, produits et déchets. Les flux financiers ne font pas partie de la cartographie des flux BC®."},
    {id:"M6",type:"multi",cat:"Niveaux de Maturité",
      q:"Quels sont les critères discriminants entre les niveaux de maturité Initial, Standard et Avancé ?",
      o:["Fréquence de renouvellement","Taille de l'organisation","Suivi des actions","Périmètre couvert","Chiffre d'affaires","Importance de la mobilisation"],
      a:[0,2,3,5],
      e:"Les critères discriminants sont : fréquence de renouvellement (4 ans / 2 ans / 1 an), suivi des actions, périmètre couvert, et importance de la mobilisation. La taille et le CA ne sont pas des critères."},
    {id:"M7",type:"multi",cat:"Plan de Transition",
      q:"Quels éléments composent obligatoirement un plan de transition BC® ?",
      o:["Objectifs de réduction alignés sur les scénarios climatiques","Un budget financier détaillé","Une série d'actions détaillées et quantifiées","Une certification externe","Une trajectoire crédible","Des indicateurs de suivi"],
      a:[0,2,4,5],
      e:"Plan de transition = (1) OBJECTIFS de réduction, (2) ACTIONS détaillées et quantifiées avec responsables, (3) TRAJECTOIRE crédible, (4) INDICATEURS de suivi. Un budget financier et une certification externe ne sont pas des exigences obligatoires."},
    {id:"M8",type:"multi",cat:"Synthèse & Restitution",
      q:"Avec quels référentiels le Bilan Carbone® est-il compatible ?",
      o:["ISO 14064-1","GHG Protocol","BEGES réglementaire français","ISO 14001","CSRD ESRS E1 (niveau Avancé)","SA8000"],
      a:[0,1,2,4],
      e:"Le BC® est compatible avec ISO 14064-1, GHG Protocol, BEGES-R, et CSRD ESRS E1 (niveau Avancé). ISO 14001 est un standard de management environnemental (pas de comptabilité carbone). SA8000 concerne le social, pas le carbone."},
  ];
  ```

- [ ] **Étape 2 : Ajouter la fonction `buildMulti()`**

  Après `buildTF()`, ajouter :
  ```js
  function buildMulti(q) {
    const { conf, showExp, quizMode } = state;
    const isStudy = quizMode === "study";
    const multiSel = state.sel || [];
    const children = [
      p({ style: { color: "#8A9DB5", fontSize: "13px", marginBottom: "8px", fontStyle: "italic" } },
        "☑ Plusieurs réponses possibles — cochez toutes les bonnes réponses."),
      p({ class: "q-text" }, q.q),
    ];

    const opts = div({ class: "options" }, q.o.map((o, i) => {
      const checked = multiSel.includes(i);
      let cls = "opt";
      if (conf) {
        cls += " locked";
        if (q.a.includes(i)) cls += " correct";
        else if (checked) cls += " wrong";
      } else if (checked) cls += " selected";

      return btn({ class: cls, onclick: conf ? null : () => {
        const cur = state.sel || [];
        const next = cur.includes(i) ? cur.filter(x => x !== i) : [...cur, i];
        setState({ sel: next });
      }}, [
        span({ class: "letter" }, conf ? (q.a.includes(i) ? "✓" : checked ? "✗" : "○") : (checked ? "☑" : "☐")),
        o,
      ]);
    }));
    children.push(opts);

    if (conf) {
      const ok = q.a.length === multiSel.length && q.a.every(x => multiSel.includes(x));
      children.push(div({ class: `feedback ${ok ? "ok" : "ko"}` }, [
        span({ style: { fontSize: "18px" } }, ok ? "✅" : "❌"),
        span({ style: { fontWeight: "600", fontSize: "14px" } }, ok ? "Parfait !" : `Incorrect — ${q.a.length} bonne(s) réponse(s)`),
        buildStarBtn(q.id),
      ]));
      if (isStudy || showExp) children.push(div({ class: "exp-box" }, q.e));
      if (!isStudy && !showExp) children.push(btn({ class: "exp-btn", onclick: () => setState({ showExp: true }) }, "▼ Voir l'explication"));
    }

    children.push(div({ class: "actions" }, [
      !conf
        ? btn({ class: "btn", style: { opacity: !multiSel.length ? 0.4 : 1 }, onclick: multiSel.length ? confirmMulti : null }, "Valider →")
        : btn({ class: "btn", onclick: next }, state.cur + 1 >= state.qs.length ? "Voir les résultats →" : "Question suivante →"),
    ]));
    return div({}, children);
  }

  function confirmMulti() {
    const q = state.qs[state.cur];
    const multiSel = state.sel || [];
    const ok = q.a.length === multiSel.length && q.a.every(x => multiSel.includes(x));
    setState(s => ({ conf: true, score: ok ? s.score + 1 : s.score, ans: [...s.ans, { q, sel: s.sel, ok }] }));
  }
  ```

- [ ] **Étape 3 : Initialiser `sel` à `[]` pour les questions multi dans `start()`**

  Dans `start()`, après la construction de `all`, ajouter :
  ```js
  // sel initial = [] pour multi, null pour les autres
  // (géré dynamiquement dans buildMulti via state.sel || [])
  ```
  Aucune modification nécessaire — `state.sel` est réinitialisé à `null` dans `next()`, et `buildMulti` utilise `state.sel || []`.

- [ ] **Étape 4 : Vérifier en navigateur**

  Lancer un quiz. Des questions multi-réponses doivent apparaître avec des cases à cocher. Tester : sélectionner des cases, décocher, valider avec une sélection incomplète → feedback rouge. Valider la bonne combinaison → feedback vert.

- [ ] **Étape 5 : Commit**
  ```bash
  git add QCM-CARBONE-v2.html
  git commit -m "feat: type multi-réponses + 8 questions MULTI"
  ```

---

## Task 4 : Renderer Associations + 6 questions MATCH

**Fichier :** `QCM-CARBONE-v2.html`

- [ ] **Étape 1 : Ajouter les 6 questions dans `MATCH`**

  Remplacer `const MATCH = [];` par :
  ```js
  const MATCH = [
    {id:"MA1",type:"match",cat:"Niveaux de Maturité",
      q:"Associez chaque niveau de maturité à sa fréquence minimale de renouvellement :",
      pairs:[{l:"Initial",r:"Maximum tous les 4 ans"},{l:"Standard",r:"Tous les 2 ans"},{l:"Avancé",r:"Tous les ans"}],
      e:"Fréquences : Initial → max 4 ans. Standard → 2 ans. Avancé → TOUS LES ANS (outil de pilotage stratégique annuel)."},
    {id:"MA2",type:"match",cat:"Les 7 Étapes",
      q:"Associez chaque étape à son numéro dans la démarche BC® :",
      pairs:[{l:"Cadrage",r:"Étape 1"},{l:"Comptabilisation",r:"Étape 4"},{l:"Plan de transition",r:"Étape 5"},{l:"Évaluation",r:"Étape 7"}],
      e:"Les 7 étapes : (1) Cadrage, (2) Périmètre, (3) Mobilisation, (4) Comptabilisation, (5) Plan de transition, (6) Synthèse/Restitution, (7) Évaluation."},
    {id:"MA3",type:"match",cat:"Comptabilisation",
      q:"Associez chaque GES à son PRG-100 approximatif :",
      pairs:[{l:"CO₂",r:"1"},{l:"CH₄",r:"≈ 28"},{l:"N₂O",r:"≈ 265"},{l:"SF₆",r:"≈ 23 900"}],
      e:"PRG-100 : CO₂ = 1 (référence), CH₄ ≈ 28, N₂O ≈ 265, SF₆ ≈ 23 900. Ces valeurs sont fixées par le GIEC et mises à jour régulièrement."},
    {id:"MA4",type:"match",cat:"Mobilisation",
      q:"Associez chaque phase de mobilisation à son objectif :",
      pairs:[{l:"Sensibilisation/vulgarisation",r:"Comprendre pour agir"},{l:"Responsabilisation",r:"Donner un rôle à chacun"},{l:"Coconstruction",r:"Co-bâtir le plan d'action"},{l:"Restitution/communication",r:"Permettre l'appropriation des résultats"}],
      e:"Les 4 phases : (1) Sensibilisation → comprendre. (2) Responsabilisation → impliquer. (3) Coconstruction → co-bâtir le plan. (4) Restitution → appropriation et communication."},
    {id:"MA5",type:"match",cat:"Périmètre",
      q:"Associez chaque périmètre à sa définition :",
      pairs:[{l:"Périmètre organisationnel",r:"Quelles entités de l'organisation ?"},{l:"Périmètre temporel",r:"Quelle période ?"},{l:"Périmètre opérationnel",r:"Quels postes d'émissions ?"}],
      e:"Les 4 périmètres : (1) Émissions comptabilisées (quels GES ?), (2) Organisationnel (quelles entités ?), (3) Temporel (quelle période ?), (4) Opérationnel (quels postes ?)."},
    {id:"MA6",type:"match",cat:"Synthèse & Restitution",
      q:"Associez chaque indicateur du plan de transition à son rôle :",
      pairs:[{l:"Indicateur de suivi",r:"Pilotage de la démarche"},{l:"Indicateur de mise en œuvre",r:"Avancement des actions"},{l:"Indicateur de performance",r:"Résultats obtenus"}],
      e:"Les 3 types d'indicateurs du plan de transition : suivi (pilotage), mise en œuvre (avancement), performance (résultats). Ensemble, ils permettent un suivi complet de la décarbonation."},
  ];
  ```

- [ ] **Étape 2 : Ajouter le CSS pour le type match**

  Dans la balise `<style>`, ajouter avant la fermeture `</style>` :
  ```css
  /* Match */
  .match-wrap{display:grid;grid-template-columns:1fr 1fr;gap:8px;margin-bottom:16px}
  .match-col{display:flex;flex-direction:column;gap:6px}
  .match-item{padding:9px 12px;border-radius:9px;border:1px solid #2A3A52;background:#1A2335;color:#C8D8E8;font-size:13px;cursor:pointer;text-align:center;transition:all 0.12s;user-select:none}
  .match-item:hover:not(.locked){border-color:#52B788;background:#162A4A}
  .match-item.selected{border-color:#E9A23B;background:rgba(233,162,59,0.1);color:#E9A23B}
  .match-item.paired{border-color:#52B788;background:#1B3A2A;color:#52B788;cursor:default}
  .match-item.wrong-pair{border-color:#E07A5F;background:#3A1B1B;color:#E07A5F;cursor:default}
  .match-item.locked{cursor:default}
  .match-line{font-size:11px;color:#6A7D90;text-align:center;margin-bottom:4px}
  ```

- [ ] **Étape 3 : Ajouter la fonction `buildMatch()`**

  Après `buildMulti()` + `confirmMulti()`, ajouter :
  ```js
  function buildMatch(q) {
    const { conf, showExp, quizMode } = state;
    const isStudy = quizMode === "study";
    const qid = q.id;
    const ms = state.matchSel[qid] || { leftIdx: null, pairs: {} };
    const done = state.matchDone[qid];
    const ok = state.matchOk[qid];

    const children = [
      p({ style: { color: "#8A9DB5", fontSize: "13px", marginBottom: "8px", fontStyle: "italic" } },
        "🔗 Cliquez sur un élément gauche puis son correspondant à droite pour les associer."),
      p({ class: "q-text" }, q.q),
    ];

    // Shuffle right column (only once, store in matchSel)
    if (!ms.shuffledRight) {
      ms.shuffledRight = shuf(q.pairs.map((p, i) => i));
    }
    const shuffledRight = ms.shuffledRight;

    const leftCol = div({ class: "match-col" }, [
      p({ class: "match-line" }, "Éléments"),
      ...q.pairs.map((pair, i) => {
        const isPaired = ms.pairs[i] !== undefined;
        const isCorrect = done && ms.pairs[i] === i;
        const isWrong = done && ms.pairs[i] !== undefined && ms.pairs[i] !== i;
        return div({
          class: `match-item${ms.leftIdx === i ? " selected" : ""}${isPaired && !done ? " paired" : ""}${isCorrect ? " paired" : ""}${isWrong ? " wrong-pair" : ""}${done ? " locked" : ""}`,
          onclick: done ? null : () => {
            const newMs = { ...ms, leftIdx: ms.leftIdx === i ? null : i };
            setState(s => ({ matchSel: { ...s.matchSel, [qid]: newMs } }));
          },
        }, pair.l);
      }),
    ]);

    const rightCol = div({ class: "match-col" }, [
      p({ class: "match-line" }, "Correspondances"),
      ...shuffledRight.map((origIdx, ri) => {
        const pairedByLeft = Object.entries(ms.pairs).find(([l, r]) => Number(r) === origIdx);
        const isPaired = !!pairedByLeft;
        const leftIdx = isPaired ? Number(pairedByLeft[0]) : -1;
        const isCorrect = done && isPaired && leftIdx === origIdx;
        const isWrong = done && isPaired && leftIdx !== origIdx;
        return div({
          class: `match-item${isPaired && !done ? " paired" : ""}${isCorrect ? " paired" : ""}${isWrong ? " wrong-pair" : ""}${done ? " locked" : ""}`,
          onclick: (done || isPaired) ? null : () => {
            if (ms.leftIdx === null) return;
            const newPairs = { ...ms.pairs, [ms.leftIdx]: origIdx };
            const newMs = { ...ms, leftIdx: null, pairs: newPairs };
            setState(s => ({ matchSel: { ...s.matchSel, [qid]: newMs } }));
          },
        }, q.pairs[origIdx].r);
      }),
    ]);

    children.push(div({ class: "match-wrap" }, [leftCol, rightCol]));

    const allPaired = Object.keys(ms.pairs).length === q.pairs.length;
    if (!done) {
      if (allPaired) {
        children.push(btn({ class: "btn", onclick: () => submitMatch(qid, q) }, "Valider les associations →"));
      }
    } else {
      children.push(div({ class: `feedback ${ok ? "ok" : "ko"}` }, [
        span({ style: { fontSize: "18px" } }, ok ? "✅" : "❌"),
        span({ style: { fontWeight: "600", fontSize: "14px" } }, ok ? "Toutes les associations sont correctes !" : "Certaines associations sont incorrectes — voir les corrections ci-dessus"),
        buildStarBtn(q.id),
      ]));
      if (isStudy || showExp) children.push(div({ class: "exp-box" }, q.e));
      if (!isStudy && !showExp) children.push(btn({ class: "exp-btn", onclick: () => setState({ showExp: true }) }, "▼ Voir l'explication"));
      children.push(div({ style: { marginTop: "10px" } }, [
        btn({ class: "btn", onclick: next }, state.cur + 1 >= state.qs.length ? "Voir les résultats →" : "Question suivante →"),
      ]));
    }
    return div({}, children);
  }

  function submitMatch(qid, q) {
    const ms = state.matchSel[qid];
    const ok = q.pairs.every((_, i) => ms.pairs[i] === i);
    setState(s => ({
      matchDone: { ...s.matchDone, [qid]: true },
      matchOk: { ...s.matchOk, [qid]: ok },
      score: ok ? s.score + 1 : s.score,
      ans: [...s.ans, { q, ok }],
    }));
  }
  ```

- [ ] **Étape 4 : Vérifier en navigateur**

  Démarrer un quiz. Les questions d'association doivent afficher deux colonnes cliquables. Tester : cliquer un élément gauche (jaune), cliquer un élément droit → lien formé (vert). Valider → feedback avec corrections.

- [ ] **Étape 5 : Commit**
  ```bash
  git add QCM-CARBONE-v2.html
  git commit -m "feat: type association (match) + 6 questions MATCH"
  ```

---

## Task 5 : Renderer Texte à trous + 8 questions BLANK

**Fichier :** `QCM-CARBONE-v2.html`

- [ ] **Étape 1 : Ajouter les 8 questions dans `BLANK`**

  Remplacer `const BLANK = [];` par :
  ```js
  const BLANK = [
    {id:"B1",type:"blank",cat:"Objectifs & Principes",
      q:"La philosophie centrale du Bilan Carbone® est « [____] pour agir ».",
      a:["compter"],synonyms:[],
      e:"« Compter pour agir » : la comptabilisation n'est pas une fin en soi mais un outil au service de l'action et de la réduction."},
    {id:"B2",type:"blank",cat:"Historique",
      q:"Le Bilan Carbone® a été officiellement formalisé en [____] par l'ADEME.",
      a:["2004"],synonyms:[],
      e:"2004 = formalisation officielle par l'ADEME avec Jean-Marc Jancovici. 2002 = travaux préparatoires. 2011 = création de l'ABC."},
    {id:"B3",type:"blank",cat:"Historique",
      q:"L'ABC (Association pour la transition Bas Carbone) a été créée en [____].",
      a:["2011"],synonyms:[],
      e:"L'ABC a été créée en 2011 par l'ADEME et l'APCC pour reprendre la gouvernance de la méthode."},
    {id:"B4",type:"blank",cat:"Comptabilisation",
      q:"La formule de base du calcul des émissions est : Données d'activité × [____] = Résultat ± Incertitude.",
      a:["facteur d'émission"],synonyms:["fe","facteur emission","facteurs d'émission"],
      e:"Formule fondamentale : DA × FE = Résultat ± Incertitude. Le facteur d'émission (FE) convertit les données d'activité en tCO₂e."},
    {id:"B5",type:"blank",cat:"Comptabilisation",
      q:"Les émissions de GES sont exprimées en [____] dans un Bilan Carbone®.",
      a:["tco2e"],synonyms:["tonne co2 équivalent","tonnes co2e","tco₂e","tco2 équivalent","tonne co2e"],
      e:"L'unité est la TONNE CO₂ ÉQUIVALENT (tCO₂e). Elle agrège différents GES selon leur PRG-100 sur 100 ans."},
    {id:"B6",type:"blank",cat:"Niveaux de Maturité",
      q:"Le niveau Avancé du Bilan Carbone® doit être renouvelé tous les [____].",
      a:["an","ans","1 an"],synonyms:["année","chaque année","annuellement"],
      e:"Niveau Avancé : renouvellement ANNUEL. Initial → max 4 ans. Standard → 2 ans. La fréquence croissante traduit l'ambition du niveau."},
    {id:"B7",type:"blank",cat:"Périmètre",
      q:"La Base [____]® de l'ADEME est la référence nationale française pour les facteurs d'émission.",
      a:["carbone"],synonyms:[],
      e:"La BASE CARBONE® de l'ADEME est la référence nationale française pour les facteurs d'émission. Elle est accessible gratuitement en ligne."},
    {id:"B8",type:"blank",cat:"Synthèse & Restitution",
      q:"Le profil GES anonymisé doit obligatoirement être déposé sur l'[____] (plateforme nationale).",
      a:["occf"],synonyms:["observatoire de la comptabilité carbone en france","observatoire comptabilité carbone"],
      e:"L'OCCF (Observatoire de la Comptabilité Carbone en France) est la plateforme où chaque organisation DOIT déposer son profil GES anonymisé. C'est une exigence du principe de TRANSPARENCE."},
  ];
  ```

- [ ] **Étape 2 : Ajouter le CSS pour le type blank**

  Dans `<style>`, ajouter :
  ```css
  /* Blank */
  .blank-input{background:#0F1E30;border:1px solid #2A3A52;border-radius:7px;padding:9px 14px;color:#E8F4EC;font-size:15px;font-family:Georgia,Palatino,serif;width:100%;box-sizing:border-box;margin-bottom:12px;outline:none}
  .blank-input:focus{border-color:#52B788}
  .blank-input.correct{border-color:#52B788;color:#52B788;background:#1B3A2A}
  .blank-input.wrong{border-color:#E07A5F;color:#E07A5F;background:#3A1B1B}
  .blank-q{color:#E8F4EC;font-size:17px;line-height:1.6;margin-bottom:12px;font-weight:500}
  .blank-placeholder{background:#2A3A52;border-radius:4px;padding:2px 8px;color:#8ABFDB;font-style:italic}
  ```

- [ ] **Étape 3 : Ajouter la fonction `buildBlank()`**

  Après `submitMatch()`, ajouter :
  ```js
  function normalizeBlank(s) {
    return s.trim().toLowerCase()
      .normalize("NFD").replace(/[\u0300-\u036f]/g, "")
      .replace(/[®]/g, "");
  }

  function buildBlank(q) {
    const { showExp, quizMode } = state;
    const isStudy = quizMode === "study";
    const qid = q.id;
    const val = state.blankVal[qid] || "";
    const done = state.blankDone[qid];
    const ok = state.blankOk[qid];

    // Render question with placeholder styled
    const qHtml = q.q.replace("[____]", '<span class="blank-placeholder">____</span>');
    const qEl = div({ class: "blank-q" });
    qEl.innerHTML = qHtml;

    const input = h("input", {
      class: `blank-input${done ? (ok ? " correct" : " wrong") : ""}`,
      type: "text",
      placeholder: "Votre réponse...",
      value: val,
      disabled: done,
    });
    input.value = val;
    if (!done) {
      input.addEventListener("input", e => {
        setState(s => ({ blankVal: { ...s.blankVal, [qid]: e.target.value } }));
      });
      input.addEventListener("keydown", e => {
        if (e.key === "Enter" && val.trim()) submitBlank(qid, q, val);
      });
    }

    const children = [qEl, input];

    if (done) {
      children.push(div({ class: `feedback ${ok ? "ok" : "ko"}` }, [
        span({ style: { fontSize: "18px" } }, ok ? "✅" : "❌"),
        span({ style: { fontWeight: "600", fontSize: "14px" } },
          ok ? "Bonne réponse !" : `Incorrect — Réponse attendue : « ${q.a[0]} »`),
        buildStarBtn(q.id),
      ]));
      if (isStudy || showExp) children.push(div({ class: "exp-box" }, q.e));
      if (!isStudy && !showExp) children.push(btn({ class: "exp-btn", onclick: () => setState({ showExp: true }) }, "▼ Voir l'explication"));
      children.push(div({ style: { marginTop: "10px" } }, [
        btn({ class: "btn", onclick: next }, state.cur + 1 >= state.qs.length ? "Voir les résultats →" : "Question suivante →"),
      ]));
    } else {
      children.push(div({ class: "actions" }, [
        btn({ class: "btn", style: { opacity: !val.trim() ? 0.4 : 1 },
          onclick: val.trim() ? () => submitBlank(qid, q, val) : null }, "Valider →"),
      ]));
    }
    return div({}, children);
  }

  function submitBlank(qid, q, val) {
    const norm = normalizeBlank(val);
    const allAccepted = [...q.a, ...q.synonyms].map(normalizeBlank);
    const ok = allAccepted.includes(norm);
    setState(s => ({
      blankDone: { ...s.blankDone, [qid]: true },
      blankOk: { ...s.blankOk, [qid]: ok },
      score: ok ? s.score + 1 : s.score,
      ans: [...s.ans, { q, sel: val, ok }],
    }));
  }
  ```

- [ ] **Étape 4 : Vérifier en navigateur**

  Lancer un quiz. Des questions à trous doivent apparaître avec un champ texte. Tester : taper "compter" (minuscules) → ✅. Taper "Compter" (majuscule) → ✅ (insensible à la casse). Taper une mauvaise réponse → ❌ avec la bonne réponse affichée.

- [ ] **Étape 5 : Commit**
  ```bash
  git add QCM-CARBONE-v2.html
  git commit -m "feat: type texte à trous + 8 questions BLANK"
  ```

---

## Task 6 : Mode Étude/Examen + Timer + Explication auto si faux

**Fichier :** `QCM-CARBONE-v2.html`

- [ ] **Étape 1 : Ajouter les contrôles dans `buildFilter()`**

  Dans `buildFilter()`, avant le `p({class:"total-count"...})`, ajouter :
  ```js
  // Séparateur
  div({style:{height:"1px",background:"#1E3048",margin:"10px 0"}}),

  // Mode
  h("p",{style:{color:"#8A9DB5",fontSize:"12px",fontWeight:"600",textTransform:"uppercase",letterSpacing:"1px",marginBottom:"8px"}},"Mode"),
  div({style:{display:"flex",gap:"8px",marginBottom:"12px"}},[
    btn({
      class:`btn${state.quizMode==="study"?"":" btn-dark"}`,
      style:{flex:"1",fontSize:"13px",padding:"8px"},
      onclick:()=>setState({quizMode:"study"})
    },"📖 Étude"),
    btn({
      class:`btn${state.quizMode==="exam"?"":" btn-dark"}`,
      style:{flex:"1",fontSize:"13px",padding:"8px"},
      onclick:()=>setState({quizMode:"exam"})
    },"📝 Examen"),
  ]),

  // Timer
  h("p",{style:{color:"#8A9DB5",fontSize:"12px",fontWeight:"600",textTransform:"uppercase",letterSpacing:"1px",marginBottom:"8px"}},"Timer"),
  div({class:`cat-row${state.timerActive?" active":""}`,style:{marginBottom:"12px"},onclick:()=>setState({timerActive:!state.timerActive})},[
    div({class:"cat-dot",style:{background:state.timerActive?"#E9A23B":"#2A3A52"}}),
    span({class:"cat-name"},"Activer le chronomètre"),
    span({style:{color:state.timerActive?"#E9A23B":"#3A4A5A",fontSize:"12px"}},state.timerActive?"ON":"OFF"),
    span({class:"cat-check",style:{color:state.timerActive?"#E9A23B":""}},state.timerActive?"☑":"☐"),
  ]),
  ```

- [ ] **Étape 2 : Afficher le timer dans `buildQuiz()`**

  Dans `buildQuiz()`, après la ligne du `progress-bar`, ajouter le timer si actif :
  ```js
  if (state.timerActive) {
    const elapsed = state.timerElapsed;
    const over = elapsed > state.timerDuration;
    const display = over
      ? `⏱ +${formatTime(elapsed - state.timerDuration)}`
      : `⏱ ${formatTime(state.timerDuration - elapsed)}`;
    children.push(div({
      style: {
        textAlign: "right", fontSize: "13px", fontWeight: "700",
        color: over ? "#E07A5F" : "#E9A23B", marginBottom: "8px"
      }
    }, display));
  }
  ```

  Puis ajouter la fonction helper `formatTime()` dans les helpers :
  ```js
  function formatTime(secs) {
    const m = Math.floor(secs / 60).toString().padStart(2, "0");
    const s = (secs % 60).toString().padStart(2, "0");
    return `${m}:${s}`;
  }
  ```

- [ ] **Étape 3 : Modifier `confirm()` pour l'explication auto en mode Étude**

  Remplacer la fonction `confirm()` existante par :
  ```js
  function confirm() {
    const q = state.qs[state.cur];
    if (state.sel === null) return;
    const ok = state.sel === q.a;
    setState(s => ({
      conf: true,
      score: ok ? s.score + 1 : s.score,
      ans: [...s.ans, { q, sel: s.sel, ok }],
      showExp: s.quizMode === "study" && !ok, // auto si étude + faux
    }));
  }
  ```

  Faire la même chose dans `confirmTF()` et `confirmMulti()` — ajouter `showExp: state.quizMode === "study" && !ok` dans le setState.

- [ ] **Étape 4 : Mode Examen — masquer les feedbacks dans buildResults()**

  Aucun changement nécessaire dans `buildResults()` — les feedback sont déjà dans `buildQuiz()`. En mode Examen, les explications sont masquées pendant le quiz. À la fin, `buildResults()` affiche le score global et le bouton "Revoir les questions" — le mode review montre déjà les bonnes réponses.

- [ ] **Étape 5 : Vérifier en navigateur**

  Tester mode Étude : répondre faux → explication s'affiche automatiquement. Tester mode Examen : répondre faux → pas d'explication, juste "Incorrect". Activer le timer → chrono visible en haut à droite. Laisser s'écouler → affichage "⏱ +00:05" en rouge.

- [ ] **Étape 6 : Commit**
  ```bash
  git add QCM-CARBONE-v2.html
  git commit -m "feat: mode étude/examen, timer global, explication auto si faux"
  ```

---

## Task 7 : Filtre "À revoir" + section résultats + toggles nouveaux types

**Fichier :** `QCM-CARBONE-v2.html`

- [ ] **Étape 1 : Ajouter les toggles TF/Multi/Match/Blank dans `buildFilter()`**

  Après le toggle `inclOrd` existant, ajouter :
  ```js
  // Toggle TF
  div({class:`cat-row${state.inclTF?" active":""}`,style:{borderColor:state.inclTF?"#52B788":"",marginBottom:"5px"},onclick:()=>setState({inclTF:!state.inclTF})},[
    div({class:"cat-dot",style:{background:state.inclTF?"#52B788":"#2A3A52"}}),
    span({class:"cat-name"},"Inclure les Vrai/Faux"),
    span({style:{color:state.inclTF?"#52B788":"#3A4A5A",fontSize:"12px"}},`${TF.length} Q`),
    span({class:"cat-check",style:{color:state.inclTF?"#52B788":""}},state.inclTF?"☑":"☐"),
  ]),
  // Toggle Multi
  div({class:`cat-row${state.inclMulti?" active":""}`,style:{borderColor:state.inclMulti?"#40916C":"",marginBottom:"5px"},onclick:()=>setState({inclMulti:!state.inclMulti})},[
    div({class:"cat-dot",style:{background:state.inclMulti?"#40916C":"#2A3A52"}}),
    span({class:"cat-name"},"Inclure les Multi-réponses"),
    span({style:{color:state.inclMulti?"#40916C":"#3A4A5A",fontSize:"12px"}},`${MULTI.length} Q`),
    span({class:"cat-check",style:{color:state.inclMulti?"#40916C":""}},state.inclMulti?"☑":"☐"),
  ]),
  // Toggle Match
  div({class:`cat-row${state.inclMatch?" active":""}`,style:{borderColor:state.inclMatch?"#2D6A4F":"",marginBottom:"5px"},onclick:()=>setState({inclMatch:!state.inclMatch})},[
    div({class:"cat-dot",style:{background:state.inclMatch?"#2D6A4F":"#2A3A52"}}),
    span({class:"cat-name"},"Inclure les Associations"),
    span({style:{color:state.inclMatch?"#2D6A4F":"#3A4A5A",fontSize:"12px"}},`${MATCH.length} Q`),
    span({class:"cat-check",style:{color:state.inclMatch?"#2D6A4F":""}},state.inclMatch?"☑":"☐"),
  ]),
  // Toggle Blank
  div({class:`cat-row${state.inclBlank?" active":""}`,style:{borderColor:state.inclBlank?"#1B5E6D":"",marginBottom:"14px"},onclick:()=>setState({inclBlank:!state.inclBlank})},[
    div({class:"cat-dot",style:{background:state.inclBlank?"#1B5E6D":"#2A3A52"}}),
    span({class:"cat-name"},"Inclure les Textes à trous"),
    span({style:{color:state.inclBlank?"#1B5E6D":"#3A4A5A",fontSize:"12px"}},`${BLANK.length} Q`),
    span({class:"cat-check",style:{color:state.inclBlank?"#1B5E6D":""}},state.inclBlank?"☑":"☐"),
  ]),
  ```

- [ ] **Étape 2 : Ajouter le bouton "À revoir" dans `buildFilter()`**

  Juste avant les boutons Retour/Démarrer dans `buildFilter()`, ajouter :
  ```js
  // Bouton À revoir
  (() => {
    const n = state.ls.starred.length;
    return div({style:{marginBottom:"12px",textAlign:"center"}},[
      btn({
        class:`btn${state.filterStarred?" ":" btn-dark"}`,
        style:{fontSize:"13px",opacity:n===0?0.4:1},
        onclick: n > 0 ? ()=>setState({filterStarred:!state.filterStarred}) : null,
        title: n === 0 ? "Marque des questions ⭐ pour les revoir ici" : "",
      }, state.filterStarred ? `⭐ Mode "À revoir" actif (${n} Q) — Tout afficher` : `⭐ À revoir uniquement (${n} Q)`),
    ]);
  })(),
  ```

- [ ] **Étape 3 : Enrichir `buildResults()` avec section "À revoir"**

  Dans `buildResults()`, après `div({class:"flex-center"}...)`, ajouter :
  ```js
  // Questions ratées → suggestion d'ajout aux étoiles
  const failed = ans.filter(a => !a.ok && a.q);
  if (failed.length > 0) {
    children.push(h("p",{class:"section-title",style:{marginTop:"18px"}},"Questions à revoir"));
    failed.slice(0, 5).forEach(a => {
      const starred = isStarred(a.q.id);
      children.push(div({
        style:{display:"flex",alignItems:"center",gap:"8px",padding:"8px 10px",
               background:"#131D2A",borderRadius:"8px",marginBottom:"5px",border:"1px solid #1E3048"}
      },[
        span({style:{flex:"1",fontSize:"12px",color:"#8A9DB5",lineHeight:"1.4"}},a.q.q.slice(0,60)+"…"),
        btn({
          style:{background:"none",border:"none",fontSize:"16px",cursor:"pointer",padding:"2px 6px"},
          onclick:()=>toggleStar(a.q.id),
          title: starred ? "Retirer des favoris" : "Ajouter aux favoris",
        }, starred ? "⭐" : "☆"),
      ]));
    });
    if (failed.length > 5) {
      children.push(p({style:{color:"#6A7D90",fontSize:"12px",textAlign:"center"}},`+${failed.length-5} autres questions ratées`));
    }
  }
  ```

- [ ] **Étape 4 : Afficher l'historique dernière session dans `buildMenu()`**

  Dans `buildMenu()`, avant le bouton "Commencer →", ajouter :
  ```js
  (() => {
    const last = state.ls.lastSession;
    if (!last) return null;
    const pct = Math.round((last.score / last.total) * 100);
    return p({style:{textAlign:"center",color:"#52B788",fontSize:"13px",marginBottom:"14px"}},
      `Dernière session : ${last.score}/${last.total} (${pct}%) — ${last.date}`);
  })(),
  ```

- [ ] **Étape 5 : Mettre à jour le compteur de questions dans `buildFilter()`**

  La fonction `cnt` existante ne compte que MCQ + ORD. La remplacer par :
  ```js
  const cnt =
    Q.filter(q => state.selCats.has(q.cat)).length +
    (state.inclOrd   ? ORD.filter(o => state.selCats.has(o.cat)).length : 0) +
    (state.inclTF    ? TF.filter(q => state.selCats.has(q.cat)).length  : 0) +
    (state.inclMulti ? MULTI.filter(q => state.selCats.has(q.cat)).length : 0) +
    (state.inclMatch ? MATCH.filter(q => state.selCats.has(q.cat)).length : 0) +
    (state.inclBlank ? BLANK.filter(q => state.selCats.has(q.cat)).length : 0);
  ```

- [ ] **Étape 6 : Vérifier en navigateur**

  - Filtres : tous les toggles (V/F, Multi, Match, Blank) fonctionnent, le compteur se met à jour.
  - Après un quiz : section "Questions à revoir" affiche les questions ratées avec bouton ⭐.
  - Cliquer ⭐ → question sauvegardée. Revenir au menu → "À revoir (N Q)" actif.
  - Lancer "À revoir uniquement" → seules les questions ⭐ apparaissent.

- [ ] **Étape 7 : Commit**
  ```bash
  git add QCM-CARBONE-v2.html
  git commit -m "feat: filtre à revoir, toggles nouveaux types, section résultats enrichie"
  ```

---

## Task 8 : Ajouter l'onglet ⭐ dans le mode review + mode Examen dans buildReview()

**Fichier :** `QCM-CARBONE-v2.html`

- [ ] **Étape 1 : Ajouter le bouton ⭐ dans `buildReview()`**

  Dans `buildReview()`, après le bloc `opts`, ajouter :
  ```js
  div({style:{display:"flex",alignItems:"center",gap:"8px",margin:"4px 0 8px"}},[
    span({style:{color:"#6A7D90",fontSize:"12px"}},"Mémorisation :"),
    btn({
      style:{background:"none",border:"1px solid #2A3A52",borderRadius:"7px",padding:"3px 10px",cursor:"pointer",fontSize:"14px"},
      onclick:()=>toggleStar(rv.q.id),
      title: isStarred(rv.q.id) ? "Retirer des favoris" : "Ajouter à revoir",
    }, isStarred(rv.q.id) ? "⭐ Marqué" : "☆ À revoir"),
  ]),
  ```

- [ ] **Étape 2 : Gérer les types multi/match/blank dans `buildReview()`**

  Dans `buildReview()`, remplacer le bloc `const opts = rv.q.o ? ...` par :
  ```js
  let opts;
  if (rv.q.type === "multi" && rv.q.o) {
    opts = div({class:"options",style:{marginBottom:"14px"}},rv.q.o.map((o,i)=>{
      let cls="opt locked";
      if(rv.q.a.includes(i))cls+=" correct";
      else if((rv.sel||[]).includes(i))cls+=" wrong";
      return div({class:cls},[
        span({class:"letter"},rv.q.a.includes(i)?"✓":(rv.sel||[]).includes(i)?"✗":"○"),o]);
    }));
  } else if (rv.q.type === "tf") {
    const label = rv.q.a ? "Vrai ✓" : "Faux ✓";
    opts = p({style:{color:"#52B788",fontWeight:"700",marginBottom:"14px"}},`Réponse correcte : ${label}`);
  } else if (rv.q.type === "match") {
    opts = div({style:{marginBottom:"14px"}},rv.q.pairs.map(pair=>
      div({style:{display:"flex",gap:"8px",alignItems:"center",marginBottom:"5px"}},[
        span({style:{flex:1,padding:"6px 10px",background:"#1B3A2A",borderRadius:"7px",color:"#52B788",fontSize:"13px"}},pair.l),
        span({style:{color:"#52B788"}}),"→",
        span({style:{flex:1,padding:"6px 10px",background:"#1B3A2A",borderRadius:"7px",color:"#52B788",fontSize:"13px"}},pair.r),
      ])
    ));
  } else if (rv.q.type === "blank") {
    opts = p({style:{color:"#52B788",fontWeight:"700",marginBottom:"14px"}},`Réponse : « ${rv.q.a[0]} »`);
  } else if (rv.q.o) {
    opts = div({class:"options",style:{marginBottom:"14px"}},rv.q.o.map((o,i)=>{
      let cls="opt locked";
      if(i===rv.q.a)cls+=" correct";
      else if(i===rv.sel&&!rv.ok)cls+=" wrong";
      return div({class:cls},[
        span({class:"letter"},"ABCD"[i]),o,
        i===rv.q.a?span({class:"tick"},"✓"):null,
        i===rv.sel&&!rv.ok?span({class:"tick"},"✗"):null,
      ]);
    }));
  } else {
    opts = p({style:{color:"#8A9DB5",fontStyle:"italic"}},"Question d'ordre");
  }
  ```

- [ ] **Étape 3 : Vérifier en navigateur**

  Terminer un quiz, cliquer "Revoir les questions". Naviguer entre les questions → le bouton ⭐ est présent. Cliquer ⭐ → "Marqué". Naviguer vers d'autres types (TF, match, blank) → affichage correct de la correction.

- [ ] **Étape 4 : Commit**
  ```bash
  git add QCM-CARBONE-v2.html
  git commit -m "feat: bouton étoile dans review, support tous types en review"
  ```

---

## Task 9 : Mettre à jour le menu + CATS + polish final

**Fichier :** `QCM-CARBONE-v2.html`

- [ ] **Étape 1 : Mettre à jour `CATS` pour inclure les catégories des nouveaux types**

  La variable `CATS` est définie comme :
  ```js
  const CATS = [...new Set(Q.map(q => q.cat))];
  ```
  La remplacer par :
  ```js
  const CATS = [...new Set([
    ...Q.map(q => q.cat),
    ...ORD.map(q => q.cat),
    ...TF.map(q => q.cat),
    ...MULTI.map(q => q.cat),
    ...MATCH.map(q => q.cat),
    ...BLANK.map(q => q.cat),
  ])];
  ```

- [ ] **Étape 2 : Mettre à jour le titre et description dans `buildMenu()`**

  Remplacer le `p({class:"desc"...})` existant par :
  ```js
  p({class:"desc",style:{textAlign:"center"}},
    `${Q.length + ORD.length + TF.length + MULTI.length + MATCH.length + BLANK.length} questions · 5 types · ${CATS.length} thèmes`),
  ```

- [ ] **Étape 3 : Ajouter les badges de types dans `buildQuiz()`**

  Dans `div({class:"quiz-meta"...})`, après le badge catégorie, ajouter un badge de type :
  ```js
  const typeLabels = { mcq: "QCM", ord: "Ordre", tf: "V/F", multi: "Multi", match: "Association", blank: "Texte" };
  const typeColors = { mcq: "#2D6A4F", ord: "#E9A23B", tf: "#1B5E6D", multi: "#744210", match: "#3D405B", blank: "#4A4E69" };
  span({class:"badge",style:{background:typeColors[curQ.type]||"#2D6A4F"}}, typeLabels[curQ.type]||curQ.type),
  ```

- [ ] **Étape 4 : Vérifier l'intégration complète en navigateur**

  Test exhaustif :
  1. Ouvrir le fichier → menu affiche le bon nombre de questions
  2. Lancer un quiz complet (tous types) → tous les types s'affichent correctement
  3. Activer timer → chrono fonctionne
  4. Mode examen → pas d'explication pendant, tout révélé à la fin
  5. Marquer des questions ⭐ → persistance après rechargement (F5)
  6. Filtre "À revoir" → seulement les questions ⭐
  7. Mode review → tous les types affichent la bonne correction
  8. Console navigateur → aucune erreur

- [ ] **Étape 5 : Commit final**
  ```bash
  git add QCM-CARBONE-v2.html
  git commit -m "feat: QCM BC v2 complet — 5 types, localStorage, modes étude/examen, timer"
  ```

---

## Self-Review du plan

**Spec coverage :**
- ✅ Types TF, multi, match, blank → Tasks 2-5
- ✅ Nouvelles questions issues du DOCX → Tasks 2-5 (questions intégrées dans les données)
- ✅ Explication auto si faux (mode Étude) → Task 6
- ✅ Mode Étude vs Examen → Task 6
- ✅ Timer global non bloquant → Task 6
- ✅ localStorage + ⭐ → Task 1 (loadLS/saveLS/toggleStar) + Tasks 2-5 (bouton ⭐)
- ✅ Filtre "À revoir" → Task 7
- ✅ Section résultats enrichie → Task 7
- ✅ Fichier HTML unique → architecture confirm

**Cohérence des types :**
- `buildStarBtn(id)` défini en Task 2, utilisé en Tasks 3-5 ✅
- `normalizeBlank()` défini en Task 5, utilisé dans `submitBlank()` ✅
- `formatTime()` défini en Task 6 ✅
- `state.ls` initialisé via `loadLS()` en Task 1, utilisé partout ✅
- `state.matchSel[qid].shuffledRight` : le shuffle est stocké dans matchSel la première fois ✅

**Placeholders :** Aucun TBD/TODO dans le plan ✅
