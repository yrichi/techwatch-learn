# Workflow LLM déterministe vs agents autonomes

> **En une phrase.** Quand la séquence d'étapes est connue et ordonnée à l'avance — comme les 9 étapes de TechWatch IA — on choisit un **workflow déterministe** (LLM et outils orchestrés par du code prédéfini), pas un **agent autonome** (le LLM dirige lui-même son processus) : on gagne en fiabilité, en traçabilité et en coût, sans rien perdre en flexibilité utile.

## Pourquoi c'est nécessaire pour TechWatch IA

Le différenciateur de TechWatch IA, c'est la **confiance** : chaque affirmation d'une synthèse de migration doit citer une source officielle vérifiable. Or la traçabilité ne se décrète pas, elle s'architecture. Comme nos 9 étapes (détection de version → récupération du changelog → extraction des breaking changes → rédaction → vérification des citations…) sont **connues et ordonnées**, un workflow déterministe permet de rattacher chaque sortie à une étape précise et à sa source d'entrée. Un agent autonome, qui déciderait lui-même de son enchaînement, casserait cette chaîne d'attribution et nous ferait payer du coût et de la latence sans aucun gain de flexibilité.

## Le principe

Anthropic distingue formellement deux architectures. Un **workflow** est un système « où les LLM et les outils sont orchestrés via des chemins de code prédéfinis » ; un **agent** est un système « où les LLM dirigent dynamiquement leur propre processus et leur usage des outils, gardant le contrôle de la façon dont ils accomplissent leurs tâches » ([Anthropic](https://www.anthropic.com/research/building-effective-agents)). Ce n'est pas un binaire mais un **curseur d'autonomie** : on peut insérer un sous-bloc agentique borné à une seule étape tout en gardant l'ossature globale déterministe.

Le **critère de choix** est simple. Si l'on peut prédire la suite et l'ordre des étapes, on prend un workflow ; on ne réserve l'agent qu'aux cas où le nombre ou l'ordre des étapes **ne peut pas être anticipé** et doit s'adapter aux résultats intermédiaires ([Redis](https://redis.io/blog/agents-vs-workflows/)). La raison profonde est statistique : « la probabilité qu'un agent accomplisse une tâche multi-étapes décroît exponentiellement avec le nombre d'étapes » ([O'Reilly](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-i/)). À l'inverse, un workflow long mais à étapes fiables, avec points de contrôle et retries, reste robuste.

La méthode recommandée tient en quelques règles. **Commencer simple** : un seul appel LLM bien conçu (avec contexte/retrieval) avant tout chaînage, et « n'ajouter de la complexité que lorsqu'elle améliore démontrablement les résultats » ([Anthropic](https://www.anthropic.com/research/building-effective-agents)). **Découper** : un prompt fait une seule chose, bien — on évite le « God Object » monolithique qui se gonfle à chaque cas limite ([applied-llms](https://applied-llms.org/)). **Générer un plan explicite puis l'exécuter** de façon déterministe (chaîne, DAG ou machine à états), pour une exécution reproductible et testable. **Forcer des sorties structurées** (JSON/XML — Claude favorise le balisage XML) pour rendre chaque étape composable et vérifiable. Anthropic propose 5 patterns réutilisables : **prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer**. Enfin, **mesurer avant d'optimiser** : l'eval-driven (tests d'assertion sur de vrais échantillons + LLM-as-Judge en comparaison par paires) est le socle ([Fowler](https://martinfowler.com/articles/gen-ai-patterns/)).

## Exemples concrets

**1. Mapper les 9 étapes sur les patterns Anthropic.** Le pipeline TechWatch reste déterministe de bout en bout :

```
[Trigger event-driven]  nouvelle version détectée (ex. React 19)
        │
        ▼
[Routing]               classer la techno / le type de source → traitement spécialisé
        │
        ▼
[Prompt chaining]       1. récupérer le changelog officiel
                        2. extraire les breaking changes
                        3. rédiger la synthèse de migration
        │
        ▼
[Evaluator-optimizer]   chaque affirmation cite-t-elle une source vérifiable ?
                        sinon → re-extraire (boucle bornée)
```

**2. Cas React 18 → 19, sortie structurée.** Le pipeline ingère le blog officiel React 19 + les release notes GitHub, puis extrait chaque breaking change (retrait de `propTypes`, nouvelle API `use`, Actions…) au format vérifiable :

```json
{
  "affirmation": "propTypes est retiré et ignoré au runtime à partir de React 19",
  "citation_url": "https://react.dev/blog/2024/12/05/react-19",
  "citation_quote": "PropTypes have been removed ..."
}
```

L'étape evaluator **rejette toute affirmation dont la `citation_quote` n'est pas retrouvable dans la source** : pas de quote vérifiable, pas de publication.

**3. La preuve chiffrée du flow engineering.** AlphaCodium illustre le gain du déterministe sur le mono-prompt : « en passant d'un prompt unique à un workflow multi-étapes, ils ont augmenté la précision de GPT-4 (pass@5) sur CodeContests de 19 % à 44 % » ([O'Reilly](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-i/)). Pour TechWatch, on documentera de même : workflow (N appels fixes, latence et coût prévisibles, citations garanties) vs agent équivalent (appels variables, coût/latence non bornés, citations non garanties).

## Subtilités & pièges à éviter

- **Déterminisme d'orchestration ≠ déterminisme de sortie.** Même à température 0, le LLM reste statistiquement variable ; seul le **graphe d'exécution** est déterministe, d'où la nécessité des evals et du monitoring de la dérive. Ne jamais promettre des sorties « reproductibles » au niveau du token.
- **Orchestrator-workers réintroduit du non-déterminisme local.** Il ressemble à de la parallélisation, mais la décomposition n'est **pas prédéfinie** : elle est décidée par le LLM orchestrateur. À surveiller si on l'utilise sur une étape.
- **LLM-as-Judge n'est pas magique.** Les classifieurs classiques et l'évaluation par exécution surpassent souvent le juge LLM. Pour TechWatch, **vérifier les citations par matching factuel/récupération**, pas par jugement LLM seul.
- **Evaluator-optimizer seulement avec un critère clair.** Sans critère d'évaluation net ni gain mesurable à l'itération, ce pattern n'ajoute que coût et latence.
- **Long-context vs vector DB interagit avec le choix.** Un workflow long-context bien découpé peut éviter le RAG au MVP, mais impose de surveiller le coût par appel et la dilution d'attention sur de gros contextes.
- **Piège — l'agent par effet de mode.** Choisir un agent alors que les 9 étapes sont connues : on paie coût, latence et non-déterminisme sans aucun gain de flexibilité. C'est l'anti-pattern central du projet.
- **Piège — le God Object.** Un prompt géant qui fait tout (détection + extraction + synthèse + sourcing) se gonfle aux cas limites et dégrade les cas fréquents.
- **Piège — le framework d'agents lourd dès le MVP.** Abstractions opaques, prompts cachés, debugging difficile : perte d'observabilité, critique quand le sourcing est le différenciateur.
- **Piège — construire avant d'avoir des evals.** Sans tests d'assertion sur de vrais échantillons, impossible de savoir si un « gain » de prompt régresse ailleurs.
- **Piège — faire confiance à la sortie sans validation.** Les modèles produisent du texte même quand ils ne devraient pas, et les log-probs prédisent mal l'exactitude → hallucination de citations.
- **Piège — boucles sans condition d'arrêt ni budget.** Toujours borner étapes, tokens et retries.

## Checklist d'application

- [ ] Vérifier que la séquence d'étapes est **connue et ordonnée** → workflow déterministe confirmé.
- [ ] Commencer par **un seul appel LLM** bien conçu avant tout chaînage.
- [ ] **Découper** : un prompt = une tâche ; bannir le God Object.
- [ ] Modéliser l'orchestration comme **chaîne / DAG / machine à états** explicite.
- [ ] Mapper chaque étape sur un **pattern Anthropic** (chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer).
- [ ] Imposer des **sorties structurées** (JSON/XML) sur chaque étape.
- [ ] Encadrer le pipeline par des **garde-fous** (validation d'entrée, filtrage de sortie) en appels LLM séparés.
- [ ] Ajouter une étape **evaluator** qui rejette toute affirmation sans citation vérifiable.
- [ ] Vérifier les citations par **matching factuel**, pas par LLM-as-Judge seul.
- [ ] Construire un **jeu d'eval** (~20 paires changelog → breaking changes attendus) avant d'optimiser.
- [ ] **Borner** étapes, tokens et retries de toute boucle.
- [ ] Privilégier les **API LLM directes** aux frameworks lourds pour garder l'observabilité.
- [ ] Mettre en place le **monitoring de la dérive** (skew structurel et sémantique).

## 10 questions / réponses

**Q1. Quelle est la différence entre un workflow et un agent ?**
Un workflow orchestre les LLM et outils via des chemins de code prédéfinis ; un agent laisse le LLM diriger dynamiquement son propre processus et son usage des outils ([Anthropic](https://www.anthropic.com/research/building-effective-agents)). Le premier est prévisible et traçable, le second flexible mais non déterministe.

**Q2. Pourquoi TechWatch est-il un workflow et pas un agent ?**
Parce que ses 9 étapes sont connues et ordonnées à l'avance. Le critère de décision est précisément là : workflow quand la séquence est fixe, agent seulement quand le nombre/ordre d'étapes ne peut pas être prédit ([Redis](https://redis.io/blog/agents-vs-workflows/)).

**Q3. Par quoi devrait-on commencer concrètement ?**
Par la solution la plus simple : un seul appel LLM bien conçu avec le bon contexte. On n'introduit chaînage puis agentivité que si la complexité « améliore démontrablement les résultats » ([Anthropic](https://www.anthropic.com/research/building-effective-agents)).

**Q4. Qu'est-ce que le « God Object » et pourquoi l'éviter ?**
C'est un prompt monolithique qui tente tout à la fois (détection, extraction, synthèse, sourcing). Il se gonfle au fil des cas limites et dégrade les cas fréquents. La parade : un prompt fait une seule chose, bien ([applied-llms](https://applied-llms.org/)).

**Q5. Quels sont les 5 patterns d'Anthropic ?**
Prompt chaining (étapes séquentielles), routing (classer puis diriger), parallelization (sectionnement ou vote), orchestrator-workers (décomposition dynamique) et evaluator-optimizer (boucle génération/critique) ([Anthropic](https://www.anthropic.com/research/building-effective-agents)).

**Q6. Pourquoi les agents échouent-ils sur les tâches longues ?**
Parce que « la probabilité qu'un agent accomplisse une tâche multi-étapes décroît exponentiellement avec le nombre d'étapes » : chaque étape peut échouer et la récupération est faible ([O'Reilly](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-i/)). Un workflow à points de contrôle et retries reste, lui, robuste.

**Q7. Le déterminisme du workflow garantit-il des sorties identiques ?**
Non. Le déterminisme porte sur le graphe d'exécution, pas sur le token : même à température 0 le LLM reste statistiquement variable. C'est pourquoi les evals et le monitoring de la dérive sont indispensables.

**Q8. Comment garantir que chaque affirmation cite vraiment sa source ?**
Avec une étape evaluator qui exige une `citation_quote` retrouvable dans la source, et une vérification par **matching factuel** plutôt que par jugement LLM, car le LLM-as-Judge est souvent surpassé par l'évaluation par exécution ou les classifieurs ([applied-llms](https://applied-llms.org/)).

**Q9. Faut-il un framework d'agents dès le MVP ?**
Plutôt non : les frameworks accélèrent le démarrage mais masquent les prompts réels et incitent à complexifier. Le coût caché est la perte d'observabilité — fatal quand le sourcing est le différenciateur. Mieux vaut implémenter les patterns en quelques lignes via les API directes ([Anthropic](https://www.anthropic.com/research/building-effective-agents)).

**Q10. Peut-on quand même introduire un peu d'agentivité ?**
Oui, c'est un curseur, pas un binaire : on peut borner un sous-bloc agentique (ex. orchestrator-workers) à une seule étape tout en gardant l'ossature globale déterministe. Mais cela réintroduit du non-déterminisme local à surveiller, et toute boucle doit être bornée en étapes, tokens et retries.

## Sources

1. [Building Effective AI Agents](https://www.anthropic.com/research/building-effective-agents) — *official-docs* — définitions formelles workflow vs agent, les 5 patterns, et la recommandation « commencer simple, ajouter de la complexité seulement si elle améliore démontrablement les résultats ».
2. [What We've Learned From a Year of Building with LLMs](https://applied-llms.org/) — *practitioner* — plaidoyer pour les pipelines déterministes, « un prompt fait une seule chose », plan explicite puis exécution structurée, eval comme fondation.
3. [What We Learned from a Year of Building with LLMs (Part I) — O'Reilly Radar](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-i/) — *practitioner* — formule de l'échec exponentiel multi-étapes, AlphaCodium 19 %→44 %, sorties structurées (Claude préfère XML), n-shot.
4. [Emerging Patterns in Building GenAI Products — Martin Fowler / Thoughtworks](https://martinfowler.com/articles/gen-ai-patterns/) — *practitioner* — architecture composable et déterministe, Evals et Guardrails comme patterns de production.
5. [Patterns for Building LLM-based Systems & Products — Eugene Yan](https://eugeneyan.com/writing/llm-patterns/) — *practitioner* — catalogue de patterns (evals, RAG, chaining, garde-fous, monitoring) pour structurer un système en composants testables et observables.
6. [AI Agents vs Workflows: When to Use Each — Redis](https://redis.io/blog/agents-vs-workflows/) — *vendor-engineering* — grille de décision synthétique : workflow si séquence connue et fixe, agent si nombre d'étapes imprévisible ; coût/latence et non-déterminisme des agents.
