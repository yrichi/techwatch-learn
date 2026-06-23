# Observabilité LLM (OpenTelemetry, tracing, SLO)

> **En une phrase.** Instrumenter chaque appel LLM avec OpenTelemetry dès le jour 1 — un span `gen_ai.*` par appel portant tokens, coût, latence et qualité — pour rendre un pipeline non déterministe traçable, chiffrable et pilotable par des SLO, dont la fraîcheur des synthèses.

## Pourquoi c'est nécessaire pour TechWatch IA

TechWatch IA vend de la **confiance** : chaque affirmation cite sa source. Sans observabilité, on ne peut ni prouver qu'une synthèse « React 18→19 » est fiable, ni savoir *où*, dans le workflow déterministe en 9 étapes, une sortie a dérapé. L'OTel dès le jour 1 transforme génération et évaluation en signaux mesurables (coût, qualité, fraîcheur), et permet des SLO explicites — dont le plus stratégique : le délai entre la publication officielle d'une version et la synthèse vérifiée publiée.

## Le principe

L'observabilité LLM repose sur les trois signaux d'OpenTelemetry — **traces, métriques, events** — appliqués aux appels de modèles via les **conventions sémantiques GenAI**, un vocabulaire standardisé et vendor-neutral (pas de lock-in sur un backend). ([OpenTelemetry for Generative AI](https://opentelemetry.io/blog/2024/otel-generative-ai/))

Concrètement, chaque appel produit un **span** nommé `gen_ai.chat` (ou `gen_ai.embeddings`) portant des attributs normalisés :

- `gen_ai.operation.name` (`chat`, `text_completion`, `embeddings`),
- `gen_ai.request.model` et `gen_ai.response.model`,
- `gen_ai.response.finish_reasons` (`["stop"]`, `["length"]`…),
- **deux compteurs de tokens distincts** : `gen_ai.usage.input_tokens` et `gen_ai.usage.output_tokens` — jamais un total agrégé, car le coût input et output diffère. ([Uptrace](https://uptrace.dev/blog/opentelemetry-ai-systems))

Côté **métriques**, on alimente le compteur `gen_ai.client.token.usage` (par modèle et type de token, pour l'attribution coût) et l'histogramme `gen_ai.client.operation.duration` (latence en secondes). Pour le streaming, le **time-to-first-token (TTFT)** est un SLI de latence plus pertinent pour l'UX que la durée totale.

Le **contenu** des prompts/réponses se capture en **opt-in** (`capture_message_content=True`) et se stocke comme **span events** (`gen_ai.user.message`, `gen_ai.assistant.message`), **jamais comme attributs** : les attributs sont indexés, plafonnés, et exposent des PII. ([Uptrace](https://uptrace.dev/blog/opentelemetry-ai-systems))

Au-dessus de l'instrumentation, on raisonne en **SRE** : SLI/SLO, coût par 1k requêtes, redaction PII centralisée dans le Collector, corrélation par `trace_id` ([OpenObserve](https://openobserve.ai/blog/opentelemetry-for-llms/)). Le tracing est le pilier central précisément *parce que* le LLM est non déterministe : dans un pipeline multi-étapes, la cause d'une mauvaise sortie finale est souvent une sortie intermédiaire trompeuse, plusieurs étapes en amont ([Langfuse — Observability](https://langfuse.com/docs/observability/overview)). Enfin, le monitoring se décline sur **4 dimensions** : qualité, coût, latence, drift ([Braintrust](https://www.braintrust.dev/articles/what-is-llm-monitoring)).

## Exemples concrets

**1. Trace de bout en bout d'une synthèse « React 18→19 »** (visualisée en waterfall dans Langfuse) :

```
span techwatch.synthesis            (racine, attrs: tech="react", from="18", to="19", release_id, env="prod")
├─ span fetch_release_notes         (source officielle React ; attrs: source.url, source.freshness_s, http.status)
├─ span gen_ai.chat                 (génération synthèse)
│     gen_ai.operation.name = "chat"
│     gen_ai.request.model = "claude-..."
│     gen_ai.usage.input_tokens = 12480
│     gen_ai.usage.output_tokens = 1830
│     gen_ai.response.finish_reasons = ["stop"]
│     event gen_ai.user.message (opt-in, tronqué à ~800 car.)
└─ span gen_ai.chat                 (éval LLM-as-judge → émet un score)
      score "taux_affirmations_sourcees" = 0.94  (source: eval)
```

**2. Calcul du coût en quasi temps réel**, directement depuis le span — sans outil de billing externe :

```python
def cost_from_span(span, pricing):  # pricing externalisé (table de prix par modèle/version)
    p = pricing[span["gen_ai.response.model"]]
    inp = span["gen_ai.usage.input_tokens"]  * p["input_per_1k"]  / 1000
    out = span["gen_ai.usage.output_tokens"] * p["output_per_1k"] / 1000
    return inp + out  # attribuable par requête, feature, équipe et techno suivie
```

**3. SLO de fraîcheur (le SLO clé du projet)** : « p95 du délai entre publication officielle d'une version (ex. React 19 GA) et synthèse vérifiée publiée **< 24 h** », instrumenté via un span d'horloge ; alerte sur le **burn-rate** quand l'error budget se consume.

**4. Détection de drift sur les notes de version** : suivre longueur et structure des release notes ingérées et alerter sur un skew structurel/sémantique signalant un changement de format de la source officielle qui casserait l'extraction des breaking changes ([applied-llms.org](https://applied-llms.org/)).

## Subtilités & pièges à éviter

- **Spec GenAI encore Experimental** : la convention a migré de site (désormais dans le repo `semantic-conventions`) et de nombreux attributs ne sont pas stables → **verrouiller la version de convention** utilisée et anticiper des renommages. ([OTel GenAI semconv](https://opentelemetry.io/docs/specs/semconv/gen-ai/))
- **Extensions par provider** (Anthropic, AWS Bedrock, OpenAI, Azure AI Inference) : les attributs spécifiques *complètent* sans *remplacer* les `gen_ai.*` génériques → rester sur le tronc commun pour la portabilité.
- **Contenu en attributs = piège silencieux** : indexation forcée, troncature, fuite de PII, coûts backend qui explosent. Utiliser des **span events** + cap à ~500–1000 caractères.
- **Un seul compteur « total » de tokens** masque la structure de coût input/output et empêche toute optimisation ciblée.
- **Logs maison non structurés** au lieu d'OTel : on perd la corrélation `trace_id` et le standard vendor-neutral → dette d'observabilité et re-instrumentation forcée plus tard.
- **Alerter sur des seuils bruts fixes** plutôt que sur des **burn-rates de SLO** : bruit, fatigue d'alerte, et on rate les dégradations lentes. Pour le coût, prévoir des **paliers 50/80/100 %** du budget tokens.
- **Coût ≠ trivial** : le pricing varie par modèle, par version, distingue input/output (et cache read/write chez certains providers) → **externaliser la table de prix**, sinon un calcul figé devient faux dès qu'une version change.
- **LLM-as-judge n'est pas une silver bullet** : préférer le **pairwise** au Likert, contrôler le biais de position (faire chaque comparaison deux fois en **swappant** l'ordre), demander un raisonnement avant le verdict, **autoriser les égalités** ([applied-llms.org](https://applied-llms.org/)). Des classifieurs/reward models classiques battent souvent le juge LLM en précision/latence/coût.
- **Goodhart's Law** : « quand une mesure devient une cible, elle cesse d'être une bonne mesure » — sur-optimiser un proxy (needle-in-haystack) dégrade les tâches réalistes de synthèse. Les SLO qualité doivent rester représentatifs de l'usage réel.
- **Hallucinations** : plancher observé de **5–10 %**, difficile de descendre sous **2 %** même sur des tâches simples comme la synthèse → un SLO « zéro hallucination » est irréaliste ([applied-llms.org](https://applied-llms.org/)).
- **Overhead & serverless** : les SDK (Langfuse) envoient en asynchrone/batché (overhead quasi nul), mais les workers serverless/event-driven **doivent appeler `flush()`** avant la fin du process, sinon les traces des invocations courtes sont perdues — exactement là où vit ce projet. ([Langfuse — Overview](https://langfuse.com/docs/observability/overview))
- **Logger seulement les succès** biaise le dataset : capturer aussi timeouts, refus de modération et sorties vides ([applied-llms.org](https://applied-llms.org/)).
- **Pas de retrieval vectoriel à tracer** (long-context, zéro vector DB), mais **tracer le fetch des sources officielles et leur fraîcheur** comme spans first-class (latence, échec, version détectée).

## Checklist d'application

- [ ] OTel instrumenté **dès le jour 1** ; chaque appel LLM = un span `gen_ai.*` vendor-neutral.
- [ ] `gen_ai.usage.input_tokens` et `gen_ai.usage.output_tokens` **séparés** ; métrique `gen_ai.client.token.usage` par (model, token type).
- [ ] Latence via `gen_ai.client.operation.duration` ; **TTFT** suivi pour le streaming.
- [ ] Coût calculé par requête/feature/équipe/techno depuis les spans ; **table de prix externalisée**.
- [ ] Contenu prompts/completions en **opt-in**, en **span events**, plafonné à ~500–1000 car.
- [ ] **Redaction PII centralisée dans le Collector** avant export.
- [ ] Chaque étape du workflow 9 étapes = une **observation enfant** (fetch source, génération, éval).
- [ ] **Scores qualité par trace** émis par génération ET évaluation (numérique/catégoriel/booléen).
- [ ] SLO définis : p95 latence par (provider, model), taux d'erreur par `error.type`, % réponses sous seuil qualité, **fraîcheur des synthèses**.
- [ ] **Alertes burn-rate** (pas seuils bruts) + paliers coût **50/80/100 %**.
- [ ] Alerte **runaway/injection** : `gen_ai.client.token.usage` > ~2× baseline sur 10 min.
- [ ] **TOUTES** les I/O loguées, y compris timeouts / blocages modération / sorties vides.
- [ ] Drift dev/prod surveillé (longueurs I/O, clusters d'embeddings, structure des release notes).
- [ ] **Vibe check quotidien** : revue manuelle d'un échantillon, convertie en assertions/évals.
- [ ] Attributs propagés (`user_id`, `session_id`, `release/version`, `environment`) + **trace IDs custom** entre workers.
- [ ] **`flush()`** appelé dans chaque worker serverless avant fin de process.

## 10 questions / réponses

**Q1. C'est quoi un « span » pour un appel LLM ?**
Un span représente une opération unitaire avec un début, une fin et des attributs. Pour le LLM, c'est typiquement `gen_ai.chat`, portant le modèle, les tokens, le `finish_reason` et la durée. Les spans s'imbriquent pour former une trace de bout en bout du workflow. ([Uptrace](https://uptrace.dev/blog/opentelemetry-ai-systems))

**Q2. Pourquoi OpenTelemetry plutôt que des logs maison ?**
OTel fournit des conventions sémantiques **standardisées et vendor-neutral** : on évite le lock-in et on peut changer de backend sans réinstrumenter. Des logs maison non structurés perdent la corrélation par `trace_id` et créent une dette d'observabilité. ([OpenTelemetry for Generative AI](https://opentelemetry.io/blog/2024/otel-generative-ai/))

**Q3. Pourquoi séparer input_tokens et output_tokens ?**
Parce que le coût d'un token d'entrée diffère de celui d'un token de sortie. Un total agrégé masque la structure de coût et empêche d'optimiser (ex. réduire la taille du prompt). On garde `gen_ai.usage.input_tokens` et `gen_ai.usage.output_tokens` distincts. ([Uptrace](https://uptrace.dev/blog/opentelemetry-ai-systems))

**Q4. Où mettre le texte des prompts et réponses ?**
Dans des **span events** (`gen_ai.user.message`, `gen_ai.assistant.message`), en opt-in via `capture_message_content=True`, et plafonné en longueur. Jamais dans des attributs : ils sont indexés, plafonnés et exposent des PII, faisant exploser les coûts du backend. ([Uptrace](https://uptrace.dev/blog/opentelemetry-ai-systems))

**Q5. Comment calculer le coût d'une synthèse sans outil de billing ?**
Directement depuis le span : `(input_tokens × prix_input) + (output_tokens × prix_output)`, en quasi temps réel, attribuable par requête, feature, équipe et techno. Il faut **externaliser la table de prix** car le pricing change par modèle et version. ([OpenObserve](https://openobserve.ai/blog/opentelemetry-for-llms/))

**Q6. Quelle différence entre SLI, SLO et SLA ?**
Le SLI est la mesure (ex. % de requêtes sous un seuil de latence), le SLO la cible interne (ex. p95 < 24 h pour la fraîcheur), le SLA un engagement contractuel. Pour un MVP interne, on définit surtout des SLO et des error budgets. ([OpenObserve](https://openobserve.ai/blog/opentelemetry-for-llms/))

**Q7. Pourquoi alerter sur des burn-rates plutôt que des seuils fixes ?**
Un seuil brut déclenche au moindre pic et noie sous le bruit (fatigue d'alerte) tout en ratant les dégradations lentes mais soutenues. Le burn-rate mesure la vitesse de consommation de l'error budget : moins de bruit, meilleure détection des vraies dérives. ([OpenObserve](https://openobserve.ai/blog/opentelemetry-for-llms/))

**Q8. Comment émettre la qualité comme signal observable ?**
Via des **scores par trace** (numérique, catégoriel ou booléen ; source eval/human/api) attachés aux observations. Ainsi l'étape d'évaluation pousse un score (ex. « taux d'affirmations sourcées »), couplable au coût pour alerter si la qualité chute pendant que le coût monte. ([Langfuse — Data Model](https://langfuse.com/docs/observability/data-model))

**Q9. Le LLM-as-judge peut-il être mon seul SLI de qualité ?**
Non. Sans calibration humaine ni assertions déterministes, biais de position et de longueur dérivent. Préférer le pairwise au Likert, **swapper l'ordre** des comparaisons, exiger un raisonnement avant le verdict et autoriser les égalités ; des classifieurs classiques le battent souvent en précision/latence/coût. ([applied-llms.org](https://applied-llms.org/))

**Q10. Pourquoi `flush()` est-il critique dans nos workers serverless ?**
Les SDK envoient les traces en asynchrone batché pour un overhead quasi nul, mais une invocation courte se termine avant le flush automatique : les traces sont perdues. Appeler `flush()` avant la fin du process garantit qu'on n'a pas de trous d'observabilité dans l'architecture event-driven du projet. ([Langfuse — Overview](https://langfuse.com/docs/observability/overview))

## Sources

1. [OpenTelemetry for Generative AI](https://opentelemetry.io/blog/2024/otel-generative-ai/) — *doc-officielle* — annonce du GenAI SIG : les 3 signaux (traces/metrics/events) appliqués au LLM et la justification de conventions sémantiques standardisées.
2. [OpenTelemetry GenAI Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — *doc-officielle* — spécification normative (Experimental, migrée vers le repo `semantic-conventions`) : spans `gen_ai`, agent spans, conventions par provider, events, metrics, MCP.
3. [OpenTelemetry for AI Systems: LLM and Agent Observability — Uptrace](https://uptrace.dev/blog/opentelemetry-ai-systems) — *praticien* — guide d'instrumentation fidèle à la spec : noms de spans (`gen_ai.chat`, `gen_ai.embeddings`), `gen_ai.operation.name`, `gen_ai.usage.input/output_tokens`, métriques `gen_ai.client.token.usage` et `gen_ai.client.operation.duration`, capture opt-in via span events.
4. [OpenTelemetry for LLMs: Complete SRE Guide — OpenObserve](https://openobserve.ai/blog/opentelemetry-for-llms/) — *praticien* — traduction SRE : SLI/SLO (p95 par modèle, `error.type`, `finish_reason`), coût par 1k requêtes, redaction PII dans le Collector, attribution coût, debug par `trace_id`.
5. [Langfuse — Tracing Data Model](https://langfuse.com/docs/observability/data-model) — *doc-officielle* — modèle de données : traces, observations (generation/span/event), scores (eval/human/api), sessions, users, tags, environnements, coût/tokens, ingestion OTel, batching async + `flush()`.
6. [Langfuse — Observability Overview](https://langfuse.com/docs/observability/overview) — *doc-officielle* — le tracing comme pilier face au non-déterminisme LLM, overhead nul par envoi asynchrone batché, bonnes pratiques (decorator vs manuel, trace IDs custom pour le tracing distribué).
7. [What We Learned from a Year of Building with LLMs — applied-llms.org](https://applied-llms.org/) — *praticien* — évaluation & monitoring en prod : LLM-as-judge avec garde-fous, logging systématique des I/O (y compris absences de sortie), data flywheel, Goodhart's Law, plancher d'hallucination 5–10 %.
8. [What is LLM monitoring? — Braintrust](https://www.braintrust.dev/articles/what-is-llm-monitoring) — *praticien* — cadre des 4 dimensions du monitoring LLM (qualité, coût, latence, drift) et leur traduction en SLI/SLO + alerting.
