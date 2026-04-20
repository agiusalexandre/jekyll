---
layout: post
title: "Boulder — Une AI Build Factory sur AWS qui génère, déploie et maintient des apps toute seule"
date: 2026-04-20 08:00:00 +0200
categories: [aws, ai, agents]
tags: [strands-agents, bedrock, amplify, cdk, agentcore, claude, multi-agent, devops]
---

On a tous vécu ça : tu as une idée d'app, tu passes 3 jours sur le scaffolding Next.js, 2 jours à debug le déploiement Amplify, et au final ton MVP sort 2 semaines plus tard. Boulder, c'est le projet perso que j'ai construit pour court-circuiter tout ça. Une commande CLI, une description en langage naturel, et 5 minutes plus tard l'app est live sur Amplify avec son repo GitHub, son backend Cognito/DynamoDB, et un backlog prêt à être itéré — par l'IA elle-même.

Pas un template generator. Pas un copilot. Une **usine autonome** qui génère du code, le pousse, le déploie, détecte les erreurs de build, les corrige, et apprend de ses échecs pour ne pas les reproduire.

## L'architecture : 9 agents, un seul runtime

Boulder tourne sur [Strands Agents SDK](https://github.com/strands-agents/sdk-python) déployé via Bedrock AgentCore. Un seul fichier `main.py` sert de multiplexeur : il reçoit un payload `{agent, prompt}`, résout le nom canonique de l'agent, charge le module Python correspondant via un lazy loader, et l'invoque.

```
                          ┌──────────────┐
                          │ Web Dashboard│
                          │ (Amplify SSR)│
                          └──────┬───────┘
                                 │
┌─────────┐  AgentCore   ┌──────┴────────┐  Bedrock    ┌──────────┐
│ Boulder  │────────────▶│ Strands Agents │───────────▶│  Claude   │
│   CLI    │◀────────────│  (9 agents)    │            │ Opus 4.7  │
└─────────┘              └──────┬────────┘            └──────────┘
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
               ┌────────┐ ┌────────┐ ┌──────────┐
               │ GitHub  │ │Amplify │ │ DynamoDB │
               │  Repo   │ │ Deploy │ │  Tables  │
               └────────┘ └────────┘ └──────────┘
                                          │
          ┌───────────────────────────────┘
          ▼                    ▼
     ┌──────────┐     ┌──────────────┐
     │Learnings │     │  Schedules   │
     └──────────┘     └──────┬───────┘
                             │
                    ┌────────┴────────┐
                    │  EventBridge    │──▶ Lambda
                    │ (every 5 min)   │    (schedule-processor)
                    └─────────────────┘
```

L'idée clé : **tous les agents partagent le même runtime AgentCore** qui scale-to-zero automatiquement. Le CLI (TypeScript) appelle `InvokeAgentRuntimeCommand`, le runtime Python route vers le bon agent. Zéro serveur à gérer.

## Les 9 agents, chacun son job

### Builder — le cœur du système

Le Builder reçoit un nom de projet + une description, et enchaîne :

1. `generate_app` → appel Bedrock avec un prompt massif (~64K tokens max) adapté au framework (Next.js, React SPA, Vue, Astro). Le modèle génère tous les fichiers source d'un coup.
2. Les fichiers sont stockés dans S3 via un **système de handles** — un UUID opaque de 32 caractères. Le LLM ne manipule jamais les fichiers directement, juste le handle. Ça évite d'exploser le contexte sur des apps de 50+ fichiers.
3. `push_to_github` → crée le repo (idempotent), pousse via l'API Git Tree.
4. `deploy_amplify` → crée l'app Amplify, lance le build, poll toutes les 10s pendant max 10 min.

Le Builder détecte automatiquement le framework à partir de la description. Un jeu ? → React SPA. Un SaaS avec auth ? → Next.js + Amplify Gen 2 backend. Un blog ? → Astro. Et il injecte un `BACKLOG.md` starter dans chaque projet pour que l'Autopilot puisse itérer dessus immédiatement.

### Orchestrator — le chef d'orchestre

L'Orchestrator fonctionne en 2 phases :

**Phase 1 — Planning :** Un agent léger (4K tokens, temperature 0.1) analyse le prompt utilisateur et produit un plan JSON structuré en phases. Exemple pour *"Find a profitable SaaS idea in fitness and build it"* :

```json
{
  "phases": [
    { "agents": ["idea_generator", "market_validator", "monetization_strategist"] },
    { "agents": ["builder"] }
  ]
}
```

**Phase 2 — Exécution :** Le plan est converti en DAG via `GraphBuilder` de Strands. Les agents d'une même phase tournent en parallèle. Un nœud **Summarizer** terminal synthétise tous les résultats. Chaque agent est instancié via une factory (pas de `deepcopy` — les clients boto3 avec leur `SSLContext` et leurs locks ne sont pas copiables).

### Les 3 Scouts — idea → validate → monetize

Trois agents purs LLM (pas d'outils, juste du structured JSON output) :

- **Idea Generator** (temperature 0.7) : génère 5 idées SaaS monétisables avec TAM, difficulté technique, time-to-MVP, revenu potentiel.
- **Market Validator** : analyse la concurrence, le market sizing, la viabilité.
- **Monetization Strategist** : propose un business model avec tiers de pricing et projections de revenus.

Ils peuvent tourner en parallèle dans une phase de l'Orchestrator, puis le Builder enchaîne avec l'idée gagnante.

### Backlog Pipeline — plan → implement → review

Le Pipeline orchestre 3 sous-agents pour traiter un item du backlog :

1. **Planner** : lit le backlog + la structure du codebase, split la feature en 1-3 tâches indépendantes avec les fichiers à modifier/créer.
2. **Implementer** : pour chaque tâche, lit les fichiers existants, écrit le code, crée une branche `backlog/{N}-task-{id}`, ouvre une PR.
3. **Reviewer** : lit les diffs, review pour correctness/patterns/bugs/sécurité, commente, et label `ready-to-merge` si OK.

Chaque sous-agent a son propre system prompt calibré et ses tools dédiés (`read_backlog`, `create_branch`, `write_files_to_branch`, `get_pr_diff`, `label_pr`...).

### Autopilot — la boucle infinie

L'Autopilot est le mode "je lâche tout" : il pick le prochain item TODO du backlog, le passe au Pipeline complet, merge si la review passe, vérifie le build Amplify, et recommence. Budget cap : 10 tâches par cycle, 30 secondes de pause entre chaque. Si un build fail, il crée automatiquement une tâche de fix dans le backlog.

En mode AgentCore, il traite **une tâche par invocation** (stateless). En mode local (`run_autopilot()`), c'est une vraie boucle infinie avec gestion du sleep et des erreurs.

### Portfolio Manager & Post-Deploy

- **Portfolio Manager** : monitoring global — liste toutes les apps déployées, health checks, historique de builds, résumé des coûts.
- **Post-Deploy** : maintenance post-déploiement — health checks, détection de dépendances obsolètes, mise à jour automatique, audit SEO, alertes.

## Self-healing builds : le loop de correction

C'est la feature qui fait la différence. Quand un build Amplify échoue (et ça arrive souvent avec du code généré) :

1. `deploy_amplify` retourne l'`appId` + `jobId` de l'échec
2. `fetch_build_logs` récupère les logs de chaque step du build via les URLs pré-signées d'Amplify
3. `analyze_and_learn` envoie les logs à Bedrock pour extraire : type d'erreur, root cause, fix rule
4. Le learning est stocké dans DynamoDB (`boulder-learnings`, PK = error pattern)
5. `fetch_existing_files` récupère l'état actuel du repo → handle S3
6. `modify_app` envoie le handle + l'erreur comme `change_request` à Bedrock → nouveau handle corrigé
7. `push_to_github` + `deploy_amplify` → retry

**Budget : max 2 retries** (3 tentatives total). Après ça, le build est marqué FAILED. Pas de boucle infinie.

## Self-improving prompts : la mémoire des erreurs

Chaque `analyze_and_learn` stocke un pattern dans `boulder-learnings`. Au prochain `generate_app`, la fonction `_get_learnings()` scanne la table et injecte une section `KNOWN ISSUES` dans le prompt :

```
## KNOWN ISSUES — Learn from past failures
- Pattern: missing_tsconfig_paths
  Rule: Always configure paths alias in tsconfig.json for @/ imports
- Pattern: amplify_backend_type_error
  Rule: Use a.string().required() not a.required().string()
```

Le cache est de 5 minutes pour éviter de scanner DynamoDB à chaque build. Résultat : plus Boulder échoue, plus il devient fiable. Le premier build d'un framework donné fail souvent. Le dixième, presque jamais.

## Stack technique : CDK, tout en un stack

Toute l'infra est dans un seul `BoulderStack` CDK :

- **7 tables DynamoDB** (PAY_PER_REQUEST) : builds, agents, agent-executions, learnings, schedules, agent-activity (TTL 7j), sessions (TTL 24h)
- **1 bucket S3** (`boulder-tmp-{account}-{region}`) avec lifecycle 24h sur `handles/` — les fichiers générés sont éphémères
- **1 Lambda** (`boulder-schedule-processor`) en Node.js 22, inline dans le CDK — gère les schedules cron ET réconcilie les builds bloqués en BUILDING depuis >30 min
- **EventBridge** rate(5 minutes) → trigger la Lambda
- **IAM** : 3 rôles séparés — Amplify service role, SSR compute role (pour le dashboard Next.js), AgentCore runtime role (créé par `deploy.py`, enrichi par CDK pour l'accès S3)
- **SSM Parameter Store** : `/boulder/github-token` (SecureString, créé manuellement)

Le deploy des agents est séparé du CDK : `deploy.py` zip le code Python, l'upload sur S3, et crée/met à jour le runtime AgentCore. Ordre de premier déploiement : `deploy.py` d'abord (crée le rôle IAM), `cdk deploy` ensuite (attache la policy S3).

## Ce que j'ai appris

**Les handles S3, c'est vital.** Au début, le Builder passait les fichiers générés inline dans la conversation agent. Sur une app de 40 fichiers, ça explosait le contexte et le coût en tokens. Le système de handles (un UUID opaque qui pointe vers un JSON dans S3) a tout changé : le LLM ne voit jamais le contenu des fichiers, juste un identifiant de 32 caractères.

**Les clients boto3 ne se copient pas.** J'ai perdu une journée à debugger des erreurs `SSLContext` dans le graph executor de l'Orchestrator. `copy.deepcopy` sur un `BedrockModel` qui contient un client boto3 avec un `SSLContext` et des `threading.Lock` → crash. Solution : une factory qui instancie un agent frais à chaque nœud du graphe.

**Le self-healing a un ROI immédiat.** 60% des premiers builds échouent — imports manquants, config TypeScript incorrecte, dépendances mal versionnées. Le loop de correction résout ~80% de ces échecs en 1-2 retries. Sans ça, Boulder serait inutilisable.

**Le prompt engineering, c'est de l'infra.** Le system prompt du Builder fait ~200 lignes. Il encode des règles framework-specific (quand utiliser React SPA vs Next.js), des patterns Amplify Gen 2 (`a.model()`, `defineAuth()`, `generateClient<Schema>()`), et des conventions de code. Chaque ligne a été ajoutée après un échec concret. C'est du code, pas de la prose.

**Temperature par agent, pas global.** Le Planner de l'Orchestrator tourne à 0.1 (on veut du JSON déterministe), l'Idea Generator à 0.7 (on veut de la créativité), le Builder à 0.3 (un compromis entre créativité du code et fiabilité). Un seul paramètre, gros impact.

## Conclusion

Boulder, c'est ~2500 lignes de Python (agents) + ~300 lignes de TypeScript (CDK) + un CLI. Le tout tourne sur des services managés AWS : Bedrock AgentCore, Amplify, DynamoDB, EventBridge, Lambda, S3. Pas de serveur, pas de container, scale-to-zero partout.

Ce n'est pas production-ready pour des apps critiques — le code généré a besoin de review humaine. Mais pour du prototypage rapide, des MVPs, ou de l'exploration d'idées, c'est un accélérateur brutal. `boulder build mon-idee -d "..."` et tu bois ton café pendant que 9 agents font le boulot.

Le repo est open source : [github.com/agiusalexandre/boulder](https://github.com/agiusalexandre/boulder)
