# Debug Pilleur Master — Methodologie de Debug Universelle

> Document de reference. A integrer dans le CLAUDE.md de chaque projet.
> Ce protocole est NON NEGOCIABLE pour TOUT diagnostic, dans TOUT projet.
> Tout bug, toute modification, tout cablage commence par un tracage de flux complet.
> Tracer le flux EST la source de verite. Pas les suppositions. Pas la memoire. Le flux reel.

## Principe Fondamental

**"Un vrai debug trace le cheminement complet de bout en bout."**

Zero mock. Zero code mort. 100% des fonctionnalites connectees. Un diagnostic qui ne trace pas le flux complet = un diagnostic incomplet = INTERDIT.

### Regle universelle : TRACER AVANT DE TOUCHER

Quel que soit le type de modification, IDENTIFIER et VERIFIER chaque maillon de la chaine AVANT de modifier quoi que ce soit. Si un seul maillon est manquant, casse, ou non verifie, le diagnostic est incomplet et toute modification est INTERDITE.

---

## Les 6 types de flux

Tout systeme logiciel se decompose en combinaisons de ces 6 flux. Identifier le(s) flux concerne(s) AVANT de debugger.

| Flux | Description | Quand l'utiliser |
|------|-------------|------------------|
| A | Entree utilisateur → Backend | Action declenchee depuis l'UI ou la CLI vers le backend |
| B | Backend → Backend | Module/service qui appelle un autre module/service |
| C | Backend → UI | Donnee produite par le backend et affichee dans l'interface |
| D | UI interne | Composants, state management, navigation frontend |
| E | Pipeline / Workflow | Chaine de traitements sequentiels (CI/CD, agents, batch) |
| F | Configuration / Infra | Configs, schemas, variables d'env, permissions, hooks |

---

## FLUX A — Entree utilisateur → Backend (6 etapes)

Pour chaque fonctionnalite declenchee par l'utilisateur (clic, commande, requete).

### 1. Point d'entree identifie

Ou l'utilisateur declenche l'action ?

```bash
# UI web — appels reseau
grep -rn "fetch\|axios\|sendMessage\|emit\|submit\|onClick\|dispatch" src/ app/ frontend/ COCKPIT/

# CLI — commandes
grep -rn "argparse\|click\|typer\|sys.argv\|command" cli/ commands/ tools/

# API — endpoints
grep -rn "route\|endpoint\|@app\.\|@router\." api/ routes/ controllers/ views/
```

### 2. Transport verifie

Par quel mecanisme la requete arrive au backend ?

| Transport | Quoi chercher |
|-----------|---------------|
| REST API | `@app.get`, `@router.post`, `route()` |
| WebSocket | `register_handler`, `on_message`, `subscribe` |
| CLI direct | `def main`, `if __name__`, commandes |
| IPC / Event bus | `emit`, `publish`, `subscribe`, `on()` |
| Message queue | `consume`, `produce`, `channel` |

```bash
grep -rn "register_handler\|@app\.\|@router\.\|route(" server/ api/ core/
```

### 3. Handler/controleur localise

Le code qui recoit et traite la requete existe ?

```bash
grep -rn "def handler_name\|def action_name\|async def" handlers/ controllers/ core/
```

### 4. Module backend appele

Le handler appelle quel module metier ?

```bash
# Suivre les imports et appels dans le handler
grep -n "from\|import\|self\._get\|self\." handlers/handler_file.py
```

### 5. Methode backend verifiee

La methode appelee existe ET sa signature correspond a l'appel ?

```bash
grep -n "def method_name\(" module_file.py
# Comparer params avec l'appel reel dans le handler
```

### 6. Reponse traitee par l'UI

L'UI/CLI gere correctement la reponse retournee ?

```bash
grep -rn "data.action\|response\.\|then(\|catch(\|onMessage\|handleResponse" src/ frontend/ COCKPIT/
```

### Checklist Flux A

- [ ] Point d'entree utilisateur identifie (UI/CLI/API)
- [ ] Transport verifie (REST/WS/CLI/IPC)
- [ ] Handler/controleur existe et recoit la requete
- [ ] Module metier appele correctement
- [ ] Methode backend existe avec signature conforme
- [ ] Reponse traitee cote UI/CLI

### Schema

```
+-------------------+
|   Utilisateur     |  1. Point d'entree
|  clic/cmd/requete |
+---------+---------+
          |
          v
+---------+---------+
|    Transport      |  2. REST/WS/CLI/IPC
|   route/handler   |
+---------+---------+
          |
          v
+---------+---------+
|   Handler         |  3. Controleur
|  recoit + parse   |
+---------+---------+
          |
          v
+---------+---------+
|   Module metier   |  4-5. Logique backend
|   method()        |
+---------+---------+
          |
          v
+---------+---------+
|   Reponse         |  6. Retour vers UI/CLI
|   JSON/stdout/evt |
+-------------------+
```

---

## FLUX B — Backend → Backend (4 etapes)

Pour chaque modification impliquant des modules/services qui s'appellent entre eux.

### 1. Appelant identifie

Qui appelle la fonction/methode/service ?

```bash
grep -rn "module_name\.\|from.*module_name import" src/ core/ lib/ avaa_dev/
```

### 2. Interface verifiee

La signature correspond a ce que l'appelant envoie ?

```bash
# Lire la signature reelle
grep -n "def method_name\(" module_name.py

# Comparer avec l'appel reel
grep -n "module_name.method_name\(" appelant.py
```

Points a verifier :
- Nombre de parametres identique
- Types compatibles
- sync vs async (await present si async)
- Parametres optionnels avec defaults coherents

### 3. Instanciation verifiee

Le module est-il instancie/accessible au moment de l'appel ?

```bash
# Singleton ? Factory ? Registry ? DI container ? Import direct ?
grep -rn "ModuleClass(\|get_instance\|registry\|inject\|@singleton" src/ core/
```

### 4. Retour verifie

Ce que la methode retourne est-il ce que l'appelant attend ?

- Type de retour (dict, list, str, objet, None possible ?)
- Structure (quelles cles ? quels champs ?)
- Cas d'erreur (exception ? None ? dict vide ? code erreur ?)

### Checklist Flux B

- [ ] Tous les appelants identifies (grep exhaustif)
- [ ] Signature conforme (params, types, sync/async)
- [ ] Module instancie et accessible au runtime
- [ ] Valeur de retour conforme a ce que l'appelant consomme

### Schema

```
+-------------------+
|    Module A       |  1. Appelant
|  appelant.method  |
+---------+---------+
          |
          | import / appel
          v
+---------+---------+
|    Interface      |  2. Signature verifiee
|  def method(...)  |
+---------+---------+
          |
          | instanciation
          v
+---------+---------+
|   Instanciation   |  3. Singleton/DI/registry
|   accessible ?    |
+---------+---------+
          |
          | retour
          v
+---------+---------+
|    Module A       |  4. Consomme le retour
|  utilise result   |
+-------------------+
```

---

## FLUX C — Backend → UI (5 etapes)

Pour chaque modification impliquant un module backend qui envoie des donnees vers l'interface.

### 1. Emetteur identifie

Quel module backend produit la donnee ?

```bash
grep -rn "send\|emit\|response\|broadcast\|publish\|return.*json\|jsonify\|render" src/ core/ api/ controllers/
```

### 2. Format de la donnee verifie

La structure envoyee correspond a ce que l'UI attend ?

- Lister CHAQUE cle/champ envoye
- Comparer avec ce que le frontend lit reellement
- Types : string vs number vs boolean vs objet vs array

### 3. Canal de transport verifie

Par quel mecanisme la donnee transite ?

| Transport | Quoi chercher |
|-----------|---------------|
| REST response | `return jsonify`, `Response()`, status codes |
| WebSocket push | `send_json`, `emit`, `broadcast`, action name |
| SSE | `event-stream`, `yield` |
| Fichier/stdout | `print`, `write`, `save` |

```bash
grep -rn "send_response\|send_json\|jsonify\|emit\|broadcast\|action.*=" src/ core/
```

### 4. Reception UI verifiee

Le frontend a un handler pour cette donnee ?

```bash
grep -rn "fetch\|axios\|onMessage\|subscribe\|data\.action\|addEventListener\|useQuery" src/ frontend/ app/ COCKPIT/
```

### 5. Rendu verifie

Le composant affiche correctement les donnees recues ?

- Bon champ (le nom de cle cote UI = le nom cote backend)
- Bon format (string attendu, pas un objet)
- Binding reactif (pas une copie statique deconnectee)
- Etats d'erreur et de chargement geres

### Checklist Flux C

- [ ] Module emetteur identifie
- [ ] Structure de donnees envoyee documentee (chaque cle listee)
- [ ] Canal de transport identifie et verifie
- [ ] Handler de reception UI existe
- [ ] Rendu/binding correct dans le composant

### Schema

```
+-------------------+
|   Module Backend  |  1. Emetteur
|  send/emit/return |
+---------+---------+
          |
          | JSON/dict/objet
          v
+---------+---------+
|   Format Donnee   |  2. Structure verifiee
|  cles, types      |
+---------+---------+
          |
          | REST/WS/SSE/IPC
          v
+---------+---------+
|   Canal Transport |  3. Mecanisme
|   verifie         |
+---------+---------+
          |
          v
+---------+---------+
|   Reception UI    |  4. Handler frontend
|  fetch/onMessage  |
+---------+---------+
          |
          v
+---------+---------+
|   Rendu           |  5. Composant
|  binding, display |
+-------------------+
```

---

## FLUX D — UI interne (3 etapes)

Pour chaque modification purement frontend (composants, state management, navigation).

### 1. Source de donnees identifiee

D'ou vient la donnee affichee ?

```bash
grep -rn "useStore\|useState\|defineProps\|computed\|ref(\|reactive(\|createStore\|useSelector\|mapState" src/ frontend/ app/ COCKPIT/
```

- Store global (Pinia, Redux, Vuex, Zustand) ?
- Props du composant parent ?
- Etat local (ref, useState) ?
- Appel reseau direct ?
- Context/injection (provide/inject, React context) ?

### 2. Flux de donnees trace

De la source jusqu'au composant qui affiche : chaque maillon verifie.

```bash
grep -rn "import.*store\|import.*component\|provide\|inject\|useContext\|connect(" src/ frontend/
```

### 3. Reactivite verifiee

Les bindings sont reactifs ? Les mises a jour se propagent ?

- Pas de copie brute deconnectee du state
- Le composant se re-rend quand la donnee change
- Watchers/subscriptions en place si necessaire
- Pas de mutation directe du state (si pattern immutable)

### Checklist Flux D

- [ ] Source de donnees identifiee (store, props, local, reseau)
- [ ] Chaine de donnees tracee de bout en bout
- [ ] Reactivite confirmee

### Schema

```
+-------------------+
|   Source Donnee   |  1. Store/Props/Local/API
|   identifiee      |
+---------+---------+
          |
          | import/props/inject
          v
+---------+---------+
|   Flux Donnees    |  2. Chaine tracee
|   parent → enfant |
+---------+---------+
          |
          | ref/computed/watch
          v
+---------+---------+
|   Reactivite      |  3. Binding verifie
|   rendu correct   |
+-------------------+
```

---

## FLUX E — Pipeline / Workflow (5 etapes)

Pour chaque chaine de traitements sequentiels (CI/CD, pipeline agentique, batch processing, ETL).

### 1. Declencheur identifie

Qu'est-ce qui demarre le pipeline ?

```bash
grep -rn "trigger\|schedule\|cron\|main\|entry_point\|start\|run_pipeline\|on_event" pipeline/ workflow/ ci/ .github/
```

### 2. Etapes listees et ordonnees

Quelles etapes, dans quel ordre, avec quelles dependances ?

```bash
grep -rn "step\|stage\|phase\|task\|job\|substep" pipeline/ workflow/ config/
```

### 3. Interface inter-etapes verifiee

Chaque etape recoit la sortie de la precedente ? Format compatible ?

- Output etape N = Input etape N+1
- Types/schemas compatibles
- Gestion des erreurs : que se passe-t-il si une etape echoue ?

### 4. Etat / statut verifie

L'etat du pipeline est correctement mis a jour a chaque etape ?

```bash
grep -rn "status\|state\|FAIT\|EN_COURS\|FAILED\|DONE\|PENDING" pipeline/ state/ config/
```

### 5. Sortie finale verifiee

Le resultat final est-il ce qui est attendu ? Artefacts produits ? Preuves presentes ?

### Checklist Flux E

- [ ] Declencheur identifie
- [ ] Toutes les etapes listees dans l'ordre
- [ ] Interfaces inter-etapes compatibles (output N = input N+1)
- [ ] Etat/statut mis a jour correctement
- [ ] Sortie finale conforme + preuves presentes

### Schema

```
+-------------------+
|   Declencheur     |  1. Trigger/cron/event
|   start pipeline  |
+---------+---------+
          |
          v
+---------+---------+
|   Etape 1         |  2. Listee + ordonnee
|   input → output  |
+---------+---------+
          |
          | output = input suivant
          v
+---------+---------+
|   Etape N         |  3. Interface verifiee
|   input → output  |
+---------+---------+
          |
          v
+---------+---------+
|   Etat pipeline   |  4. Statut mis a jour
|   DONE/FAILED     |
+---------+---------+
          |
          v
+---------+---------+
|   Sortie finale   |  5. Artefacts + preuves
|   livrable reel   |
+-------------------+
```

---

## FLUX F — Configuration / Infra (4 etapes)

Pour chaque modification touchant aux configs, schemas, permissions, variables d'environnement, hooks.

### 1. Fichier de config identifie

Quel fichier de configuration est concerne ?

```bash
grep -rn "config\|settings\|env\|schema\|permission" config/ .env .config/ settings/
```

### 2. Consommateurs identifies

Quels modules/services lisent cette config ?

```bash
grep -rn "config_key\|getenv\|load_config\|read_settings" src/ core/ lib/
```

### 3. Coherence verifiee

La valeur dans la config est coherente avec ce que les consommateurs attendent ?

- Cle existante (pas de typo)
- Type correct (string vs int vs bool)
- Valeur dans les bornes attendues
- Pas de config en double avec des valeurs differentes

### 4. Impact trace

Quels comportements changent si cette config change ?

```bash
grep -rn "if.*config_key\|when.*setting\|gate\|hook\|permission" src/ core/ hooks/
```

### Checklist Flux F

- [ ] Fichier de config identifie
- [ ] Tous les consommateurs identifies (grep exhaustif)
- [ ] Coherence cle/type/valeur verifiee
- [ ] Impact du changement trace

### Schema

```
+-------------------+
|   Fichier Config  |  1. .env/yaml/json/toml
|   cle = valeur    |
+---------+---------+
          |
          | lu par
          v
+---------+---------+
|   Consommateurs   |  2. Modules qui lisent
|   load/getenv     |
+---------+---------+
          |
          | applique
          v
+---------+---------+
|   Coherence       |  3. Type/valeur/bornes
|   pas de conflit  |
+---------+---------+
          |
          v
+---------+---------+
|   Impact          |  4. Comportements
|   gates/hooks/if  |
+-------------------+
```

---

## Problemes courants (tous flux)

| Probleme | Symptome | Flux | Solution |
|----------|----------|------|----------|
| Handler manquant | Action ignoree silencieusement | A | Ajouter le handler/route |
| Methode inexistante | AttributeError / 404 | A, B | Implementer la methode |
| Mock oublie | Donnees statiques retournees | A, B | Supprimer le mock, appeler le vrai module |
| Handler UI manquant | UI ne reagit pas | A, C | Ajouter le handler de reponse |
| Singleton non initialise | Module retourne None/undefined | B | Verifier instanciation |
| Signature incompatible | TypeError a l'appel | B | Comparer def vs appel reel |
| Import circulaire | ImportError au demarrage | B | Lazy import ou injection |
| Cle JSON manquante | undefined/KeyError cote UI | C | Comparer cles envoyees vs lues |
| Donnee non reactive | UI ne se met pas a jour | C, D | Verifier bindings reactifs |
| Store non initialise | undefined dans le composant | D | Verifier initialisation du store |
| Etape pipeline cassee | Chaine s'arrete silencieusement | E | Verifier output N = input N+1 |
| Statut jamais mis a jour | Pipeline bloque EN_COURS | E | Ajouter mise a jour statut |
| Config typo | Valeur par defaut prise a tort | F | Verifier nom exact de la cle |
| Config doublon | Comportement imprevisible | F | Deduplication, source unique |
| Async non awaite | Promise au lieu de valeur | B | Ajouter await, verifier async/def |
| Transport silencieux | Donnee envoyee jamais recue | C | Verifier canal + handler reception |
| Format incompatible | TypeError cote consommateur | B, C | Comparer type envoye vs attendu |
| Preuve absente | Statut FAIT sans evidence | E | Collecter preuves avant de marquer |

---

## Commandes de reference

```bash
# === FLUX A — Entrees utilisateur ===

# Points d'entree UI/CLI
grep -rn "onClick\|submit\|sendMessage\|fetch\|axios\|command" src/ frontend/ cli/

# Routes/endpoints
grep -rn "@app\.\|@router\.\|route(\|register_handler" api/ server/ core/

# === FLUX B — Backend interne ===

# Qui appelle un module
grep -rn "from.*module_name import\|module_name\." src/ core/ lib/

# Ou un module est instancie
grep -rn "ModuleClass(\|get_instance\|@singleton" src/ core/

# Sync/async
grep -n "async def\|def " module_name.py

# === FLUX C — Backend → UI ===

# Envois vers le frontend
grep -rn "send_response\|send_json\|jsonify\|emit\|broadcast\|return.*json" src/ core/ api/

# Receptions frontend
grep -rn "onMessage\|data.action\|then(\|useQuery\|subscribe" src/ frontend/ app/

# === FLUX D — UI interne ===

# Sources de donnees
grep -rn "useStore\|useState\|defineProps\|createStore\|useSelector" src/ frontend/

# Reactivite
grep -rn "ref(\|reactive(\|computed(\|watch(\|useEffect\|useMemo" src/ frontend/

# === FLUX E — Pipeline ===

# Etapes
grep -rn "step\|phase\|stage\|task\|substep" pipeline/ workflow/

# Statuts
grep -rn "DONE\|FAIT\|FAILED\|EN_COURS\|PENDING\|status" pipeline/ state/

# === FLUX F — Config ===

# Fichiers de config
find . -name "*.yaml" -o -name "*.yml" -o -name "*.toml" -o -name "*.env" -o -name "settings.*" | head -20

# Consommateurs
grep -rn "load_config\|getenv\|config\[" src/ core/

# === UNIVERSEL ===

# Chercher n'importe quoi
grep -rn "terme" src/ core/ lib/ tests/ docs/
```

---

## Protocole de lecture des resultats — Regle #30

### Pourquoi ce protocole existe

Les agents IA lisent en diagonale par defaut. Ils scannent les premieres et dernieres lignes, devinent le reste, et concluent "tout semble OK". Ce protocole INTERDIT ce comportement.

### Regles absolues

1. **NE JAMAIS RESUMER** un output sans l'avoir lu ligne par ligne d'abord.
2. **NE JAMAIS DIRE** "ca a l'air bon", "tout semble correct", "le reste est similaire".
3. **Chaque affirmation = une preuve.** Si tu dis "la methode existe", cite la ligne exacte avec son numero.
4. **Chaque comptage = un decompte exhaustif.** Si tu dis "3 erreurs", liste les 3.
5. **Si un resultat fait 200 lignes, lis 200 lignes.** Decoupe en segments si necessaire. Ne saute rien.

### Protocole en 5 passes

**PASSE 1 — Comptage brut**
- Nombre total de lignes de donnees (hors decoration)
- Nombre de lignes negatives (NON, 0, None, null, error, failed, ABSENT, N/A, vide)
- Nombre de lignes positives (OK, PASS, True, success)
- Ecrire : `COMPTAGE : X lignes, Y negatifs, Z positifs`

**PASSE 2 — Lecture ligne par ligne**
Pour CHAQUE ligne, dans l'ordre :
- LIRE mot par mot
- IDENTIFIER le type (valeur, chemin, statut, nom)
- Signaler IMMEDIATEMENT : caracteres corrompus, donnees tronquees, zeros inattendus, NULL inattendus, doublons, incoherences, encodage casse, valeurs par defaut suspectes

**PASSE 3 — Cross-referencement**
- Croiser colonnes/sections entre elles
- Verifier coherence des totaux (somme sous-totaux = total ?)
- Identifier les absences (quelles donnees DEVRAIENT etre la mais ne sont PAS ?)

**PASSE 4 — Detection des mensonges d'agent**
Patterns connus a verifier :
- "FAIT" sans preuve — aucune commande ne confirme l'action
- Comptage embelli — "9/10" mais 3 sont casses en realite
- Erreur masquee — `except: pass` qui avale les erreurs
- Resultat selectif — `| head -5` au lieu de tout montrer
- Statut partial — "Tests passent" sans comptage complet visible
- Redefinition de critere — l'agent change la definition de "succes"

**PASSE 5 — Implications et actions**
Pour chaque probleme : impact, priorite, cause racine probable, action corrective exacte.

### Score de lecture

Apres les 5 passes :
- `SCORE : X/Y lignes propres`
- `VERDICT : PROPRE | SUSPECT — VERIFIER | CORROMPU — CORRIGER`
- Si une seule ligne est critique, le verdict ne peut PAS etre PROPRE.

---

## Comportements interdits

Ces anti-patterns sont les plus frequents. Ils DOIVENT etre interceptes.

### Paresse de lecture
- **INTERDIT** : Lire 30 lignes sur 500 et conclure
- **INTERDIT** : "Le reste du fichier suit le meme pattern"
- **INTERDIT** : Ouvrir un fichier, lire la signature de la classe, et fermer
- **OBLIGATOIRE** : Lire integralement ou par segments sans saut

### Diagnostic par supposition
- **INTERDIT** : "Cette methode devrait exister" → GREP d'abord
- **INTERDIT** : "Ce module est probablement initialise" → VERIFIER d'abord
- **INTERDIT** : "L'import semble correct" → LIRE l'import reel
- **OBLIGATOIRE** : Chaque fait avance est prouve par une commande executee

### Debug reactif au lieu de proactif
- **INTERDIT** : Executer d'abord, diagnostiquer apres quand ca plante
- **OBLIGATOIRE** : Tracer le flux complet AVANT de modifier (Debug Pilleur Master)
- **OBLIGATOIRE** : Comprendre le contexte INPUT/OUTPUT/CONNEXIONS avant de toucher

### Faux diagnostics
- **INTERDIT** : "Tout fonctionne correctement" sans preuve d'execution
- **INTERDIT** : Marquer "FAIT" sans avoir execute la commande de verification
- **INTERDIT** : Montrer un resultat partiel (head -5, tail -3) comme preuve de succes global
- **OBLIGATOIRE** : Resultat complet ou rien. Si trop long, decouper et tout montrer.

### Conservatisme excessif
- **INTERDIT** : Demander permission a chaque micro-etape quand la tache est claire
- **INTERDIT** : Proposer au lieu d'executer quand l'instruction est explicite
- **OBLIGATOIRE** : Si l'instruction dit "fais", FAIS. Si doute bloquant, UN SEUL STOP, pas un STOP tous les 3 fichiers.

---

## Comment utiliser ce document

### Dans un nouveau projet

1. Copier ce fichier dans `docs/DEBUG_PILLEUR_MASTER.md`
2. Dans le CLAUDE.md du projet, ajouter une reference :
   ```
   > Methodologie de debug : docs/DEBUG_PILLEUR_MASTER.md
   > Ce protocole est NON NEGOCIABLE pour tout diagnostic.
   ```
3. Adapter les commandes grep aux chemins specifiques du projet

### Pour chaque bug ou modification

1. Identifier le(s) flux concerne(s) (A a F)
2. Suivre les etapes du flux dans l'ordre
3. Remplir la checklist — chaque point verifie avec preuve
4. Si un point manque, le flux est casse → corriger AVANT de continuer

### Pour chaque lecture de resultat

1. Appliquer le protocole en 5 passes
2. Produire le score et le verdict
3. Ne JAMAIS conclure sans avoir lu integralement

---

*Methodologie originale : session AVAA v3.3.0 (2026-01-27)*
*Version universelle : 2026-05-24*
*Applicable a tout projet logiciel — aucune dependance a un framework ou langage specifique*
