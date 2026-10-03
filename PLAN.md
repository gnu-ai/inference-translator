<!--
SPDX-License-Identifier: GPL-3.0-or-later
Copyright (C) 2026 Claire Ivanenka <claire@gnu-ai.org>

This file is part of the Inference Translator and is free software:
you can redistribute it and/or modify it under the terms of the GNU
General Public License as published by the Free Software Foundation,
either version 3 of the License, or (at your option) any later version.
-->

# Inference Translator — Projet et feuille de route

`inference-translator` est la **couche d'interface homme-machine** de
la pile GNU AI pour GNU/Hurd, dans l'esprit d'un agent de code en
ligne de commande : il reçoit les prompts de l'utilisateur, compose
avec n'importe quel éditeur de texte (`$EDITOR` : nano, vim, emacs,
tout éditeur en ligne de commande), explique ce qu'il fait à chaque
étape, et expose le résultat sous forme **structurée** (requête,
statut, résultat) à
[orchestrator-translator](https://github.com/gnu-ai/orchestrator-translator).
Il ne fait ni calcul neuronal, ni requête réseau applicative, ni
persistance lui-même : il traduit l'intention de l'utilisateur en
requête POSIX — ou en trames JSON chiffrées quand GNU AI tourne dans
un **datacenter ou un cluster local**, accessible avec une **clé par
utilisateur**.

Licence : GPLv3 ou version ultérieure. Langage : C23, POSIX.1-2008,
interfaces Hurd (`trivfs` pour le MVP, `netfs` dès que l'arborescence
de commandes s'étoffe).

---

## 1. Rôle et positionnement

L'esprit Hurd est respecté : chaque responsabilité reste dans un
translator dédié, l'interface ne fait que **traduire** l'humain en
système de fichiers et réciproquement.

| Composant | Responsabilité | Lien |
|---|---|---|
| `inference-translator` | interface de dialogue : prompts, éditeur, couleurs, requêtes structurées, accès distant | ce dépôt |
| `orchestrator-translator` | coordination : scheduler, supervisor, evaluator, aggregator | gnu-ai/orchestrator-translator |
| `neuron-translator` | unité de calcul : réseau sigmoïde feedforward | gnu-ai/neuron-translator |
| `httpfs-translator` | transport pur HTTP → système de fichiers | gnu-ai/httpfs-translator |
| `data-base-translator` | persistance PostgreSQL | gnu-ai/data-base-translator |

### Les deux faces du même rôle

`inference-translator` est un **translator à deux faces** :

- **Face POSIX** (translator `/inference`) : un prompt soumis par
  `write` est stocké, analysé, et exposé en **requête structurée**
  lisible par `read`. C'est cette face que l'orchestrateur consomme —
  contrat gelé en phase 0, conforme à la section 3.3 du
  `PLAN.md` d'`orchestrator-translator`.
- **Face interactive** (client `inference`) : une IHM en ligne de
  commande, colorée et animée, qui explique ce qu'elle fait, laisse
  composer le prompt dans `$EDITOR`, et affiche le résultat agrégé
  en streaming dès que l'orchestrateur l'a produit.

Les deux faces partagent le même état : le client `inference` n'est
qu'un front-end qui lit et écrit dans `/inference`. Tout ce qui est
faisable en interactif l'est aussi en pur POSIX
(`echo "résume https://…" | tee /inference/prompt`).

### Les deux modes de déploiement

Le même contrat fonctionne dans deux situations, sans que le client
ne voie la différence :

- **Mode local** : tout tourne sur la machine ; le translator est
  monté par `settrans` sur `/inference`, l'orchestrateur sur
  `/orchestrate`, tout passe par le système de fichiers.
- **Mode distant** : la pile GNU AI (inference, orchestrateur,
  neurones, PostgreSQL) tourne dans un **datacenter ou un cluster
  local** ; l'utilisateur n'a localement que l'IHM, qui se connecte
  au serveur `inference` distant par une **socket TCP chiffrée** et
  s'authentifie avec une **clé par utilisateur**.

L'invariant est le trio de requêtes JSON de la section 3.3 : en
local ce sont des fichiers lus dans `/inference` et `/orchestrate`,
en distant ce sont des trames portant exactement les mêmes objets.

---

## 2. Fonctionnalités principales

1. **Réception des prompts** : saisie multi-lignes interactive ou
   `write` POSIX sur `/inference/prompt`.
2. **Éditeur intégré** : composition du prompt dans n'importe quel
   éditeur de texte en ligne de commande via `$VISUAL`/`$EDITOR`
   (nano, vim, emacs, ed…), lancé sur un fichier temporaire ; le
   buffer est relu et re-soumis à la sortie de l'éditeur. Un éditeur
   en ligne de commande suffit : pas de TUI embarquée obligatoire.
3. **Explication des actions** : à la manière des agents CLI
   modernes (Mistral Vibe), chaque étape est narrée avant d'être
   exécutée — « je lis le prompt », « j'extrais l'URL https://… »,
   « je soumets la requête à l'orchestrateur », « j'attends le
   résultat agrégé » — avec le point de montage POSIX concerné.
4. **Trois requêtes JSON structurées** (le cœur du contrat,
   section 3.3) : `request` (prompt → requête structurée),
   `status` (suivi de l'exécution), `result` (réponse agrégée
   finale). Mêmes schémas en local (fichiers) et en distant
   (trames).
5. **Extraction structurée** : URLs (expression rationnelle), type de
   tâche (résumer, traduire, répondre, classer…), paramètres
   (langue, longueur, contraintes), exposés dans `request`.
6. **Interface en couleurs et animations** : code d'échappement ANSI
   (256 couleurs) directement via `termios` — pas de dépendance
   TUI externe. Bannière, spinner d'attente, barre de progression,
   réaffichage propre du prompt.
7. **Coloration syntaxique** : le prompt affiché distingue texte,
   URLs, commandes ; `status` et `result` sont colorés (clés,
   chaînes, nombres, états des instances).
8. **Affichage en streaming** : lecture incrémentale de `status`
   puis du `result` final (effet machine à écrire) ; les statuts
   intermédiaires sont affichés comme des données, non comme des
   erreurs.
9. **Mode distant datacenter/cluster** : un démon `inference-serveur`
   écoute sur TCP chiffré, authentifie l'utilisateur par clé, et
   relaie le cycle `request` → `status` → `result`. Les clés sont
   **par utilisateur**, émises par l'opérateur du cluster et
   **révocables côté serveur** sans redéploiement.
10. **Interface translator** : `/inference` reste pilotable en pur
    POSIX (`cat`, `tee`, `settrans`), donc scriptable et testable
    sans terminal ; le mode local n'a besoin ni de socket ni de clé.

---

## 3. Architecture et flux de données

### 3.1 Vue d'ensemble — mode local

```
   utilisateur ◄──────► client `inference` (IHM : couleurs,
        │                éditeur $EDITOR, animations, narration)
        │ read/write
        ▼
┌───────────────────────┐        read          ┌──────────────┐
│  inference-translator │ ─────────────────────►│ orchestrator │
│       (/inference)    │  requête structurée   │(/orchestrate) │
└───────────────────────┘                      └──────┬───────┘
        ▲                                              │ read
        │ read : status/result en streaming            ▼
        └────────────────────────────────  /orchestrate/{status,result}
```

### 3.2 Vue d'ensemble — mode distant (datacenter / cluster local)

```
  machine utilisateur            datacenter / cluster local
┌────────────────────┐  TCP+TLS  ┌────────────────────────────┐
│ client `inference` │◄─────────►│ démon inference-serveur    │
│ (IHM, éditeur,     │  trames   │  │ auth par clé utilisateur │
│  couleurs)         │  JSON     │  ▼                         │
└────────────────────┘           │ /inference, /orchestrate,  │
   clé: ~/.inference/keys/       │ neurones, PostgreSQL       │
        <cluster>.key            └────────────────────────────┘
```

L'orchestrateur poursuit ensuite son flux nominal (httpfs →
N neuron-translator → agrégation → PostgreSQL), décrit en section 3.2
de son `PLAN.md`. `inference-translator` ne connaît de l'orchestrateur
que les points de montage et les schémas du contrat — jamais les
binaires.

### 3.3 Les trois requêtes JSON (gelées en phase 0)

Le contrat repose sur **trois types de requêtes JSON**, identiques en
local (fichiers lus/écrits) et en distant (trames du protocole de la
section 3.5). Rien d'autre ne circule entre l'utilisateur et le
système.

**Type 1 — `request`** : le prompt traduit en requête structurée.
En local : `write` sur `/inference/prompt`, `read` de
`/inference/request`. En distant : trame `request` émise après le
`hello`.

```json
{
  "prompt": "résume https://example.org/article en 5 points",
  "task": "summarize",
  "urls": ["https://example.org/article"],
  "params": { "language": "fr", "points": 5 }
}
```

**Type 2 — `status`** : le suivi de l'exécution orchestrée. En
local : `read` de `/orchestrate/status` (et `/inference/status` pour
l'état de l'interface elle-même).

```json
{
  "run_id": 42,
  "state": "running",
  "instances": [
    { "id": 1, "topology": "10,20,5",  "state": "busy" },
    { "id": 2, "topology": "10,30,5",  "state": "busy" },
    { "id": 3, "topology": "10,20,10", "state": "failed" }
  ]
}
```

**Type 3 — `result`** : la réponse agrégée finale, seule sortie
« officielle » du système. En local : `read` de
`/orchestrate/result`.

```json
{
  "run_id": 42,
  "aggregate_strategy": "majority",
  "output": "1. … 2. … 3. …",
  "confidence": 0.87
}
```

L'orchestrateur y lit les URLs (qu'il confie à `httpfs-translator`),
la tâche et les paramètres ; il n'a **jamais** besoin d'interpréter
le langage naturel lui-même. La détection de tâche est heuristique et
remplissable explicitement via `/inference/task` ; en cas de doute,
`task` vaut `custom` et l'orchestrateur applique sa politique par
défaut.

### 3.4 Arborescence POSIX du translator `/inference`

```
/inference
├── prompt      (write)  prompt brut tel que soumis par l'utilisateur
│               (read)   dernier prompt en cours
├── request     (read)   requête structurée (type 1, JSON)
├── urls        (read)   URLs extraites, une par ligne, dans l'ordre
├── task        (r/w)    type de tâche : summarize | translate |
│                        answer | classify | custom
├── params      (r/w)    paramètres additionnels (JSON, optionnel)
├── status      (read)   état : idle | composing | parsed | submitted
└── version     (read)   version et contrats supportés
```

### 3.5 Mode distant : protocole filaire et clés

Le démon `inference-serveur` écoute sur TCP (port configurable) et
parle un protocole minimal à trames :

- **Chiffrement** : TLS via GnuTLS — le protocole de trames est
  maison, la cryptographie ne l'est jamais (pas de chiffrement
  artisanal).
- **Trame** : en-tête fixe (type : `hello` | `request` | `status` |
  `result` | `error`, longueur de la charge) suivi d'une charge JSON
  — exactement les objets de la section 3.3 pour `request`, `status`
  et `result`.
- **Poignée de main** : à la connexion, le client envoie
  `hello { "version": 1, "key": "…" }` ; le serveur vérifie la clé
  dans son registre et répond `accept` (avec son `version`) ou
  `error` (clé inconnue ou révoquée). Aucune trame applicative n'est
  acceptée avant `accept`.

**Clés par utilisateur** :

- Émises par l'opérateur du datacenter/cluster, à la manière des
  clés API d'un fournisseur d'IA ; chaque clé identifie **un
  utilisateur**, pas une machine.
- Stockées côté client dans `~/.inference/keys/<hôte>.key`
  (permissions 0600), jamais dans le dépôt ni dans les journaux ;
  passées au client (`inference --remote hôte:port --key …`) ou
  prises par défaut dans le répertoire ci-dessus.
- **Révocables côté serveur** : le registre des clés vit dans
  PostgreSQL via `data-base-translator` ; révoquer une clé bloque un
  utilisateur sans toucher les autres, sans redéploiement.
- Toute tentative échouée (`hello` refusé) est journalisée côté
  serveur avec horodatage et origine.

### 3.6 Contrats d'interface (principe clé)

En local, chaque interaction passe par le système de fichiers, jamais
par des sockets ni des API propriétaires ; la socket du mode distant
est l'unique exception, confinée au saut réseau. Les contrats
suivants sont gelés dès la phase 0, alignés sur la section 3.3 du
plan de l'orchestrateur :

| Contract | Échange |
|---|---|
| `utilisateur → inference` | `write` du prompt sur `/inference/prompt` (ou saisie via le client interactif) ; `read` de `status`, `request`. |
| `orchestrator → inference` | `read` de la requête structurée `/inference/request` (type 1). |
| `inference → orchestrator` | `read` de `status` (type 2) et `result` (type 3) sur `/orchestrate` ; l'interface n'écrit jamais dans l'orchestrateur. |
| `client → éditeur` | `fork`/`exec` de `$VISUAL` (sinon `$EDITOR`, sinon nano, sinon vi) sur un fichier temporaire ; relecture du buffer si code de sortie nul. Aucun éditeur ne nécessite de protocole dédié. |
| `client distant → serveur` | TCP+TLS, trames `hello` puis `request`/`status`/`result`/`error` ; clé par utilisateur vérifiée à la poignée de main. |

---

## 4. Décisions de conception

### Pourquoi deux binaires (translator + client) ?

Séparer le translator `/inference` (sans terminal, sans état global,
testable par POSIX seul) du client `inference` (IHM) évite qu'une
interface utilisateur ne s'incruste dans un translator monté, et
permet d'utiliser l'un sans l'autre. C'est la convention Hurd
(à la manière de `settrans` et des translators qu'il pilote) : le
client ne fait que des `open`/`read`/`write` sur `/inference` et
`/orchestrate`.

### Pourquoi une socket TCP dédiée en mode distant, et pas httpfs ?

`httpfs-translator` est un transport **unidirectionnel de contenu
récupéré** (lecture de `content`/`headers`/`status`) : il n'offre ni
poignée de main authentifiée, ni canal bidirectionnel, ni
notification. Le mode distant a besoin des trois : soumettre le
prompt, suivre `status`, recevoir `result` en streaming, le tout avec
une clé par utilisateur. D'où une socket TCP dédiée, porteuse du
**même trio JSON** que le mode local — le contrat ne change pas, seul
le transport diffère. La socket est l'unique exception à « tout par
le système de fichiers », et elle est confinée au saut réseau : dès
que les trames sont reçues côté serveur, tout redevient des fichiers
(`/inference`, `/orchestrate`) dans le datacenter.

### Chiffrement : TLS, pas de cryptographie maison

Le protocole de trames (en-tête + JSON) est défini ici, mais le
chiffrement et l'authenticité de la connexion sont délégués à GnuTLS.
Écrire son propre chiffrement est exclu. Conséquence assumée : le
mode distant ajoute une dépendance (GnuTLS) que le mode local n'a
pas ; le binaire client compile sans GnuTLS si le mode distant est
désactivé à la configuration.

### Clés par utilisateur, révocables

Modèle choisi : celui d'un fournisseur d'IA — l'opérateur du
cluster émet des clés nominatives, les vérifie à chaque poignée de
main, et peut en révoquer une côté serveur (registre dans
PostgreSQL via `data-base-translator`) sans redéployer ni toucher
les autres utilisateurs. Une clé n'identifie pas une machine :
le même utilisateur peut se connecter depuis plusieurs postes avec
la même clé ; la révocation le coupe de partout.

### Pas de bibliothèque TUI

Couleurs, curseur, spinner et effacements passent par des séquences
ANSI écrites directement sur le terminal, avec `termios` pour le mode
brut. Aucune dépendance à ncurses ou à un framework : le MVP doit
rester compilable avec la toolchain GNU/Hurd seule. La coloration
syntaxique est une table de règles minimale (URL, JSON, markdown
léger), pas un moteur complet.

### L'éditeur est un processus enfant, pas une dépendance

Lancer nano, vim, emacs ou tout autre éditeur est un simple
`fork` + `exec` sur un fichier temporaire sous `/tmp`, avec
ré-affichage du buffer édité. Aucune intégration spécifique : la
variable d'environnement standard fait le travail. Les éditeurs
graphiques ne sont pas requis ni exclus : ce qui compte est le code
de sortie et le fichier.

### Interface muette vs interface parlante

Le translator `/inference` ne raconte rien : il expose des données.
La narration (« je fais X ») est une responsabilité du client
interactif, qui affiche chaque étape avec le fichier POSIX concerné.
Ainsi un script consommant `/inference` n'a jamais à filtrer de
prose.

### Neutralité vis-à-vis de l'orchestrateur

`inference-translator` ne connaît que `/orchestrate` et les schémas
du contrat de phase 0. Il ne suppose ni le nombre d'instances, ni
l'existence de `httpfs` ou de PostgreSQL : ces détails restent dans
l'orchestrateur, conformément au principe « chaque translator reste
remplaçable ».

---

## 5. Phases

Chaque phase a un livrable, des critères d'acceptation et une
dépendance explicite. La phase 3 d'`orchestrator-translator`
(acquisition réseau) dépend de notre MVP (phases 0–1) : ces deux
phases sont prioritaires.

### Phase 0 — Spécification et contrats (avant tout code)

- Gel du **trio de requêtes JSON** `request`/`status`/`result`
  (section 3.3) et de l'arborescence `/inference` (section 3.4).
- Gel du protocole filaire distant : format de trame, poignée de main
  `hello`, gestion de version, format et emplacement des clés.
- Protocole éditeur : ordre de résolution `$VISUAL` → `$EDITOR` →
  `nano` → `vi`, fichier temporaire, sémantique du code de sortie.
- Palette et conventions d'affichage (codes ANSI, thèmes de base,
  réglage `NO_COLOR`).
- Convention des points de montage : `/inference`, `/orchestrate`.
- **Livrable** : `SPEC.md` + squelette de code compilable.
- **Acceptation** : revue croisée des contrats avec
  `orchestrator-translator` — le `read` de `/inference/request` doit
  suffire à sa phase 3 sans modification de son code ; le schéma du
  registre de clés validé avec `data-base-translator`.

### Phase 1 — MVP : le translator `/inference` (mode local)

- Translator monté via `settrans` sur `/inference`.
- `write` du prompt sur `/inference/prompt` ; extraction des URLs
  (expression rationnelle simple, pas d'interprétation HTML).
- `read` de `/inference/request` (type 1), `/inference/urls`,
  `/inference/task`, `/inference/status`.
- Détection heuristique minimale de la tâche (mots-clés : « résume »,
  « traduis », « réponds »…), surcharge manuelle via `/inference/task`.
- **Livrable** : `inference-translator` compilable sous Hurd, tests
  POSIX (`echo … | tee /inference/prompt && cat /inference/request`).
- **Acceptation** : un prompt contenant une URL produit une requête
  `request` exacte ; `make check` vert.

### Phase 2 — Le client `inference` : IHM minimale

- Bannière colorée, saisie multi-lignes (mode brut `termios`),
  commandes de base (`/editor`, `/task`, `/submit`, `/status`,
  `/quit`).
- Intégration `$EDITOR` : composition du prompt dans nano/vim/emacs,
  relecture du buffer, re-soumission.
- Narration des actions avec le nœud POSIX concerné à chaque étape.
- **Livrable** : client `inference` fonctionnel contre le translator
  de la phase 1.
- **Acceptation** : un utilisateur compose un prompt dans son éditeur,
  le soumet, et voit la requête `request` expliquée puis affichée.

### Phase 3 — Interface riche : couleurs, animations, streaming

- Coloration syntaxique du prompt (URLs, markdown léger) et des
  sorties JSON (clés, chaînes, nombres) ; états des instances et
  statuts HTTP colorés en données, pas en erreurs.
- Animations : spinner pendant l'attente du `result`, effet machine
  à écrire sur la sortie, réaffichage propre.
- Lecture incrémentale de `status` (type 2) puis du `result`
  (type 3) dès leur production par l'agrégateur.
- **Livrable** : IHM complète contre un orchestrateur simulé par
  fichiers de test.
- **Acceptation** : démonstration visuelle sans scintillement
  (réaffichage atomique), respect de `NO_COLOR` et des redirections
  non-TTY (pas d'ANSI vers un tube).

### Phase 4 — Intégration de bout en bout avec l'orchestrateur

- Scénario complet : prompt (éditeur) → `request` → orchestrateur
  (httpfs, N neuron-translator) → `result` agrégé affiché en
  streaming dans l'IHM.
- Affichage du `status` (instances démarrées, états, scores) comme
  données de progression.
- **Livrable** : démonstration de bout en bout sur Debian GNU/Hurd.
- **Acceptation** : le même prompt fonctionne via l'IHM et via
  `tee`/`cat` purs ; dépend des phases 1–3 d'
  `orchestrator-translator`.

### Phase 5 — Mode distant : datacenter et cluster local

- Démon `inference-serveur` : écoute TCP+TLS (GnuTLS), poignée de
  main `hello` avec vérification de clé, relais des trames
  `request`/`status`/`result` vers `/inference` et `/orchestrate`
  locaux au serveur.
- Registre des clés par utilisateur dans PostgreSQL via
  `data-base-translator` ; émission, vérification à chaque poignée
  de main, **révocation côté serveur**, journal des tentatives
  refusées.
- Côté client : `inference --remote hôte:port --key fichier` ; en
  l'absence de ces options, comportement local inchangé.
- **Livrable** : une pile GNU AI complète exécutée sur un cluster
  local (plusieurs machines ou conteneurs), IHM sur le poste
  utilisateur.
- **Acceptation** : le même prompt donne le même résultat en mode
  local et distant, sans rien changer au client sauf
  `--remote`/`--key` ; la révocation d'une clé bloque son
  utilisateur sans affecter les autres.

### Phase 6 — Historique et ergonomie de session

- Historique des prompts de la session (navigation, ré-édition dans
  `$EDITOR`, re-soumission).
- Commandes `history`, `replay` côté client, cohérentes avec les
  commandes `replay`/`history` de `/orchestrate` (phase 4 de
  l'orchestrateur) — sans dupliquer la persistance, qui reste dans
  PostgreSQL via `data-base-translator` ; l'interface ne garde que
  l'état de session volatile.
- **Livrable** : session interactive complète multi-requêtes.
- **Acceptation** : rejouer une requête précédente depuis l'IHM
  n'émet qu'un `write` sur `/inference/prompt` (ou une trame
  `request` en mode distant).

### Phase 7 — Durcissement, tests, CI

- Tests déterministes du translator (suites POSIX sans terminal),
  tests du client via pseudo-terminaux.
- Tests du mode distant : TLS sur boucle locale, clé invalide,
  révocation à chaud, coupure réseau en cours d'exécution.
- CI sous QEMU GNU/Hurd, pilotée par le sandbox
  [gnu-ai/mistral-vm-debian-hurd](https://github.com/gnu-ai/mistral-vm-debian-hurd).
- Documentation utilisateur (`docs/interface.md`, `docs/remote.md`)
  et architecture (`docs/architecture.md`).
- **Livrable** : version 1.0.

---

## 6. Contraintes et conventions techniques

- **Langue du code et des commentaires** : anglais, style Claude
  Delannoy (commentaires abondants), en cohérence avec
  `neuron-translator` et `orchestrator-translator`.
- **C23 / POSIX.1-2008**, bibliothèques Hurd (`trivfs` pour le MVP,
  `netfs` pour l'arborescence complète de la section 3.4).
- **Zéro allocation dans les chemins chauds** : les chemins d'affichage
  réutilisent des buffers pré-alloués, cohérent avec les choix de
  performance de la pile.
- **Pas de dépendance TUI** : ANSI + `termios` uniquement ; `NO_COLOR`
  et les sorties non-TTY désactivent tout décor.
- **Une seule dépendance réseau : GnuTLS**, requise uniquement par le
  mode distant ; le mode local et le translator seul compilent et
  fonctionnent sans elle.
- **Interface muette côté translator, narration côté client** :
  `/inference` n'expose que des données interprétables.
- **Erreurs réseau ≠ erreurs POSIX** : un statut HTTP non-200 vu dans
  un résultat est une donnée affichée en couleur d'information
  (philosophie `httpfs`), jamais une erreur de l'interface.
- **Clés : jamais dans le dépôt, jamais dans les journaux** ;
  fichiers `~/.inference/keys/` en 0600 ; le registre serveur vit
  dans PostgreSQL.
- Chaque translator reste remplaçable : l'interface ne connaît que
  les points de montage et les contrats, jamais les binaires de la
  pile.

---

## 7. Jalons synthétiques

| Phase | Contenu | Dépend de |
|---|---|---|
| 0 | Spécification, trio JSON, arborescence, protocole distant, clés | — |
| 1 | MVP translator : prompt → `request` (mode local) | 0 |
| 2 | Client `inference` : saisie, `$EDITOR`, narration | 1 |
| 3 | Couleurs, animations, coloration, streaming | 2 |
| 4 | Bout en bout avec `orchestrator-translator` | 1–3, orchestrateur 1–3 |
| 5 | Mode distant : TCP+TLS, démon, clés révocables, cluster | 0, 4 |
| 6 | Historique, `replay`, sessions | 2–4 |
| 7 | Durcissement, CI Hurd, v1.0 | 1–6 |

La phase 1 de ce dépôt est un prérequis direct de la phase 3 de
`orchestrator-translator` (acquisition réseau) : les deux fils de
travail se synchronisent sur le contrat `/inference/request` gelé
en phase 0. La phase 5 (mode distant) dépend du registre de clés
chez `data-base-translator` et n'invalide aucun contrat local : le
trio JSON `request`/`status`/`result` reste l'invariant des deux
modes.
