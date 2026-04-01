# Pattern de Rétro-ingénierie

Ce document décrit le workflow de rétro-ingénierie d'un codebase existant, les décisions
de design derrière `codebase-analysis`, et son articulation avec le système de documentation
architecture.

Il est le pendant de [ARCHITECTURE_DOCUMENTATION_PATTERN.md](ARCHITECTURE_DOCUMENTATION_PATTERN.md)
pour la phase d'acquisition — celle qui précède la documentation.

---

## Pourquoi ce pattern ?

### Le problème

Comprendre un module inconnu avant de le modifier prend du temps et produit une connaissance
éphémère : elle reste dans la tête du développeur ou dans la conversation, puis disparaît.

La session suivante repart de zéro. Les edge cases redécouverts. Les mêmes erreurs commises.

### La solution

```
Séparer l'acquisition de la connaissance de sa capitalisation.
Adapter la profondeur d'analyse à la taille du périmètre.
Persister le plan d'analyse entre sessions.
Laisser l'utilisateur décider quoi mémoriser.
```

**Principe central** :
> Analyser = explorer et comprendre.
> Documenter = décider de ce qui mérite d'être mémorisé.
> Ces deux actes sont distincts et ne doivent pas être forcés ensemble.

---

## Où `codebase-analysis` se situe dans l'écosystème

```
codebase-analysis          →  /architecture-document     →  /promote-to-memory
(explorer et comprendre)      (capturer en mémoire)          (promouvoir les règles)
        ↓                              ↓                              ↓
  analysis-plan-*.md        docs/ai/architecture/<name>/      .claude/rules/
  (traçabilité sessions)    full-analysis + summary + decision  systemPatterns.md
        ↓                      (## PROMOTE-CANDIDATE inclus)
  doc=<file> (option 2)              ↓
  ## PROMOTE-CANDIDATE       architecture-map.md
  (dans le fichier,                  ↓
   avec niveau de confiance
   high | medium | low)
        ↑                   architecture-reviewer
        └── /codebase-analysis doc=  (exploiter en tâches futures)
            (approfondissement ciblé)
        ↓
  /promote-to-memory doc=<file>
  (session ultérieure)
```

`codebase-analysis` est la **Phase 0 du workflow de documentation architecture**. Sans lui,
`/architecture-document` peut être lancé directement depuis une session exploratoire manuelle.
Avec lui, l'exploration est structurée, tracée, et reproductible.

---

## Quel outil utiliser ?

| Situation | Outil |
|-----------|-------|
| Unité inconnue, jamais analysée | `codebase-analysis` |
| Unité déjà dans `architecture-map.md` | `architecture-reviewer` |
| Session exploratoire libre, sans structure | Conversation directe + `/architecture-document` |
| Convertir un document d'analyse existant | `/architecture-document doc=<path>` |
| Comprendre une règle ou contrainte existante | `.claude/rules/` + `systemPatterns.md` |

**Règle de routage** : si l'unité est dans `architecture-map.md`, ne pas relancer
`codebase-analysis` — l'analyse a déjà été faite. Utiliser `architecture-reviewer`
pour une tâche ciblée, ou relire `full-analysis.md` pour une révision approfondie.

---

## Les décisions de design du skill

### Pourquoi l'output gate existe

`codebase-analysis` ne sait pas ce que l'utilisateur veut faire avec les résultats.
Parfois on analyse pour comprendre, pas pour documenter. Imposer `/architecture-document`
à la fin serait un anti-pattern — il forcerait une décision de mémorisation sur une
exploration qui n'en mérite peut-être pas.

L'output gate préserve la séparation des responsabilités :
- Le skill analyse
- L'utilisateur décide
- `/architecture-document` mémorise si demandé

### Pourquoi 1 conversation par unité sur les grands repos

Le contexte Claude s'accumule. Analyser 4 unités dans la même conversation signifie
que la 4ème session d'analyse se fait avec les 3 premières toujours présentes en contexte
— même compactées, elles génèrent du bruit et réduisent la précision.

Une conversation par unité garantit un contexte propre et focalisé. Les résultats
des sessions précédentes sont accessibles via `architecture-map.md` (si option 1 choisie)
— pas via le contexte de conversation.

### Pourquoi le plan est un fichier externe

La mémoire Claude ne persiste pas entre conversations. Un plan stocké uniquement dans
la conversation disparaît à la fermeture. Le fichier `analysis-plan-<repo>.md` est la
seule source de vérité fiable entre sessions — il est versionnable, lisible par un humain,
et interrogeable par Claude sans mémoire préalable.

### Pourquoi la boucle de review précède l'output gate

Les phases 1-5 produisent une analyse brute — Claude lit le code mais ne sait pas
si l'utilisateur veut approfondir un aspect, corriger une interprétation, ou passer
directement à la capitalisation.

Présenter l'analyse dans la conversation avant de proposer l'output gate permet :
- de corriger des erreurs d'interprétation avant qu'elles soient persistées
- d'approfondir un aspect ciblé sans refaire toute l'analyse
- de valider que la qualité est suffisante pour mémorisation

**La règle** : l'output gate n'est jamais proposé sans approbation explicite de l'utilisateur.

### Pourquoi le mode `doc=` (raffinage itératif)

Une analyse complète d'une unité complexe peut nécessiter plusieurs sessions.
Le mode `doc=` permet de reprendre une analyse exportée (option 2) dans une nouvelle
conversation, de cibler les zones manquantes, puis de mettre à jour le même fichier.

Ce pattern découple deux décisions indépendantes :
- Quand est-ce que l'analyse est assez complète ? (décision progressive)
- Quand est-ce qu'elle mérite d'être mémorisée ? (décision de capitalisation)

Sans `doc=`, il faudrait soit tout refaire, soit choisir option 1 prématurément.

### Pourquoi les paramètres d'analyse sont persistés dans le plan

L'objectif d'analyse (débugger, modifier, comprendre, refactoriser, documenter) détermine
quelles phases sont prioritaires et à quelle profondeur. Le scope et le contexte connu
déterminent ce que Claude cherche en premier.

Sans persistance, une nouvelle session repart de zéro : Claude pose les questions à nouveau,
l'utilisateur réexplique, et le risque est fort que la reprise ne soit pas calibrée de la
même façon — surtout si plusieurs jours séparent les sessions.

Le fichier plan stocke ces paramètres dans `## Détails d'analyse` :
scope, liberté (exclusif / indicatif), objectif, contexte connu.

Au démarrage de la session suivante, Claude affiche les paramètres stockés et demande
"Confirmer ou modifier ?" — ce qui prend 5 secondes. Aucune information n'est perdue,
et l'utilisateur peut ajuster si le contexte a changé (nouvelle décision, bug identifié
entre-temps, objectif révisé).

L'utilisateur peut aussi modifier ces paramètres **à tout instant** pendant l'analyse —
avant, pendant ou après les phases — et les changements sont immédiatement écrits dans le
plan et appliqués aux phases restantes.

### Pourquoi `🚫 ignoré` est un statut persistant

Certaines unités ne méritent pas d'analyse (thin layers, config boilerplate, code généré).
Sans statut persistant, le skill les repropose à chaque session. `🚫 ignoré` est une
décision explicite et durable — elle exprime "nous avons choisi de ne pas documenter ça",
ce qui est une information architecturale en soi.

---

## PROMOTE-CANDIDATE — routing et niveau de confiance

Chaque PROMOTE-CANDIDATE identifié pendant l'analyse porte une destination et un niveau
de confiance obligatoire. Le niveau est déterminé par le mécanisme de détection, pas par
l'importance métier de l'item.

### Table de routing complète

| Type détecté | Phase | Destination | Confiance typique |
|---|---|---|---|
| Invariant repo-wide (extrait d'un `throw` ou grep 2+ fichiers) | 3 | `.claude/rules/` | `high` |
| Invariant subtree-specific (module unique) | 3 | Local `CLAUDE.md` (via registre) | `high` |
| Politique métier liée au mécanisme analysé | 3 | `decision.md` (via `/architecture-document`) | `medium` |
| Décision cross-cutting ou mécanisme déjà documenté | 3, 4 | `docs/ai/architecture-decisions.md` | `medium` |
| Contrainte opérationnelle (DB, infra, système externe) | 3 | `docs/ai/known-issues.md` | `high` ou `medium` |
| Pattern récurrent (2+ fichiers distincts) | 2, 3 | `.claude/memory-bank/systemPatterns.md` | `high` |
| Terme métier (vocabulaire sans équivalent dans le code) | 2 | `docs/ai/domain-glossary.md` | `medium` |
| Pattern d'intégration partagé (2+ unités) | 4 | `docs/ai/integration-patterns.md` | `medium` |
| Flux de données (source, transformation, cible) | 4 | `docs/ai/data-sources.md` | `medium` |
| Règle implicite `[IMPLICIT]` (suspicion depuis commentaire) | 3 | Destination la plus proche | `low` |

### Niveaux de confiance

| Niveau | Critère |
|--------|---------|
| `high` | Extrait d'un `throw`/exception, ou grep-confirmé dans 2+ fichiers distincts, ou coverage gap trouvé par grep |
| `medium` | Observation cohérente mais jugement de Claude (termes métier, patterns heuristiques, intégration vue sur une seule unité) |
| `low` | `[IMPLICIT]` — suspicion depuis un commentaire, convention tacite, ou preuve indirecte |

**Format** : chaque entrée dans `## PROMOTE-CANDIDATE` doit avoir la forme :
```
- "<item>"  → <destination>  [high | medium | low — <brief reason>]
```

**Conséquence sur la promotion** : `/promote-to-memory` et `memory-maintainer` voient le niveau
de confiance persisté dans `full-analysis.md`. Les items `[low]` doivent être vérifiés dans le
code avant que le patch soit appliqué — ils ne doivent pas être traités comme des faits établis.

---

## Le workflow complet

### Petit repo (< 5k lignes)

```
Session unique :
  /codebase-analysis
    Phase 0 : cartographie → confirmation
    Phases 1-5 : analyse en une passe
    Boucle de review → "c'est bon"
    Output gate → option 1 recommandée
    /architecture-document <name> type=<type>
    /promote-to-memory
```

### Moyen repo (5k–30k lignes)

```
Session unique ou deux :
  /codebase-analysis
    Phase 0 : cartographie → plan sauvegardé
    Phases 1-5 par couche (domain → services → adapters)
    Boucle de review par unité → validation → output gate
    Output gate par unité → /architecture-document pour chaque
    /promote-to-memory en fin de session
```

### Grand repo (> 30k lignes)

```
Conversation 1 :
  /codebase-analysis
    Phase 0 : cartographie globale → plan sauvegardé
    Analyse unité A [HIGH]
    Boucle de review → "ok"
    Output gate → option 1 → /architecture-document unit-a type=<type>
    Plan mis à jour : unit-a ✅

Conversation 2 :
  /codebase-analysis  (plan détecté automatiquement)
    Reprend avec unité B [HIGH]
    → Affiche paramètres stockés (objectif, scope, contexte) → "Confirmer ou modifier ?"
    Phases 1-5 calibrées selon l'objectif
    Boucle de review → "ok"
    Output gate → option 1 → /architecture-document unit-b type=<type>
    Plan mis à jour : unit-b ✅

...

Dernière conversation :
  Passe transversale (transactions, sécurité, flux end-to-end)
  /architecture-document cross-cutting-concerns type=subsystem (si warranté)
  /promote-to-memory global
```

### Analyse itérative sur plusieurs sessions (via doc=)

Pour les unités complexes nécessitant plusieurs passes avant capitalisation :

```
Session 1 — exploration initiale :
  /codebase-analysis
    Phases 1-5 → boucle de review → output gate option 2
    → docs/notes/my-unit-analysis.md créé
      (inclut ## PROMOTE-CANDIDATE — persistants entre sessions)

Session 2 — approfondissement :
  /codebase-analysis doc=docs/notes/my-unit-analysis.md
    → Résumé couverture (zones ✅ / ⚠️ / ❌)
    → Approfondissement ciblé sur zones manquantes
    → Boucle de review → output gate option 2
    → docs/notes/my-unit-analysis.md mis à jour

Session 3 — capitalisation :
  /codebase-analysis doc=docs/notes/my-unit-analysis.md
    → "continue" → boucle de review → "c'est bon"
    → Output gate option 1
    → /architecture-document my-unit type=<type>
```

Ce pattern est adapté quand l'analyse complète dépasse le scope d'une conversation,
ou quand la qualité doit être validée progressivement avant mémorisation.

---

## Ce que le système apporte une fois l'analyse terminée

Une fois une unité analysée ET documentée (option 1) :

| Capacité activée | Via |
|-----------------|-----|
| Claude connaît les invariants avant toute modification | `architecture-map.md` @importé |
| Modification sûre sans réexpliquer le contexte | `summary.md` lu sur déclenchement |
| Décisions architecturales traçables | `decision.md` + ADR |
| Règles d'implémentation actives à chaque session | `.claude/rules/` |
| Analyse approfondie avant tâche complexe | `architecture-reviewer` |

Si l'option 2 ou 3 a été choisie, la connaissance n'est pas intégrée au système —
elle reste dans les artefacts exportés ou disparaît. C'est un choix valide pour une
analyse ponctuelle, mais sans les bénéfices ci-dessus.

---

## ⚠️ Maintenir le plan synchronisé

Le fichier `analysis-plan-<repo>.md` est mis à jour automatiquement après chaque output gate.
Deux situations nécessitent une intervention manuelle :

**1. Une unité a été refactorisée et renommée**
→ Mettre à jour le nom dans le plan ET dans `architecture-map.md`

**2. Une unité `✅ documenté` a été significativement modifiée**
→ Remettre en `⬜ à faire` dans le plan
→ Relancer `codebase-analysis` sur cette unité
→ Relancer `/architecture-document` pour régénérer

---

## Signaux d'alerte

| Signal | Cause | Action |
|--------|-------|--------|
| Plan absent sur un grand repo | Sessions impossibles à reprendre | Créer le plan rétroactivement |
| Paramètres stockés devenus obsolètes (objectif a changé) | Décision prise entre sessions | Dire "Change l'objectif de <name> en <nouveau>" — plan mis à jour immédiatement |
| Unités toutes en `💬 conversation only` | Analyse sans capitalisation | Relancer avec option 1 pour les unités HIGH/MEDIUM |
| PROMOTE-CANDIDATE non promus, session fermée | Règles perdues si option 3 | Utiliser option 2 → `/promote-to-memory doc=<file>` en session suivante |
| PROMOTE-CANDIDATE `[low]` promus sans vérification | Suspicion traitée comme fait établi | Vérifier le code avant d'appliquer le patch dans memory-maintainer |
| `architecture-map.md` vide malgré des analyses | Option 1 jamais choisie | Aucune mémoire active — le système ne sert pas |
| Unité dans le plan mais plus dans le code | Refactoring non tracé | Marquer `🚫 ignoré` + mettre à jour `architecture-map.md` |
| Plan avec toutes les unités `🚫 ignoré` | Scope trop large défini en Phase 0 | Refaire la cartographie avec un périmètre réduit |

---

## Voir aussi

- `.claude/skills/codebase-analysis/SKILL.md` — spécification complète
- `.claude/skills/codebase-analysis/references/user-manual.md` — guide d'utilisation
- [ARCHITECTURE_DOCUMENTATION_PATTERN.md](ARCHITECTURE_DOCUMENTATION_PATTERN.md) — ce qui suit l'analyse
- `.claude/skills/architecture-reviewer/references/user-manual.md` — exploiter les docs produits
