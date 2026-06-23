# Choix & gestion des modèles (Claude)

> **En une phrase.** Démarrer fort puis descendre au plus petit modèle qui passe l'éval, router chaque étape vers le bon tier (Haiku pré-traitement, Sonnet synthèse, Opus en secours), épingler des versions figées et traiter les dépréciations comme une routine — pour que TechWatch reste fiable, sourcé et reproductible sans se ruiner.

## Pourquoi c'est nécessaire pour TechWatch IA

Le différenciateur de TechWatch est la *confiance* : chaque affirmation d'une synthèse de migration doit citer sa source. Cette barre de fiabilité n'a pas le même prix selon l'étape — filtrer et dédupliquer des changelogs officiels ne demande pas la même intelligence que rédiger une note de migration sourcée sans hallucination. Comme le workflow est déterministe (9 étapes) et piloté par les évals, le choix de modèle devient une décision d'ingénierie mesurable : on affecte le bon modèle à chaque étape, on épingle la version pour reproduire les évals, et on planifie la migration plutôt que de présumer qu'une version épinglée restera disponible pour toujours.

## Le principe

Trois idées se combinent.

**1. Démarrer fort, puis descendre.** On prototype la *faisabilité* sur le modèle le plus capable (Opus 4.8), pour savoir si la tâche est atteignable du tout. Une fois la barre de qualité connue, on vise « le plus petit modèle qui fait le travail » : c'est là que se gagnent la latence et le coût (applied-llms.org). « Le plus petit qui suffit » est relatif à la tâche *et* à sa difficulté — un même modèle peut suffire pour le pré-traitement et pas pour la synthèse sourcée.

**2. Router par tâche selon la complexité.** Plutôt qu'un gros modèle partout, on décompose le workflow et on affecte un modèle par étape (pattern « routing » d'Anthropic) : classifier l'entrée et l'envoyer vers une tâche spécialisée. Pour TechWatch :

- **Haiku 4.5** ($1/$5 par MTok, contexte 200K) — pré-traitement volumineux : classification, extraction des claims candidates, déduplication des changelogs, normalisation des sources.
- **Sonnet 4.6** ($3/$15 par MTok, contexte 1M) — synthèse de migration fiable et sourcée, avec citations vérifiables.
- **Opus 4.8** ($5/$25 par MTok, contexte 1M) — réservé aux sections que l'éval révèle insuffisantes (fidélité ou sourçage sous le seuil).

Le pré-traitement étant le plus gros consommateur de tokens, le faire en Haiku réduit fortement le coût total. Et comme Haiku plafonne à 200K de contexte, dès que les sources dépassent ce volume, on route vers Sonnet/Opus (1M) — pertinent pour un projet long-context sans vector DB.

**3. Piloter chaque décision par les évals.** On ne choisit ni ne rétrograde un modèle « au feeling ». On gèle un jeu d'éval (affirmations attendues + leurs sources officielles), on fait tourner le candidat dessus, on compare aux seuils (fidélité, sourçage vérifiable, citations correctes), et on ne descend en gamme que si la qualité tient. L'éval doit tolérer de la variance : même à paramètres fixes, la sortie n'est jamais garantie identique bit-à-bit.

Enfin, **on épingle les versions**. Depuis la génération 4.6, chaque ID est un *snapshot figé* — les IDs sans date (`claude-sonnet-4-6`, `claude-opus-4-8`) sont déjà des versions immuables, pas des pointeurs « evergreen ». Pour les modèles plus anciens, on préfère l'ID daté (ex. `claude-sonnet-4-5-20250929`) à l'alias qui peut bouger.

## Exemples concrets

**Pipeline React 18→19 à 3 niveaux.** Haiku pré-traite (filtrer les sources officielles React/Next, dédupliquer les changelogs, extraire les claims candidates), Sonnet produit la synthèse sourcée avec citations, et un *gate* d'éval déclenche un re-run sur Opus uniquement pour les sections dont le score de fidélité/sourçage passe sous le seuil.

**Config de modèles centralisée.** Un seul ID épinglé par étape, jamais en dur dispersé dans le code — une migration devient alors un changement de chaîne + un passage d'éval.

```python
# config/models.py — un ID figé par étape du workflow
MODELS = {
    "pretraitement": "claude-haiku-4-5",   # extraction, dédup, normalisation
    "synthese":      "claude-sonnet-4-6",  # note de migration sourcée
    "fallback":      "claude-opus-4-8",    # si l'éval l'exige
}

def run_synthese(messages):
    # Sonnet 4.6 : pas de budget_tokens — on utilise effort.
    return client.messages.create(
        model=MODELS["synthese"],
        max_tokens=16000,
        thinking={"type": "adaptive"},
        output_config={"effort": "high"},  # low | medium | high | max
        messages=messages,
    )
```

**Monter l'effort plutôt que de monter en gamme.** Sur Opus/Sonnet, `output_config.effort` (`low`/`medium`/`high`/`max`, et `xhigh` sur Opus 4.7+) ajuste la profondeur de raisonnement. Souvent, passer de `medium` à `high` suffit là où on aurait, par réflexe, changé de modèle — à tester avant de rétrograder ou d'upgrader. Attention : ce levier n'est pas universel — **Haiku 4.5 n'a ni `effort` ni thinking adaptatif** (et Opus 4.8 n'a pas de thinking étendu hérité), donc le levier de qualité diffère selon le tier.

## Subtilités & pièges à éviter

- **Alias vs snapshot a changé de sémantique.** Avant la 4.6, un alias sans date pouvait pointer vers un nouvel ID daté et *bouger* silencieusement. À partir de la 4.6, l'ID sans date *est* le snapshot figé. Ne supposez jamais qu'un ID sans date est evergreen sans vérifier la génération.
- **Épingler protège du *drift*, pas du *retrait*.** Anthropic préserve les poids mais ne garantit pas la re-disponibilité : un modèle retiré renvoie une erreur. La migration reste obligatoire à terme — un plan documenté n'est pas optionnel.
- **Changer de modèle en cours de session invalide le prompt cache** (les caches sont scopés par modèle). Un routing qui alterne les modèles *au milieu d'un turn* paie des écritures de cache à froid : routez à la **frontière des étapes**, pas en cours d'appel.
- **Le tokenizer a changé avec Opus 4.7** (partagé par Opus 4.8) : pour un même texte, ~1× à 1,35× de tokens de plus que sur les modèles antérieurs. Lors d'une migration cross-génération, re-baseliner les budgets de contexte et `max_tokens` via `count_tokens` — pas de multiplicateur en aveugle.
- **Une recall plus basse après migration peut être un artefact de harnais**, pas une régression : les Opus récents suivent plus littéralement une consigne du type « ne remonte que le critique ». Adaptez le prompt avant de conclure que le nouveau modèle est moins bon.
- **Le préavis de 60 jours couvre l'API publique.** Certains retraits historiques ont été plus courts, et les plateformes partenaires (Bedrock, Vertex AI) fixent leurs propres calendriers, distincts de l'API Anthropic.
- **Piège — Opus partout « pour être sûr ».** Multiplie coût et latence sans gain de fiabilité sur les étapes simples : le pré-traitement n'a pas besoin d'Opus.
- **Piège — rétrograder sans éval avant/après.** On découvre la dégradation de fidélité/sourçage en prod, exactement ce que le différenciateur « confiance » ne peut pas se permettre.
- **Piège — coder les IDs en dur, dispersés.** Chaque migration devient une chasse multi-fichiers et empêche un routing/éval propre.
- **Piège — ignorer le cycle de dépréciation.** Ne pas surveiller la page `model-deprecations` ni auditer l'usage Console mène à des requêtes 400 le jour du retrait — en pleine détection d'une nouvelle version à synthétiser.
- **Piège — migrer le jour du retrait.** Sans test préalable du remplaçant sur le jeu d'éval, on hérite des changements cassants (`budget_tokens`→adaptatif, prefill retiré, params d'échantillonnage en 400) au pire moment.

## Checklist d'application

- [ ] Prototyper la faisabilité de chaque étape sur Opus 4.8 d'abord, pour fixer la barre de qualité.
- [ ] Centraliser un ID de modèle épinglé **par étape** dans une seule config (Haiku pré-traitement, Sonnet synthèse, Opus fallback).
- [ ] Vérifier que les IDs en prod sont des snapshots figés (sans date pour 4.6+, datés pour les modèles antérieurs) — jamais d'alias evergreen.
- [ ] Geler un jeu d'éval (claims attendues + sources officielles) et mesurer fidélité, sourçage vérifiable, citations correctes.
- [ ] Avant toute rétrogradation de gamme, faire tourner le candidat sur le jeu d'éval gelé et comparer aux seuils.
- [ ] Tester chaque migration sur **une requête unique** d'abord, inspecter `stop_reason`/`usage` et le comportement (thinking, citations), puis dérouler.
- [ ] Essayer de monter `effort` (ou activer le thinking adaptatif) avant de changer de gamme.
- [ ] Router à la frontière des étapes, jamais en alternant les modèles au milieu d'un turn (préserve le prompt cache).
- [ ] Re-baseliner les tokens via `count_tokens` lors d'une migration cross-génération (tokenizer Opus 4.7+).
- [ ] Surveiller `model-deprecations`, auditer l'usage via l'export CSV de la Console (par clé API + modèle), et assigner une date de retrait dès la dépréciation annoncée.
- [ ] (Optionnel) Lire les capacités par modèle via la Models API (`max_input_tokens`, `max_tokens`, `capabilities`) pour des gates de routing robustes aux nouvelles versions.

## 10 questions / réponses

**Q1. Par quel modèle commencer quand on ne sait pas si la tâche est faisable ?**
Par le plus capable (Opus 4.8). On vérifie d'abord que la tâche est atteignable, ce qui fixe la barre de qualité ; ensuite seulement on cherche à descendre vers un modèle plus petit et moins cher. C'est la recommandation conjointe des praticiens et de la doc Anthropic (applied-llms.org ; Models overview).

**Q2. Pourquoi ne pas simplement tout faire en Opus « pour être tranquille » ?**
Parce que ça multiplie le coût et la latence sans gain de fiabilité sur les étapes simples. Le pré-traitement (extraction, dédup) n'a pas besoin d'Opus ; le faire en Haiku ($1/$5 vs $5/$25 par MTok) réduit fortement le coût d'un workflow volumineux. C'est l'anti-pattern direct de « smallest model that works » (applied-llms.org).

**Q3. Comment décide-t-on quel modèle va sur quelle étape ?**
Par routing selon la complexité : on classe la tâche et on l'envoie vers le modèle adapté — Haiku pour le pré-traitement, Sonnet pour la synthèse, Opus pour les cas durs. C'est exactement le pattern « routing » d'Anthropic, qui cite l'exemple d'envoyer les questions faciles vers Haiku et les difficiles vers un modèle plus capable (Building Effective Agents).

**Q4. Que veut dire « épingler une version » et pourquoi est-ce différent depuis la 4.6 ?**
Épingler = utiliser un identifiant de modèle qui ne bougera pas. Avant la 4.6, un alias sans date pointait vers un snapshot daté susceptible de changer ; à partir de la 4.6, l'ID sans date (`claude-sonnet-4-6`, `claude-opus-4-8`) *est* déjà un snapshot figé. Pour les modèles antérieurs, on épingle l'ID daté (ex. `claude-sonnet-4-5-20250929`) plutôt que l'alias evergreen (Models overview ; applied-llms.org).

**Q5. Comment justifier le choix Haiku/Sonnet/Opus côté budget ?**
Par l'échelle de prix input/output par MTok : Haiku 4.5 $1/$5, Sonnet 4.6 $3/$15, Opus 4.8 $5/$25. Comme le pré-traitement est l'étape la plus gourmande en tokens, le placer en Haiku écrase le coût total ; on ne paie Sonnet et Opus que là où la qualité l'exige (Models overview).

**Q6. Quel est le critère pour autoriser une rétrogradation de gamme ?**
Le passage d'un jeu d'éval gelé. On fait tourner le modèle plus petit sur des affirmations attendues et leurs sources officielles, et on ne rétrograde que si la fidélité, le sourçage vérifiable et les citations correctes restent au-dessus des seuils. Sans harnais d'éval, on ne peut pas rétrograder avec confiance (applied-llms.org).

**Q7. Comment gérer concrètement le cycle de dépréciation ?**
Comme une routine : surveiller la page `model-deprecations` (états Active / Legacy / Deprecated / Retired), auditer l'usage via l'export CSV de la Console (par clé API et par modèle) pour repérer tout appel à un modèle déprécié, et assigner une date de retrait dès la dépréciation. Compter sur ≥ 60 jours de préavis avant retrait, mais comme plancher, pas comme garantie (Model deprecations).

**Q8. Si une version est épinglée, suis-je à l'abri pour toujours ?**
Non. Anthropic préserve les *poids* mais ne garantit pas la re-disponibilité : un modèle retiré renvoie une erreur. L'épinglage protège du *drift silencieux* (le comportement ne change pas sous vos pieds), pas du retrait. Un plan de migration documenté reste obligatoire (Commitments on Model Deprecation and Preservation).

**Q9. Faut-il monter en gamme ou monter l'effort quand la qualité manque ?**
Souvent monter l'effort suffit. Sur Opus/Sonnet, `output_config.effort` (`low`→`max`, plus `xhigh` sur Opus 4.7+) ajuste la profondeur de raisonnement, et activer le thinking adaptatif peut donner ce qu'on cherchait sans changer de modèle. À tester avant de rétrograder/upgrader — sachant que Haiku n'a ni effort ni thinking adaptatif (Models overview ; Migration guide).

**Q10. Quels changements cassants anticiper lors d'une migration cross-génération ?**
Le thinking adaptatif (`type: "adaptive"`) remplace `budget_tokens` (qui renvoie 400 sur Opus 4.7+) ; `temperature`/`top_p`/`top_k` renvoient 400 sur Opus 4.7+ ; le prefill assistant est retiré (utiliser `output_config.format`). De plus, le tokenizer a changé avec Opus 4.7 (~1×–1,35× de tokens), donc re-baseliner `max_tokens` et les budgets via `count_tokens`. Tester d'abord sur une requête unique, inspecter `stop_reason`/`usage`, puis dérouler (Migration guide).

## Sources

1. [Models overview — Claude API Docs (Anthropic)](https://platform.claude.com/docs/en/about-claude/models/overview) — *official-docs* — familles Claude (Haiku 4.5, Sonnet 4.6, Opus 4.8), tarifs ($1/$5, $3/$15, $5/$25 par MTok), fenêtres de contexte (200K vs 1M), mécanique des IDs figés depuis la 4.6, et recommandation de démarrer sur le modèle le plus capable.
2. [Model deprecations — Claude API Docs (Anthropic)](https://platform.claude.com/docs/en/about-claude/model-deprecations) — *official-docs* — politique du cycle de vie (Active / Legacy / Deprecated / Retired), préavis d'au moins 60 jours, audit d'usage via export CSV de la Console, dépréciation des paramètres d'échantillonnage.
3. [What We Learned from a Year of Building with LLMs (applied-llms.org)](https://applied-llms.org/) — *practitioner* — prototyper sur le modèle le plus capable puis viser le plus petit qui suffit, piloter les choix de modèle par les évals, et épingler des versions précises pour la reproductibilité en prod.
4. [Building Effective Agents (Anthropic Engineering)](https://www.anthropic.com/engineering/building-effective-agents) — *official-engineering* — pattern « routing » : classifier l'entrée et router vers une tâche spécialisée (questions faciles → Haiku, difficiles → modèle plus capable).
5. [Commitments on Model Deprecation and Preservation (Anthropic)](https://www.anthropic.com/research/deprecation-commitments) — *official-policy* — pourquoi les modèles sont retirés (coût d'inférence), engagements de préservation des poids, et l'implication : aucune disponibilité garantie à long terme — il faut planifier la migration.
6. [Model migration guide — Claude API Docs (Anthropic)](https://platform.claude.com/docs/en/about-claude/models/migration-guide) — *official-docs* — changements cassants par modèle (thinking adaptatif vs `budget_tokens`, suppression des paramètres d'échantillonnage, prefill retiré) et méthode « tester sur une seule requête d'abord ».
