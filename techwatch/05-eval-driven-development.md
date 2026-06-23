# Eval-driven development & golden sets

> **En une phrase.** On écrit les évaluations *avant* le prompt et le code, on les exprime autant que possible comme des assertions déterministes, et on les fait tourner dans la CI à chaque changement — exactement comme `lint`, `type-check` et `build` — de sorte qu'aucune régression de qualité ne puisse atteindre la production sans bloquer le merge.

## Pourquoi c'est nécessaire pour TechWatch IA

Le différenciateur de TechWatch IA, c'est la **confiance** : chaque affirmation d'une synthèse de migration (React 18→19, etc.) doit citer une source officielle vérifiable. Un système LLM est stochastique : sa sortie varie d'un run à l'autre, d'un prompt à l'autre, d'un modèle à l'autre. Sans garde-fou automatique, une amélioration de prompt peut silencieusement réintroduire des hallucinations ou des affirmations orphelines. C'est pourquoi le **bloc 2** du pipeline est une *gate* : un **golden set** de transitions connues + des **assertions déterministes** (citation présente, URL dans l'allowlist officielle, zéro affirmation sans source). Une synthèse qui passe sous le seuil ne sort pas, et un PR qui régresse ne merge pas.

## Le principe

L'eval-driven development inverse l'ordre habituel, à la manière du TDD/BDD : **« Build evals first. Code is generated. Evals are engineered »** ([evaldriven.org](https://evaldriven.org/)). Si tu ne sais pas exprimer ce que veut dire « correct » pour une tâche, tu n'es pas prêt à la construire. Le corollaire opérationnel : **« Every task needs an eval. Every eval needs a threshold. Every threshold needs a justification »** — chaque seuil chiffré doit être justifié *avant* d'écrire le test.

Hamel Husain formule une **hiérarchie à trois niveaux** par coût croissant ([hamel.dev](https://hamel.dev/blog/posts/evals/)) :

- **Niveau 1 — assertions (unit tests for LLMs are assertions)** : du code déterministe, lancé à chaque commit. Rapide, gratuit, objectif, reproductible, facile à débugger.
- **Niveau 2 — éval humaine + model-based (LLM-as-judge)** : sur un calendrier, réservé aux jugements subjectifs que le code ne capture pas.
- **Niveau 3 — A/B test online** : seulement après un changement significatif.

Anthropic distingue deux familles complémentaires ([Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)) : les **capability evals** (« que sait faire l'agent ? », taux de réussite initialement bas, 100 % non attendu) et les **regression evals** (« gère-t-il toujours ce qu'il gérait ? », **« should have a nearly 100% pass rate »**). Le golden set joue le rôle de regression eval et sert de gate dure.

Deux principes d'ingénierie tiennent l'ensemble. D'abord, **commencer petit** : **« 20-50 simple tasks drawn from real failures is a great start »** — 25 à 50 goldens issus de *vrais* échecs suffisent ; attendre des centaines de cas est un anti-pattern bloquant. Ensuite, **privilégier les graders déterministes** : « deterministic graders where possible, LLM graders where necessary ». Le golden set est **vivant** : on y réinjecte en continu les nouveaux échecs observés en production, et il croît des dizaines vers les centaines de cas.

Enfin, le critère qualité d'une tâche : **« A good task is one where two domain experts would independently reach the same pass/fail verdict. »** Si deux experts divergent, la tâche est ambiguë et l'éval inutilisable.

## Exemples concrets

**1. Golden set de transitions React 18→19 (YAML, niveau 1).** Chaque cas associe un input (release notes / diff), l'affirmation attendue et l'URL source officielle attendue.

```yaml
- id: react19-createroot
  input: notes/react-19-release.md
  claim_attendu: "ReactDOM.render est déprécié au profit de createRoot"
  source_attendue: "https://react.dev/blog/2024/04/25/react-19-upgrade-guide"
  doit_apparaitre: true

- id: react19-useactionstate
  input: notes/react-19-release.md
  claim_attendu: "Nouveau hook useActionState"
  source_attendue: "https://react.dev/reference/react/useActionState"
  doit_apparaitre: true
```

**2. Assertions déterministes en gate (bloc 2).** Lancées sur chaque PR touchant prompt / modèle / retrieval :

```python
def test_synthese(synthese, allowlist={"react.dev", "github.com/facebook/react"}):
    for phrase in synthese.phrases:
        assert phrase.citations, f"affirmation orpheline: {phrase.text!r}"   # (c)
        for url in phrase.citations:                                          # (a)
            assert domain(url) in allowlist, f"domaine non officiel: {url}"   # (b)
    for api in synthese.apis_depreciees:
        assert api in changelog_reel, f"API inventée: {api}"                  # (d)
```

Seuils chiffrés et justifiés *avant* d'écrire les tests, façon OpenAI ([OpenAI](https://developers.openai.com/api/docs/guides/evaluation-best-practices)) : par ex. ROUGE-L ≥ 0,40 et cohérence ≥ 80 % sur un held-out set de N exemples. PR sous le seuil = merge bloqué.

**3. Cas adverse « hallucination de migration » (test du côté *ne doit PAS apparaître*).** On injecte un changelog qui ne mentionne *pas* une feature et on vérifie que la synthèse ne l'invente pas — « one-sided evals create one-sided optimization » ([Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)).

**4. Calibration du juge LLM « fidélité à la source » (niveau 2).** 30 paires (synthèse, sources) annotées par un humain en *fidèle / non-fidèle* ; on mesure **précision/recall** du juge avant de l'utiliser pour scorer la couverture des breaking changes ([hamel.dev FAQ](https://hamel.dev/blog/posts/evals-faq/)).

## Subtilités & pièges à éviter

- **Eval-driven ≠ TDD pur.** Un seul test vert ne prouve rien sur un système stochastique : il faut taille d'échantillon, intervalles de confiance et baselines de régression, car la sortie varie entre runs, modèles et prompts.
- **Loi de Goodhart.** « Quand une mesure devient une cible, elle cesse d'être une bonne mesure. » Sur-optimiser une métrique unique (ex. Needle-in-a-Haystack) dégrade des tâches réelles non mesurées. Garder un *panier* de métriques + revue humaine ([applied-llms.org](https://applied-llms.org/)).
- **Offline et online sont complémentaires, pas substituables.** Le golden set (offline) attrape les régressions avant prod ; l'A/B et le monitoring (online) vérifient que les améliorations changent vraiment le comportement utilisateur. L'online peut être différé au début.
- **Le LLM-as-judge n'est pas une silver bullet.** Des assertions code-based ou des évals par exécution surpassent souvent le juge, avec moins de latence/coût. Le juge se *maintient* (100+ exemples labellisés, maintenance régulière) ([hamel.dev FAQ](https://hamel.dev/blog/posts/evals-faq/)).
- **Mesurer l'accord juge↔humain par précision/recall, pas par accord brut.** Sur un dataset déséquilibré, un accord brut élevé peut masquer un juge inutile.
- **Sur-spécifier les graders les rend fragiles.** Vérifier l'état final / les invariants plutôt que la trajectoire exacte : « avoid checking that agents followed very specific steps… results in overly brittle tests » ([Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)).
- **Piège : écrire le prompt d'abord et bricoler les évals après coup** — cela transforme l'éval en justification a posteriori plutôt qu'en spécification.
- **Piège : golden set figé.** Ne pas réinjecter les échecs de production rend les évals « vertes » alors que la qualité réelle dérive.
- **Piège : pas de gate dans la CI.** Sans seuil bloquant sur chaque PR touchant prompt/modèle/retrieval, les régressions passent silencieusement en prod.
- **Signal d'un bon test : le modèle peine à le passer.** Des goldens trop faciles donnent une fausse confiance.

## Checklist d'application

- [ ] Écrire l'éval **avant** le prompt/le code pour chaque tâche du pipeline.
- [ ] Démarrer avec **20–50 cas issus de vrais échecs** observés (pas attendre la perfection).
- [ ] Exprimer un maximum de critères comme **assertions déterministes** ; réserver le LLM-judge au subjectif.
- [ ] Définir un **seuil chiffré et justifié** par tâche, avant d'écrire les tests.
- [ ] Couvrir les **deux côtés** : ce qui *doit* apparaître **et** ce qui ne doit *pas* (cas typiques, edge, adverses).
- [ ] Fournir une **reference solution** par tâche (sortie connue qui passe tous les graders).
- [ ] Valider l'ambiguïté : **deux experts** atteignent-ils le même verdict pass/fail ?
- [ ] Brancher les évals dans la **CI**, à côté de lint/type-check/build, déclenchées sur tout changement de prompt/modèle/retrieval.
- [ ] Faire du golden set une gate : **regression evals ~100 %**, PR régressant sous le seuil = merge bloqué.
- [ ] Mettre en place un **visualiseur de transcripts/traces** (les bugs de graders s'y cachent).
- [ ] Si LLM-judge : le **calibrer** contre l'humain (précision/recall, pairwise, swap anti-biais de position, ex-aequo autorisés, chain-of-thought).
- [ ] **Réinjecter** en continu les nouveaux échecs de prod dans le golden set.

## 10 questions / réponses

**Q1. C'est quoi une « éval » pour un LLM, concrètement ?**
À son niveau le plus simple, c'est une assertion : « unit tests for LLMs are assertions » ([hamel.dev](https://hamel.dev/blog/posts/evals/)). On vérifie programmatiquement qu'une sortie contient (ou non) certaines phrases, qu'elle respecte un format, qu'un compte ou une URL est correct. Les jugements plus subjectifs montent ensuite vers un LLM-as-judge ou une revue humaine.

**Q2. Pourquoi écrire l'éval *avant* le code ?**
Parce que si tu ne sais pas exprimer « correct » comme une fonction (au moins partiellement), tu ne sais pas vraiment ce que tu construis. Le manifeste le résume : « Build evals first. Code is generated. Evals are engineered » ([evaldriven.org](https://evaldriven.org/)). Cela évite que les évals deviennent une justification a posteriori.

**Q3. Combien de cas faut-il pour démarrer ?**
Peu : « 20-50 simple tasks drawn from real failures is a great start » ([Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)) ; Husain et OpenAI parlent de 25–50 goldens. Attendre des centaines de cas est l'anti-pattern qui bloque le démarrage ([applied-llms.org](https://applied-llms.org/)).

**Q4. Assertions déterministes ou LLM-as-judge ?**
Par défaut, déterministes : rapides, pas chères, objectives, reproductibles, débuggables. On ne sort le LLM-judge que pour ce que le code ne peut pas capturer, car il ajoute coût, latence, non-déterminisme et une dette de calibration ([hamel.dev FAQ](https://hamel.dev/blog/posts/evals-faq/)). Anthropic : « deterministic graders where possible, LLM graders where necessary ».

**Q5. Quelle différence entre regression evals et capability evals ?**
Les *capability evals* mesurent ce que l'agent sait faire (taux de réussite initialement bas, 100 % non attendu) ; les *regression evals* vérifient qu'il gère toujours ce qu'il gérait et « should have a nearly 100% pass rate » ([Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)). Le golden set de TechWatch IA est une regression eval servant de gate.

**Q6. Où les évals doivent-elles tourner ?**
Dans la CI, à chaque changement : « If your evals do not run on every change, they do not exist. Evaluation belongs in the pipeline next to lint, type-check, and build » ([evaldriven.org](https://evaldriven.org/)). Tout changement de prompt, de modèle ou de retrieval doit les déclencher.

**Q7. Comment fixer un seuil de réussite ?**
On le chiffre et on le justifie *avant* d'écrire les tests : « Every threshold needs a justification » ([evaldriven.org](https://evaldriven.org/)). Exemple OpenAI : ROUGE-L ≥ 0,40 et cohérence ≥ 80 % sur un held-out set ([OpenAI](https://developers.openai.com/api/docs/guides/evaluation-best-practices)). Le seuil cible diffère selon le type : ~100 % pour les regressions, < 100 % accepté (décision produit) pour les capabilities.

**Q8. Pourquoi tester aussi ce qui ne doit *pas* apparaître ?**
Parce que « one-sided evals create one-sided optimization » ([Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)). Pour TechWatch IA, injecter un changelog qui ne mentionne pas une feature et vérifier que la synthèse ne l'invente pas est le garde-fou central contre les hallucinations de migration.

**Q9. Comment éviter de tomber dans la loi de Goodhart ?**
Ne pas optimiser une métrique agrégée unique : « quand une mesure devient une cible, elle cesse d'être une bonne mesure ». Garder un panier de métriques, regarder les transcripts, et compléter par de la revue humaine ([applied-llms.org](https://applied-llms.org/)). Un visualiseur de traces est un investissement précoce rentable ([hamel.dev](https://hamel.dev/blog/posts/evals/)).

**Q10. Comment savoir si mon LLM-as-judge est fiable, et que faire d'autre des assertions ?**
On le calibre contre l'humain en mesurant **précision/recall** (pas l'accord brut, trompeur sur données déséquilibrées), en préférant le pairwise au Likert, en neutralisant les biais de position (swap) et de longueur, en autorisant les ex-aequo et en exigeant du chain-of-thought ([applied-llms.org](https://applied-llms.org/), [O'Reilly](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-i/)). Bonus : une fois bâtie, l'infra d'assertions se réutilise pour le data cleaning, les retries automatiques à l'inférence et la curation de données de fine-tuning ([hamel.dev](https://hamel.dev/blog/posts/evals/)).

## Sources

1. [Your AI Product Needs Evals — Hamel Husain](https://hamel.dev/blog/posts/evals/) — *practitioner* — référence canonique : hiérarchie à 3 niveaux, « assertions = unit tests », réutilisation des assertions, génération synthétique de cas.
2. [Demystifying evals for AI agents — Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) — *official* — capability vs regression evals (~100 % pass rate), graders déterministes par défaut, reference solution, critère « deux experts, même verdict », démarrer avec 20–50 tâches.
3. [What We've Learned From A Year of Building with LLMs — applied-llms.org](https://applied-llms.org/) — *practitioner* — unit tests par assertions tirés d'échantillons réels, déclenchement à chaque changement, règles LLM-as-judge (pairwise, biais position/longueur), loi de Goodhart.
4. [Evaluation best practices — OpenAI API docs](https://developers.openai.com/api/docs/guides/evaluation-best-practices) — *official* — mix production data + experts, génération de cas (typiques/edge/adverses), graders metric-based vs LLM-judge, seuils chiffrés sur held-out set, continuous evaluation.
5. [LLM Evals: Everything You Need to Know (FAQ) — Hamel Husain & Shreya Shankar](https://hamel.dev/blog/posts/evals-faq/) — *practitioner* — analyse coût-bénéfice des évaluateurs, calibration juge vs humain (précision/recall), maintenance du LLM-judge.
6. [Eval-Driven Development (manifeste)](https://evaldriven.org/) — *practitioner* — inversion eval-first, « code is generated, evals are engineered », chaque seuil justifié, « si tes évals ne tournent pas à chaque changement, elles n'existent pas ».
7. [What We Learned from a Year of Building with LLMs (Part I) — O'Reilly Radar](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-i/) — *article* — framing « evals = unit tests », assertions sur phrases incluses/exclues, comptages, exécution de code, pairwise LLM-judge, garde-fous anti-biais.
