# Pattern de Documentation Architecture

Ce document décrit le pattern et le workflow pour documenter les mécanismes complexes du projet,
gérer la connaissance architecture de façon économique en contexte, et l'exploiter efficacement
dans toutes les tâches futures.

Il est le pendant de [MEMORY_ARCHITECTURE.md](MEMORY_ARCHITECTURE.md) pour la couche
documentation architecture (distincte de la couche mémoire projet).

---

## ⚠️ Prérequis — Câblage obligatoire dans `CLAUDE.md`

**Sans cette étape, les skills `architecture-document` et `architecture-reviewer` ne fonctionnent
pas comme attendu.** Claude ne sait pas que `architecture-map.md` existe et ne le consultera
jamais, même après plusieurs sessions documentées.

### Ce qui doit être présent dans `CLAUDE.md` racine

```markdown
## Mécanismes complexes documentés

@docs/ai/architecture-map.md

Si une tâche touche un mécanisme listé dans cette carte, lire le `summary.md` correspondant
AVANT de proposer quoi que ce soit. Le chemin est indiqué dans l'entrée du mécanisme.

Pour une analyse approfondie avant une modification complexe, utiliser le subagent :
Utilise l'architecture-reviewer. mechanism: <nom>. task: <description>
Spécification : `.claude/skills/architecture-reviewer/SKILL.md`

⚠️ `architecture-reviewer` = TOUJOURS via Agent tool (subagent isolé).
Ne jamais lire `full-analysis.md` directement dans le thread principal.
```

**Pourquoi l'instruction est impérative ("AVANT de proposer quoi que ce soit") :**
Une formulation suggestive ("consulter si besoin") laisse Claude décider de sauter la lecture.
Une formulation impérative garantit que `summary.md` est lu avant toute proposition — c'est
la différence entre ~70% et ~95% de fiabilité sur les mécanismes documentés.

**Pourquoi la règle Agent tool est dans `CLAUDE.md` :**
Claude sait qu'il doit spawner un subagent grâce à 3 signaux cumulés : la description du skill
dans le system-reminder ("Invoke via the Agent tool"), le contenu de `SKILL.md` chargé à
l'invocation, et cette ligne dans `CLAUDE.md`. Sans ce troisième signal, une formulation floue
("regarde la doc de X") peut conduire Claude à lire `full-analysis.md` directement dans le
thread principal — annulant le bénéfice d'isolation. La formulation canonique garantie :
`Utilise l'architecture-reviewer. mechanism: <nom>. task: <description>`.

### Pourquoi c'est critique

| Sans câblage | Avec câblage |
|-------------|-------------|
| Claude ignore `architecture-map.md` | Claude charge la carte à chaque session |
| Mécanismes documentés jamais consultés | Claude sait qu'il faut vérifier avant de toucher |
| `architecture-reviewer` jamais invoqué naturellement | Claude sait comment l'invoquer |
| Skills produisent des docs orphelins | Docs raccordés à la mémoire active |

### Vérification

Ce câblage est **fait une seule fois** à l'initialisation du pattern.
Pour vérifier qu'il est en place, lire `CLAUDE.md` racine et confirmer que la section
"Mécanismes complexes documentés" avec `@docs/ai/architecture-map.md` est présente.

### Ce qui NE doit pas être dans `CLAUDE.md`

```markdown
# INTERDIT — trop coûteux en contexte
@docs/ai/architecture/<mechanism>/full-analysis.md
@docs/ai/architecture/<mechanism>/summary.md   # sauf CLAUDE.md local, mécanisme HIGH risk
```

`CLAUDE.md` racine importe uniquement `architecture-map.md` (hub court).
Les `summary.md` individuels s'importent uniquement dans les CLAUDE.md locaux des modules
à risque HIGH — jamais à la racine.

---

## Pourquoi ce pattern ?

### Le problème

Une session de rétro-ingénierie produit 1500+ lignes de contenu précieux.
Si ce contenu est stocké dans un fichier unique et référencé directement depuis `CLAUDE.md`,
il est chargé dans chaque conversation — même celles qui n'en ont pas besoin.

Résultat : contexte pollué, signal dilué, coût élevé, Claude moins précis.

### La solution

```
Séparer archive et référence opérationnelle.
Ne charger que ce qui est utile à la tâche en cours.
Déléguer la lecture des archives à un subagent isolé.
```

**Principe central** :
> Mémoire active = résumés courts et règles d'action.
> Document long = archive d'expertise consultée à la demande.
> Subagent = lecteur spécialisé qui ne pollue pas le thread principal.

---

## La structure 3 fichiers

Pour chaque mécanisme complexe documenté :

```
docs/ai/architecture/
  <mechanism-name>/
    summary.md         → 50-100 lignes  — guide d'action opérationnel
    decision.md        → 100-200 lignes — contexte décisionnel
    full-analysis.md   → sans limite    — archive passive complète
```

### Ce que contient chaque fichier

#### `summary.md` — Le guide d'action

Répond à une seule question : *"Puis-je modifier ce mécanisme en toute sécurité ?"*

```markdown
## Rôle
## Architecture actuelle (schéma ou liste courte)
## Points sensibles (ce qui peut casser)
## Règles d'implémentation (ce qui doit toujours être respecté)
## Règles d'évolution (comment étendre sans risque)
## Références → full-analysis.md, decision.md
```

**Limite stricte : 100 lignes.** Si c'est plus long, c'est un document de compréhension,
pas un guide d'action — le raccourcir.

#### `decision.md` — Le contexte décisionnel

Répond à : *"Pourquoi a-t-on fait ce choix et quelles alternatives ont été écartées ?"*

```markdown
## Décision retenue
## Alternatives écartées (tableau avec raisons)
## Contraintes ayant guidé le choix
## Impact sur le code
## Règles dérivées de cette décision
```

Consulté quand une tâche remet en question l'architecture ou nécessite de comprendre le
raisonnement derrière les choix.

#### `full-analysis.md` — L'archive passive

Contient tout ce qui a été découvert, discuté, mesuré, comparé pendant la session
de rétro-ingénierie. Aucune limite de taille.

**Ne jamais @importer.** Consulté uniquement à la demande, via le subagent ou directement
quand l'analyse complète est nécessaire.

---

## Le hub de routage — `architecture-map.md`

`docs/ai/architecture-map.md` est l'index central de toutes les unités documentées.
C'est **le seul fichier architecture importé via @ dans `CLAUDE.md`**.

Structure d'une entrée :

```markdown
## <Name> [<type>]

**Type** : component | module | subsystem | bounded-context
**Périmètre** : [packages ou classes concernés]
**Risque** : HIGH | MEDIUM | LOW
**Résumé** : [une phrase décrivant le rôle]
**⚠️ Obligatoire** : LIRE `docs/ai/architecture/<name>/summary.md` avant toute modification.
**Règle** : [contrainte d'implémentation principale en une ligne]
**Analyse complète** : `docs/ai/architecture/<name>/full-analysis.md` (ne pas @importer)
```

Il reste court (~16 lignes par unité) même avec 10 unités documentées.
Claude lit l'entrée, connaît le type, et sait exactement où aller si la tâche touche cette unité.

---

## Les points d'injection dans la mémoire

### Ce qui doit être @importé (chargé à chaque session)

| Fichier | Via @ dans | Condition |
|---------|-----------|-----------|
| `docs/ai/architecture-map.md` | `CLAUDE.md` racine | Dès qu'un mécanisme est documenté |
| `docs/ai/architecture/<name>/summary.md` | CLAUDE.md local du module | Mécanisme HIGH risk uniquement |

### Ce qui ne doit jamais être @importé

| Fichier | Pourquoi |
|---------|---------|
| `full-analysis.md` | Trop long — charge tout le contexte inutilement |
| `decision.md` | Consulté ponctuellement, pas à chaque session |

### Instruction dans `CLAUDE.md` racine (à ajouter une seule fois)

```markdown
## Mécanismes complexes documentés
Avant de modifier un module à risque, consulter :
@docs/ai/architecture-map.md

Pour une analyse approfondie d'un mécanisme, utiliser le subagent :
architecture-reviewer (voir .claude/skills/architecture-reviewer/SKILL.md)
```

### Instruction dans un CLAUDE.md local (pour modules HIGH risk)

```markdown
## Mécanisme critique documenté

Ce module implémente un mécanisme complexe.
Avant toute modification :
@docs/ai/architecture/<mechanism>/summary.md

Analyse complète (ne pas @importer) :
docs/ai/architecture/<mechanism>/full-analysis.md
```

---

## Le workflow complet

### Phase 1 — Acquisition (3 modes)

Le skill supporte trois sources d'entrée. Choisir selon le contexte :

#### Phase 1a — Session de travail (source : conversation)

```
Développeur + Claude explorent le code ensemble.
Questions posées, réponses trouvées, patterns identifiés.
Tout reste dans la conversation — ne rien écrire encore.

→ /architecture-document <mechanism-name>
  Le skill extrait tout depuis la conversation.
```

#### Phase 1b — Document existant (source : fichier markdown)

```
Tu as un document existant : analyse technique, plan de migration,
étude d'évolution architecturale, RFC, notes de session passée, etc.

→ /architecture-document <name> type=<type> doc=<path>
  Le skill lit le fichier et en extrait le contenu.
```

Ce mode fonctionne pour deux familles de documents :

**Documents rétrospectifs** (analyse d'un existant) : analyses de code, post-mortems,
notes de session d'exploration. Le contenu alimente surtout `full-analysis.md` et `summary.md`.

**Documents prospectifs** (étude d'évolution, RFC) : nouvelle architecture cible,
comparaison de solutions, justification technique, notes de performance, stratégie
de développement avec priorisation. Le contenu alimente surtout `decision.md` et
les règles d'implémentation via PROMOTE-CANDIDATE.

**Lecture par section pour les documents ≥ 300 lignes** :

```
1. Le skill lit les 50 premières lignes → détecte la structure et les titres ##
2. Présente la liste des sections :
   "Ce document contient X sections (~N lignes). Lecture par section :
    § 1. Contexte (l.1-80)
    § 2. Architecture actuelle (l.81-250)
    § 3. Problèmes identifiés (l.251-480)
    ..."
3. Attend ta confirmation (tu peux exclure des sections)
4. Lit chaque section individuellement (offset + limit)
5. Classe chaque section dans le bon fichier de destination
6. Fusionne les buckets → extraction Step 3
```

**Routage par type de section :**

| Mots-clés dans le titre | Destination |
|------------------------|-------------|
| Architecture, Structure, Composants | `full-analysis` + candidats `summary` |
| Problèmes, Risques, Edge cases, Bugs | `full-analysis` + candidats `known-issues` |
| Options, Alternatives, Comparaison, POC, Benchmark | candidats `decision` |
| Solution retenue, Décision, Justification | `decision` + candidats `.claude/rules/` |
| Règles, Contraintes, Impératifs | `decision` + PROMOTE-CANDIDATE directs |
| Performance, Mesures, Métriques | `full-analysis` + candidats `decision` si comparatif |
| Stratégie, Priorisation, Roadmap, Étapes | `decision` + candidats `summary` |
| Historique, Contexte, Background | `full-analysis` archive |
| Code, Implémentation, Exemples | `full-analysis` + candidats `systemPatterns` |
| Vocabulaire, Termes métier, Glossaire | candidats `domain-glossary` |
| Intégration, API, Flux entre services | candidats `integration-patterns` |
| Sources de données, Transformations, Cibles | candidats `data-sources` |

**Pourquoi cette stratégie** : lire un document de 800 lignes d'un coup
surcharge le contexte pendant la génération. La lecture par section permet
de classer le contenu au fur et à mesure, sans risque de perte.

#### Phase 1c — Hybride (source : fichier + conversation)

```
Tu as un document préparé ET une session de validation/décision récente.

→ /architecture-document <mechanism-name> doc=<path>
  Le skill fusionne les deux sources.
  Règle : la conversation gagne en cas de conflit (plus récente).
```

Cas typique : plan de migration discuté et raffiné pendant la session.

### Phase 2 — Génération (commun aux 3 modes)

```
Le skill :
  1. Infère ou confirme le nom du mécanisme
  2. Lit : systemPatterns.md + architecture-map.md + activeContext.md
  3. Extrait et fusionne le contenu des sources actives
  4. Rédige les 3 fichiers + 2 patches (map + ADR)
  5. Liste les PROMOTE-CANDIDATE (règles pour .claude/rules/)
  6. Présente les drafts → attend confirmation
  7. Écrit uniquement après approbation
```

### Phase 3 — Validation humaine

```
Relire summary.md :
  → Est-ce que ça répond à "puis-je toucher ça ?" ?
  → Est-ce ≤ 100 lignes ?
  → Les règles d'implémentation sont-elles claires et actionnables ?

Relire decision.md :
  → Les alternatives écartées sont-elles documentées ?
  → Le raisonnement est-il reproductible ?

Vérifier les patches :
  → architecture-map.md : l'entrée est-elle correcte ?
  → architecture-decisions.md : l'ADR est-il bien numéroté ?
```

### Phase 4 — Promotion des règles d'action

```
# Même session que /architecture-document
/promote-to-memory

# Session ultérieure (PROMOTE-CANDIDATE plus dans la conversation)
/promote-to-memory doc=docs/ai/architecture/<name>/full-analysis.md
```

`/promote-to-memory` détecte les PROMOTE-CANDIDATE en priorité — via la conversation
ou via la section `## PROMOTE-CANDIDATE` du fichier `full-analysis.md`. Ces items sont
traités comme des candidats confirmés — le filtre durable/non-durable ne leur est pas
réappliqué.

Avant chaque promotion, déduplication automatique contre les fichiers cibles existants.

**Format des PROMOTE-CANDIDATE** (persisté dans `full-analysis.md`) :
```
- "<item>"  → <destination>  [high | medium | low — <brief reason>]
```

Le niveau de confiance est préservé inter-sessions. Les items `[low]` (prospectifs ou
indirects) sont signalés à l'humain au moment du patch — ils ne sont pas automatiquement
rejetés mais nécessitent une attention particulière lors de la validation.

> **⚠️ Gap 3 — Confidence sans grep** : `/architecture-document` ne grepp pas le code.
> Les PROMOTE-CANDIDATE qu'il génère lui-même sont au mieux `[medium]` sauf si le document
> source cite des preuves code explicites. Les items issus d'un export `codebase-analysis`
> via `doc=` conservent leur niveau d'origine (basé sur des preuves code réelles).

Destinations complètes :
  → Invariant repo-wide          → .claude/rules/
  → Invariant subtree            → local CLAUDE.md
  → Décision cross-cutting       → docs/ai/architecture-decisions.md
  → Contrainte opérationnelle    → docs/ai/known-issues.md
  → Pattern récurrent (2+)       → .claude/memory-bank/systemPatterns.md
  → Terme métier                 → docs/ai/domain-glossary.md
  → Pattern d'intégration        → docs/ai/integration-patterns.md
  → Flux de données              → docs/ai/data-sources.md

NE PAS promouvoir :
  → Contenu déjà dans decision.md (duplication)
  → Contenu déjà dans architecture-decisions.md
  → Observations ponctuelles sans portée générale

**Pourquoi les PROMOTE-CANDIDATE sont persistés dans `full-analysis.md` :**
La conversation disparaît à la fermeture de session. Sans persistance fichier,
les PROMOTE-CANDIDATE sont perdus si `/promote-to-memory` n'est pas lancé immédiatement.
`full-analysis.md` est le point de handoff inter-sessions.

Pipeline complet :
```
/architecture-document  →  Step 4 identifie PROMOTE-CANDIDATE
                        →  Step 6 les écrit dans full-analysis.md (## PROMOTE-CANDIDATE)
                        →  les affiche aussi en conversation (Step 5)
                                    ↓
/promote-to-memory      →  Source A : ## PROMOTE-CANDIDATE de full-analysis.md (si doc=)
                        →  Source B : conversation (si même session)
                        →  déduplication contre .claude/rules/ existants
                        →  approval gate → écriture
```

> **⚠️ Gap 4 — progress.md et activeContext.md non mis à jour automatiquement** :
> `/architecture-document` n'écrit jamais dans ces fichiers. Si la documentation fait partie
> d'un chantier en cours, mettre à jour `activeContext.md` manuellement. Si elle clôt un
> jalon, le déplacer de `activeContext.md` vers `progress.md`. Ces mises à jour peuvent aussi
> être incluses dans le passage suivant à `/promote-to-memory`.

### Phase 5 — Câblage (selon niveau de risque)

```
Risque LOW   → Aucun câblage supplémentaire.
               L'entrée dans architecture-map.md suffit.

Risque MEDIUM → Vérifier que CLAUDE.md racine @importe architecture-map.md.

Risque HIGH  → Créer/compléter un CLAUDE.md local dans le module.
               @importer summary.md (pas full-analysis).
```

### Phase 6 — Utilisation dans les tâches futures

```
Tâche simple sur mécanisme connu
  → Claude lit architecture-map.md (@importé)
  → Suit les règles de l'entrée
  → Pas de consultation supplémentaire

Tâche complexe ou risquée
  → "Utilise l'architecture-reviewer. mechanism: X. task: Y."
  → Subagent lit summary.md dans contexte isolé
  → Si insuffisant : lit full-analysis.md (sections pertinentes)
  → Retourne recommandation 20 lignes au thread principal
  → Thread principal continue sans avoir chargé 1500 lignes
```

---

## Les deux outils

### `/architecture-document` — Le générateur

Invoqué à la fin d'une session d'analyse ou pour convertir un document existant.
Produit les 3 fichiers et les patches.

```bash
# Source : conversation uniquement
/architecture-document generator-pipeline
/architecture-document                          # infère le nom
/architecture-document order-cache lang=en

# Source : fichier markdown uniquement
/architecture-document doc=docs/PRODUCT_PERSISTENCE_ANALYSIS.md
/architecture-document product-persistence doc=docs/PRODUCT_PERSISTENCE_ANALYSIS.md

# Sources combinées : fichier + conversation
/architecture-document product-sync-pipeline doc=docs/PRODUCT_SYNC_MIGRATION_PLAN.md
```

Pour les documents ≥ 300 lignes, le skill applique automatiquement la stratégie
de lecture par section (présentation de la liste des sections → confirmation → lecture
section par section → classification → fusion).

Documentation : `.claude/skills/architecture-document/references/user-manual.md`

### `architecture-reviewer` — Le lecteur isolé

Invoqué avant une tâche d'implémentation complexe. Lit les docs dans un contexte séparé.

⚠️ **Toujours via Agent tool (subagent isolé).** Ne jamais lire `full-analysis.md`
directement dans le thread principal — cela annule le bénéfice d'isolation.

```
Utilise l'architecture-reviewer. mechanism: product-sync-pipeline. task: Ajouter un nouveau type de source sans casser la validation.
```

**Pourquoi cette formulation canonique** : Claude reconnaît le reviewer via 3 signaux
(description skill, SKILL.md, CLAUDE.md racine). Une formulation floue comme "regarde la
doc de X" peut conduire Claude à lire les fichiers directement sans spawner de subagent.
La formulation `Utilise l'architecture-reviewer. mechanism: X. task: Y.` garantit l'isolation.

Documentation : `.claude/skills/architecture-reviewer/references/user-manual.md`

---

## Intégration avec le système mémoire existant

Ce pattern s'intègre dans les 4 couches de mémoire définies dans
[MEMORY_ARCHITECTURE.md](MEMORY_ARCHITECTURE.md) :

| Couche mémoire | Rôle dans ce pattern |
|---------------|----------------------|
| Layer 1 — Règles (`CLAUDE.md`, `.claude/rules/`) | Reçoit les règles promues depuis PROMOTE-CANDIDATE |
| Layer 2 — Connaissance (`docs/ai/`) | Contient les 3 fichiers + architecture-map.md + ADRs |
| Layer 3 — Contexte (`memory-bank/`) | `systemPatterns.md` reçoit les patterns promus |
| Layer 4 — Auto-memory | Source de détection pour `/promote-to-memory` |

**La couche architecture (`docs/ai/architecture/`) est une extension de Layer 2.**
Elle suit les mêmes règles : durable, explicite, révisable comme du code.

### Ce qui va où après une session

**Écrits directement par le skill (Step 6) :**

| Fichier | Remarque |
|---------|---------|
| `docs/ai/architecture/<name>/full-analysis.md` | Archive passive — ne pas @importer |
| `docs/ai/architecture/<name>/summary.md` | Importable si mécanisme HIGH risk |
| `docs/ai/architecture/<name>/decision.md` | Passif — inclut les politiques métier du mécanisme |
| `docs/ai/architecture-map.md` | @importé depuis `CLAUDE.md` racine |
| `docs/ai/architecture-decisions.md` | ADR court |
| `CLAUDE.md` racine | Bootstrap uniquement (une seule fois) — ajout `@architecture-map.md` |

> **`CLAUDE.md` racine n'est pas une destination PROMOTE-CANDIDATE.** Les règles repo-wide
> vont dans `.claude/rules/`. `CLAUDE.md` est un hub d'import, pas un fichier de règles.

**Via PROMOTE-CANDIDATE (promotion différée, via `/promote-to-memory`) :**

| Destination | Type d'item |
|-------------|-------------|
| `.claude/rules/` | Invariant repo-wide |
| Local `CLAUDE.md` | Invariant subtree-specific |
| `docs/ai/architecture-decisions.md` | Décision cross-cutting (en plus de l'ADR) |
| `docs/ai/known-issues.md` | Contrainte opérationnelle |
| `.claude/memory-bank/systemPatterns.md` | Pattern récurrent (2+) |
| `docs/ai/domain-glossary.md` | Terme métier — Tier 2 on-demand |
| `docs/ai/integration-patterns.md` | Pattern d'intégration — Tier 2 on-demand |
| `docs/ai/data-sources.md` | Flux de données — Tier 2 on-demand |

---

## Ce que le système apporte une fois un mécanisme documenté

Dès qu'un mécanisme est documenté et câblé, Claude dispose d'un contexte persistant pour
toutes les tâches futures le concernant — sans que tu aies à le réexpliquer.

### Capacités activées par tâche

| Type de tâche | Ce que Claude peut faire | Condition |
|---------------|--------------------------|-----------|
| **Fix simple** | Respecte les invariants, identifie les points sensibles | Automatique via map + summary |
| **Évolution MEDIUM** | Propose une approche compatible avec l'architecture existante | Automatique via map + summary |
| **Évolution HIGH** | Analyse les contraintes, vérifie la compatibilité avec decision.md | Reviewer explicite |
| **Décision architecturale** | S'appuie sur les alternatives déjà écartées, évite de reparcourir le même raisonnement | Reviewer + decision.md |
| **Rétro-ingénierie** | Repart de full-analysis.md au lieu de zéro — accélère la compréhension | À la demande |
| **Refactoring safe** | Connaît les invariants à préserver, les contrats à respecter | Automatique via summary |
| **Revue de code** | Vérifie la conformité aux règles d'implémentation documentées | Automatique via summary |

### Ce que tu ne feras plus jamais

Une fois documenté, tu n'as plus à :
- réexpliquer comment fonctionne le mécanisme à chaque session
- rappeler les contraintes à respecter avant chaque modification
- redécouvrir les edge cases déjà analysés
- reconstruire le raisonnement derrière les choix architecturaux

### Les trois limites à connaître

**1. La reconnaissance dépend du nommage**
Si la tâche dit "modifie le client CatalogSync" mais que la carte utilise `product-sync-pipeline`,
Claude peut ne pas faire le lien. La carte doit lister les packages et classes concernés
précisément. En cas de doute, mentionner explicitement le nom du mécanisme dans la tâche.

**2. Le reviewer n'est pas automatique pour les tâches complexes**
Pour les modifications simples : `summary.md` est suffisant et chargé automatiquement.
Pour les modifications complexes : **invoquer explicitement le reviewer** — Claude ne spawne
pas un subagent de sa propre initiative.

**3. Les docs vieillissent si le code évolue sans mise à jour**
Un `summary.md` obsolète est **activement dangereux** : Claude travaillera sur une base
incorrecte avec une confiance injustifiée. Voir la section suivante.

---

## ⚠️ Maintenir la documentation synchronisée avec le code

**C'est la règle la plus importante de tout ce pattern.**

Un document d'architecture qui ne reflète plus le code réel est pire que l'absence de
documentation. Claude lui fera confiance et proposera des modifications basées sur une
réalité qui n'existe plus.

> **Principe** : les docs architecture sont du code. Ils évoluent avec lui, sont révisés
> comme lui, et leur obsolescence est un bug.

### Quand mettre à jour obligatoirement

| Événement dans le code | Fichier(s) à mettre à jour | Urgence |
|------------------------|---------------------------|---------|
| Refactoring d'un mécanisme documenté | `summary.md` + entrée `architecture-map.md` | **Immédiate** |
| Changement d'une règle d'implémentation | `summary.md` + `.claude/rules/` concerné | **Immédiate** |
| Ajout d'un nouveau composant dans le périmètre | `summary.md` section "Architecture actuelle" | Avant prochaine PR |
| Décision architecturale révisée | `decision.md` + `architecture-decisions.md` | Avant prochaine PR |
| Nouveau pattern émergent (2+ occurrences) | `full-analysis.md` + `systemPatterns.md` | Fin de session |
| Bug découvert lié au mécanisme | `full-analysis.md` section edge cases + `known-issues.md` | Fin de session |
| Contrainte externe modifiée (API, DB, infra) | `full-analysis.md` + `summary.md` si impact sur règles | Avant prochaine PR |

### Hiérarchie de criticité des mises à jour

```
CRITIQUE — mettre à jour dans la même PR que le code :
  summary.md              → règles d'implémentation modifiées
  architecture-map.md     → périmètre ou risque modifié
  .claude/rules/          → invariants modifiés

IMPORTANT — mettre à jour avant la PR suivante :
  decision.md             → décision révisée
  architecture-decisions.md → nouvel ADR

UTILE — mettre à jour en fin de session :
  full-analysis.md        → nouvelles découvertes, edge cases
  systemPatterns.md       → nouveaux patterns confirmés
  known-issues.md         → nouveaux risques identifiés
```

### Comment détecter l'obsolescence

**Signal humain (le plus fiable)** :
Claude propose quelque chose qui contredit ce que tu sais du code →
`summary.md` ou les règles sont obsolètes. Mettre à jour immédiatement.

**Signal Claude** :
Pendant une tâche, si Claude dit "selon la documentation, X devrait fonctionner ainsi"
mais que tu sais que X a changé → arrêter, corriger la doc, reprendre.

**Signal `memory-maintainer`** (audit périodique) :
Toutes les 2-4 semaines, l'audit compare les fichiers mémoire avec le code réel.
Pour les fichiers architecture, ajouter explicitement dans la demande d'audit :

```text
mode: audit
scope: targeted
paths:
- docs/ai/architecture-map.md
- docs/ai/architecture/<mechanism>/summary.md
notes: Vérifier la cohérence avec le code actuel du mécanisme.
```

**Signal git** :
Après un merge de PR touchant un mécanisme documenté, vérifier si les fichiers
architecture concernés ont été mis à jour dans le même commit ou dans une PR de suivi.

### Workflow de mise à jour

```
Code modifié dans un mécanisme documenté
  │
  ├─ Changement mineur (ajout d'une classe, refactor interne)
  │    → Éditer summary.md directement
  │    → Mettre à jour l'entrée architecture-map.md si périmètre change
  │
  ├─ Changement significatif (nouvelle règle, nouveau pattern)
  │    → Éditer summary.md + decision.md si nécessaire
  │    → /promote-to-memory pour les nouvelles règles → .claude/rules/
  │    → memory-maintainer pour valider la cohérence
  │
  └─ Refactoring majeur (architecture révisée)
       → Nouvelle session d'analyse si nécessaire
       → /architecture-document <mechanism> pour régénérer
       → Garder full-analysis.md ancien en archive (renommer avec date)
       → Nouveau full-analysis.md pour l'état actuel
```

### Ce qui NE doit PAS être mis à jour

`full-analysis.md` est une **archive** — ne pas réécrire l'historique.
Si l'architecture change significativement, créer un nouveau `full-analysis.md` et
archiver l'ancien (ex: `full-analysis-2025-03.md`). L'historique des raisonnements
a de la valeur.

---

## Niveaux de complexité des évolutions

Le reviewer adapte sa profondeur d'analyse selon la complexité annoncée de la tâche :

| Complexité | Comportement du reviewer | Documentation minimale nécessaire |
|------------|--------------------------|-----------------------------------|
| Simple | Lit summary.md uniquement | summary.md |
| Moyenne | Lit summary.md + sections ciblées de full-analysis | summary.md + full-analysis.md |
| Complexe | Lit summary.md + decision.md + full-analysis.md complet | Les 3 fichiers |
| Très complexe | Comme complexe + recommande une nouvelle session d'analyse | Les 3 fichiers + nouvelle session |

---

## Signaux d'alerte

### Signaux de structure (câblage et format)

| Signal | Cause | Action |
|--------|-------|--------|
| `summary.md` > 100 lignes | Pas assez ciblé | Pruner — enlever ce qui appartient à `full-analysis.md` |
| `architecture-map.md` non @importé dans `CLAUDE.md` | Câblage manquant | Ajouter l'instruction dans `CLAUDE.md` |
| PROMOTE-CANDIDATE non promus après génération | Règles restent dans docs passifs | Exécuter `/promote-to-memory` |
| Mécanisme HIGH risk sans `CLAUDE.md` local | Claude ne charge pas `summary.md` automatiquement | Créer le `CLAUDE.md` local |

### Signaux d'obsolescence (⚠️ les plus dangereux)

| Signal | Cause | Action |
|--------|-------|--------|
| Claude propose quelque chose qui contredit le code réel | `summary.md` obsolète | Mettre à jour immédiatement avant de continuer |
| PR mergée sans mise à jour des docs architecture | Discipline manquante | Mettre à jour dans la PR de suivi |
| Classe ou package listé dans la carte qui n'existe plus | Refactoring non documenté | Mettre à jour `architecture-map.md` + `summary.md` |
| Règle dans `summary.md` qui ne s'applique plus | Code a évolué sans mise à jour | Corriger `summary.md` + vérifier `.claude/rules/` |
| `decision.md` décrit une alternative "écartée" qui a finalement été implémentée | Décision révisée non tracée | Mettre à jour `decision.md` + nouvel ADR |
| `full-analysis.md` décrit un comportement contredit par les tests actuels | Architecture changée | Créer un nouveau `full-analysis.md`, archiver l'ancien avec date |
| `memory-maintainer` audit détecte des classes inexistantes dans les docs | Docs ont précédé ou suivi un refactoring | Lancer `/architecture-document` pour régénérer |

---

## Annexe A — Performance et économie de contexte

### Hypothèses de base

```
full-analysis.md  : ~800 lignes  → ~16 000 tokens
summary.md        : ~75 lignes   → ~1 500 tokens
decision.md       : ~150 lignes  → ~3 000 tokens
entrée map        : ~15 lignes   → ~300 tokens
réponse reviewer  : 20 lignes    → ~400 tokens
```

### Comparaison des scénarios (par session)

| Scénario | Tokens dans le thread principal | Notes |
|----------|---------------------------------|-------|
| **Sans système** — développeur réexplique le contexte | ~3 000 | Risque d'oubli d'invariants |
| **Anti-pattern** — `@full-analysis.md` dans CLAUDE.md | ~16 000 par mécanisme | Chargé même quand inutile |
| **Anti-pattern × 5 mécanismes** | ~80 000 (fixe) | 40% du contexte Sonnet saturé |
| **Avec le système** — map + reviewer | ~1 900 | 1 500 (map) + 400 (reviewer) |

### Réduction mesurée

| Comparaison | Réduction tokens | Réduction contexte |
|-------------|------------------|--------------------|
| vs anti-pattern @import (1 mécanisme) | −14 100 tokens | **−88%** |
| vs anti-pattern @import (5 mécanismes) | −78 100 tokens | **−97.6%** |
| vs réexplication manuelle | −1 100 tokens | **−37%** mais couverture complète |

### Comportement à l'échelle

```
Anti-pattern : coût ∝ nombre de mécanismes → contexte saturé à 10-12 mécanismes
Avec le système : coût fixe (map) + coût à la demande (reviewer)
→ le coût de la mémoire architecture reste constant quelle que soit la croissance
```

À 10 mécanismes documentés :
- Anti-pattern : ~160 000 tokens fixes → impossibilité pratique sur Sonnet (200K)
- Avec le système : ~3 000 tokens fixes + reviewer ponctuel

### Synthèse

| Axe | Score | Détail |
|-----|-------|--------|
| Vision | ★★★★★ | 3 niveaux d'accès gradués — rien n'est perdu |
| Isolation contexte | ★★★★★ | −97% vs anti-pattern, stable à l'échelle |
| Robustesse | ★★★★☆ | 1 point faible résiduel (obsolescence = discipline humaine) |
| Déterminisme | ★★★★☆ | ~95% sur le reviewer avec formulation canonique |

---

## Annexe B — Points d'amélioration à venir

### Failles résiduelles connues

**Faille 1 — Nommage (risque MEDIUM)**

Si la tâche utilise un terme différent du nom dans la carte ("modifie le client CatalogSync"
vs entrée `product-sync-pipeline`), Claude peut ne pas faire le lien et ne pas consulter
`summary.md`.

Atténuation actuelle : lister les packages et classes concernés dans l'entrée de la carte.
Amélioration possible : ajouter un champ `aliases:` dans le format d'entrée de la carte.

**Faille 2 — Reviewer non automatique (risque LOW)**

Pour les tâches complexes, Claude ne spawne pas le reviewer de sa propre initiative — le
développeur doit utiliser la formulation canonique. Si la demande est floue ("regarde la doc
de X"), Claude peut lire les fichiers directement sans subagent.

Atténuation actuelle : 3 signaux cumulés (system-reminder + SKILL.md + CLAUDE.md).
Amélioration possible : ajouter dans l'entrée de la carte un champ `reviewer: required` pour
les mécanismes HIGH, que Claude pourrait interpréter comme un déclencheur automatique.

**Faille 3 — Obsolescence silencieuse (risque HIGH si discipline absente)**

Un `summary.md` qui ne reflète plus le code est activement dangereux : Claude travaille
avec une confiance injustifiée sur une réalité qui n'existe plus.

Atténuation actuelle : section "Maintenir la documentation synchronisée" + signaux d'alerte.
Amélioration possible : ajouter dans les CLAUDE.md locaux des modules HIGH risk une ligne
indiquant la date de dernière mise à jour de `summary.md`, visible à chaque session :

```markdown
> ⚠️ Dernière mise à jour summary.md : 2026-01-15 — vérifier si toujours à jour.
```

### Évolutions envisageables

| Évolution | Valeur | Complexité |
|-----------|--------|------------|
| Champ `aliases` dans les entrées de la carte | Résout la faille de nommage | Faible |
| Champ `reviewer: required` pour HIGH risk | Rend le reviewer semi-automatique | Moyenne |
| Date de mise à jour dans CLAUDE.md local | Détecte l'obsolescence silencieuse | Faible |
| Audit périodique `memory-maintainer` scope architecture | Détecte les classes inexistantes | Déjà possible |
