<!--
SPDX-License-Identifier: GPL-3.0-or-later
Copyright (C) 2026 Claire Ivanenka <claire@gnu-ai.org>

Kanban de l'Inference Translator, dérivé du PLAN.md : une carte
par tâche. Déplacer une carte = la déplacer entre les sections
ci-dessous. Le détail des livrables et des critères d'acceptation
reste dans PLAN.md.
-->

# Inference Translator — Kanban

Dérivé du [`PLAN.md`](PLAN.md). Chaque carte est préfixée par sa
phase ; les critères d'acceptation de chaque phase sont dans le plan.

## À faire

### Phase 0 — Spécification et contrats
- [ ] Gel du trio JSON `request`/`status`/`result` et de l'arborescence `/inference`
- [ ] Gel du mode distant : transport SSH (commande serveur, session), format de trame, poignée de main `hello`, version, registre des clés publiques SSH
- [ ] Protocole éditeur : résolution `$VISUAL` → `$EDITOR` → `nano` → `vi`, fichier temporaire, sémantique du code de sortie
- [ ] Palette et conventions d'affichage (codes ANSI, thèmes de base, réglage `NO_COLOR`)
- [ ] Convention des points de montage : `/inference`, `/orchestrate`
- [ ] Livrable : `SPEC.md` + squelette de code compilable

### Phase 1 — MVP : le translator `/inference` (mode local)
- [ ] Translator monté via `settrans` sur `/inference`
- [ ] `write` du prompt sur `/inference/prompt` ; extraction des URLs (expression rationnelle simple)
- [ ] `read` de `/inference/request` (type 1), `/urls`, `/task`, `/status`
- [ ] Détection heuristique minimale de la tâche (mots-clés), surcharge manuelle via `/inference/task`

### Phase 2 — Le client `inference` : IHM minimale
- [ ] Bannière colorée, saisie multi-lignes (`termios`), commandes de base (`/editor`, `/task`, `/submit`, `/status`, `/quit`)
- [ ] Intégration `$EDITOR` : composition du prompt, relecture du buffer, re-soumission
- [ ] Narration des actions avec le nœud POSIX concerné à chaque étape

### Phase 3 — Interface riche : couleurs, animations, streaming
- [ ] Coloration syntaxique du prompt (URLs, markdown léger) et des sorties JSON (clés, chaînes, nombres, états)
- [ ] Animations : spinner pendant l'attente du `result`, effet machine à écrire, réaffichage propre
- [ ] Lecture incrémentale de `status` (type 2) puis du `result` (type 3) en streaming
- [ ] Mode cluster : le trio JSON transite par un canal SSH minimal vers l'orchestrateur d'un cluster Hurd (2 à 5 nœuds)

### Phase 4 — Intégration de bout en bout avec l'orchestrateur
- [ ] Scénario complet : prompt (éditeur) → `request` → orchestrateur (httpfs, N neuron-translator) → `result` affiché en streaming
- [ ] Affichage du `status` (instances, états, scores) comme données de progression

### Phase 5 — Mode distant : SSH, datacenter et cluster local
- [ ] Processus `inference-serveur` lancé par session SSH (trames sur stdio, aucun port d'écoute dédié)
- [ ] Registre des clés publiques SSH via PostgreSQL (`data-base-translator`) : enregistrement, `authorized_keys`, révocation côté serveur, journal des refus
- [ ] Client : `inference --remote utilisateur@hôte` ; sans l'option, comportement local inchangé

### Phase 6 — Historique et ergonomie de session
- [ ] Historique des prompts de la session (navigation, ré-édition dans `$EDITOR`, re-soumission)
- [ ] Commandes `history` et `replay` cohérentes avec `/orchestrate`, sans dupliquer la persistance (PostgreSQL reste la source)

### Phase 7 — Durcissement, tests, CI
- [ ] Tests déterministes du translator (suites POSIX sans terminal), tests du client via pseudo-terminaux
- [ ] Tests du mode distant : sshd de test sur boucle locale, clé refusée, révocation à chaud, coupure de session
- [ ] CI sous QEMU GNU/Hurd, pilotée par [gnu-ai/mistral-vm-debian-hurd](https://github.com/gnu-ai/mistral-vm-debian-hurd)
- [ ] Documentation (`docs/interface.md`, `docs/remote.md`, `docs/architecture.md`)

## En cours

_(rien)_

## Fait

_(rien — le dépôt en est au squelette autotools ; la première carte est la phase 0)_
