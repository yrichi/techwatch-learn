# LLM-as-a-judge : usages et pièges

> **En une phrase.** Un LLM peut servir de juge *offline* pour évaluer des qualités subjectives (ton, clarté, complétude d'une synthèse) à condition d'être aligné sur un expert et mesuré rigoureusement — mais il ne doit jamais arbitrer la factualité, qui se vérifie de façon déterministe.

## Pourquoi c'est nécessaire pour TechWatch IA

TechWatch IA produit des synthèses de migration (ex. React 18→19) dont une partie de la qualité — clarté, structure, caractère actionnable pour un dev senior — est *subjective* et ne se teste pas avec une simple assertion. Pour itérer sur les prompts de génération sans relire manuellement chaque sortie, on utilise un juge LLM en **éval offline, hors chemin critique**. En revanche, le cœur du différenciateur du projet — chaque affirmation cite et prouve sa source — repose sur une vérification **déterministe** (présence d'URL, exactitude de version, validité du lien), jamais déléguée au juge.

## Le principe

Un « LLM-as-a-judge » est un grand modèle de langage à qui l'on demande d'évaluer une sortie produite par un autre système (ou par lui-même), selon des critères décrits dans un prompt. Il remplace ou réduit le travail d'annotation humaine pour des dimensions difficiles à mesurer par du code : pertinence, ton, cohérence, qualité rédactionnelle.

Trois formats de verdict existent. Le **scoring direct** (note de 1 à 5 ou 1 à 10) est tentant mais piégeux : les annotateurs ne savent pas distinguer un « 3 » d'un « 4 », les scores intermédiaires masquent l'incertitude, et la magnitude n'est ni calibrée ni comparable dans le temps (Hamel Husain ; Liu et al., COLM 2024). Le **verdict binaire pass/fail** force une décision nette, plus facile à aligner sur un expert et à auditer. Le **pairwise** (« A est-il meilleur que B ? », avec option d'égalité) est le plus stable et le mieux aligné sur l'humain pour les tâches subjectives, car comparer est plus facile que noter dans l'absolu (Liu et al. ; Eugene Yan).

Un juge n'est utile que s'il est **aligné** sur le jugement humain. Le processus de référence (Hamel Husain) : (1) un *expert principal unique* de domaine pose les pass/fail de référence et rédige des **critiques** détaillées de chaque cas ; (2) ces critiques sont réinjectées en few-shot dans le prompt du juge (« critique shadowing ») ; (3) on mesure l'accord juge↔expert sur un jeu *held-out*, puis on fait de l'error analysis et on recommence. L'objectif réaliste est d'atteindre **>90 % d'accord** en quelques itérations (exemple Honeycomb : trois itérations ; cas applied-llms : 68 %→94 %).

Point crucial : le juge est fiable sur l'**ordinal/relatif** (A>B), peu fiable sur la **magnitude** (un 7,8/10 n'est pas une distance interprétable). On ne suit donc jamais un delta de score absolu comme métrique produit (Eugene Yan).

## Exemples concrets

**1. Éval offline « qualité de synthèse de migration React 18→19 » (juge binaire).**
On sépare clairement les deux natures de vérification :

```python
# (A) Factualité = DÉTERMINISTE, dans le chemin critique
def verifier_synthese(synthese):
    assert toutes_les_citations_resolvent(synthese.citations)      # HTTP 200 + domaine officiel
    assert re.search(r"\b18\.\d+\b.*\b19\.\d+\b", synthese.texte)   # versions présentes
    assert chaque_affirmation_a_une_source(synthese)               # schéma respecté
    # -> si KO : on rejette, on ne génère pas

# (B) Qualité subjective = JUGE LLM, offline, asynchrone, hors gating
PROMPT_JUGE = """Tu évalues une synthèse de migration pour un dev senior.
Raisonne d'abord (clarté ? structure ? actionnable ?), PUIS conclus par PASS ou FAIL.
La longueur n'est PAS un critère de qualité."""
verdict = juge_llm(PROMPT_JUGE, synthese, few_shot=critiques_expert_react)
```

On mesure le juge sur ~100 synthèses *held-out* avec **TPR et TNR séparés** (pas seulement le % d'accord brut), et on reporte le **Cohen's κ** (Eugene Yan).

**2. Régression de prompt en pairwise A/B.**
À chaque évolution du pipeline, on rejoue N changelogs réels (React 18.0→18.2, 18→19) et on demande au juge « ancien prompt vs nouveau ». Pour neutraliser le biais de position, **chaque paire est jugée deux fois en inversant l'ordre** ; le gagnant n'est attribué qu'après swap, avec option « égalité » autorisée (Applied LLMs). Le swap peut renverser plus de 10 % des verdicts.

**3. Test anti-biais d'autorité (lié au différenciateur « confiance »).**
On injecte dans une synthèse une fausse citation d'apparence autoritaire (faux lien `react.dev`). On vérifie que le **check déterministe la rejette** — démontrant que la factualité ne dépend pas du juge (qui, lui, serait *hackable* par cette même fausse citation ; cf. CALM, GPT-4-Turbo retourné par biais d'autorité).

## Subtilités & pièges à éviter

- **Ne jamais confier la factualité au juge.** Les LLM ont un **recall faible** sur la détection d'incohérences factuelles (≈30–60 % des résumés incohérents détectés, malgré >95 % de précision sur les corrects) : ils ratent les défauts subtils (Eugene Yan). De plus, le biais d'autorité les rend attaquables par de fausses citations (CALM).
- **L'échelle de Likert traitée comme métrique continue** : scores non calibrés, distribution décalée vs annotations humaines, inter-annotateurs incohérents. Préférer pass/fail ou pairwise (Hamel/Shankar).
- **Biais de position** : juger A vs B une seule fois dans un ordre fixe est une erreur ; GPT-3.5 montre ~50 %, Claude-v1 ~70 % de préférence positionnelle (Eugene Yan). Toujours faire le swap.
- **Biais de verbosité** : le juge préfère la réponse la plus longue **>90 % du temps**, même sans info additionnelle (Eugene Yan). Égaliser les longueurs et l'expliciter dans le prompt.
- **Biais de self-preference** : faire générer *et* juger par le même modèle sur la même sortie gonfle le win-rate (**+10 % GPT-4, +25 % Claude-v1**, Eugene Yan). Lié à la perplexité : le juge favorise le « familier », à faible perplexité (Self-Preference Bias paper). Anonymiser les modèles ; éviter producteur=juge sur la même sortie. *Acceptable* si la tâche du juge diffère (évaluer ≠ générer).
- **% d'accord trompeur** : avec classes déséquilibrées (peu de « fail »), 80 % d'accord peut cacher un κ=0,62 et un recall faible sur la classe rare (Eugene Yan). Toujours regarder le recall sur les défauts.
- **Pairwise ≠ panacée** : coût quadratique (n² comparaisons), impraticable pour ranker beaucoup de candidats sans échantillonnage/aggregation (PairS ; Liu et al.). Et sur les tâches *objectives*, son avantage s'évapore (cohérence factuelle : 0,47 pairwise vs 0,46 direct) — donc inutile pour la factualité du projet.
- **CoT à double tranchant** : demander un raisonnement avant le verdict améliore la fiabilité et permet d'auditer (Eugene Yan), mais peut aussi *rationaliser* un verdict biaisé (biais CoT, CALM) — d'où l'audit humain des critiques.
- **Évaluateurs fine-tunés** : ce sont des classifieurs spécialisés ; forte corrélation entre eux mais chute catastrophique sur un autre schéma de notation. Ne pas les réutiliser hors cadre (Eugene Yan).
- **Pass-rate à 100 %** : signe d'un jeu de test trop facile, pas d'un système parfait. Durcir le test plutôt qu'optimiser le score (Hamel/Shankar).
- **Complexité prématurée** : empiler des juges spécialisés avant d'avoir fait le critique shadowing et l'alignement de base avec l'expert.
- **Prompt de juge pauvre** : sans critiques, critiques trop laconiques, sans contexte externe, ou sans exemples diversifiés — les erreurs de prompt les plus fréquentes (Hamel).

## Checklist d'application

- [ ] Le juge LLM est **hors chemin critique** (async, post-génération), pas en gating bloquant.
- [ ] La factualité (citations, versions, présence/validité des sources, code de migration) est vérifiée par **assertions déterministes**, pas par le juge.
- [ ] Le verdict est **binaire pass/fail** (ou **pairwise** si une notation relative est requise), jamais une échelle Likert traitée comme continue.
- [ ] Un **expert de domaine principal unique** a posé les pass/fail de référence et rédigé des critiques.
- [ ] Les critiques expert sont réutilisées en **few-shot** dans le prompt du juge, avec exemples diversifiés.
- [ ] Le prompt demande le **raisonnement (CoT) avant** le verdict.
- [ ] Le prompt précise que **la longueur n'est pas un critère** ; les réponses comparées sont de longueur égalisée.
- [ ] Les comparaisons pairwise sont **jouées deux fois avec swap d'ordre** ; l'égalité est autorisée.
- [ ] Les modèles sont **anonymisés** ; le producteur n'est pas juge de sa propre sortie sur la même tâche.
- [ ] L'accord est mesuré sur un **held-out** avec **TPR + TNR séparés** et **Cohen's κ** reporté.
- [ ] Au moins **~100 exemples labellisés** servent à valider/aligner le juge.
- [ ] Un **humain reste dans la boucle** : échantillonnage régulier et ré-alignement quand la distribution change.
- [ ] On se méfie d'un **pass-rate trop élevé** : on durcit le test au lieu de viser 100 %.

## 10 questions / réponses

**Q1. C'est quoi un LLM-as-a-judge, en deux mots ?**
Un LLM à qui l'on demande d'évaluer la qualité d'une sortie (la sienne ou celle d'un autre système) selon des critères donnés dans un prompt. Il sert surtout à automatiser une partie de l'annotation humaine sur des dimensions subjectives difficiles à tester par du code (Hamel Husain).

**Q2. Pourquoi ne pas s'en servir pour vérifier que les sources et versions sont correctes ?**
Parce que les LLM ont un *recall* faible sur les défauts factuels (≈30–60 % des incohérences détectées) et sont attaquables par biais d'autorité — une fausse citation bien formulée peut retourner leur verdict (Eugene Yan ; CALM). La factualité doit être déterministe (regex, schéma, lookup d'URL), surtout pour une plateforme dont la valeur *est* la citation fiable.

**Q3. Note de 1 à 5 ou pass/fail ?**
Pass/fail. Les scores intermédiaires (un « 3 ») ne veulent rien dire de stable, ne se comparent pas dans le temps et corrèlent mal avec le jugement expert. Le binaire force une décision nette, plus facile à aligner et à auditer (Hamel Husain ; Hamel/Shankar).

**Q4. Et si j'ai besoin de comparer deux variantes ?**
Utilise le *pairwise* (« A vs B », avec option d'égalité) plutôt qu'un score absolu : c'est plus stable et mieux aligné sur l'humain pour les tâches subjectives (Liu et al., COLM 2024 ; Eugene Yan).

**Q5. Comment aligner concrètement le juge sur mon expert ?**
Un expert principal unique pose les pass/fail de référence et rédige des critiques détaillées ; ces critiques sont réinjectées en few-shot dans le prompt (« critique shadowing »). On mesure l'accord sur un held-out, on fait de l'error analysis, on itère. Viser >90 % d'accord en quelques tours (Hamel Husain ; Honeycomb).

**Q6. Pourquoi le simple « % d'accord » ne suffit pas ?**
Parce qu'avec des classes déséquilibrées (peu de « fail »), un accord brut élevé peut masquer un recall médiocre sur les défauts. Exemple : 80 % d'accord mais seulement κ=0,62. Il faut reporter TPR et TNR séparément et le Cohen's κ (Eugene Yan).

**Q7. Quels biais dois-je absolument neutraliser ?**
Au minimum trois : **position** (jouer chaque paire dans les deux ordres ; le swap peut renverser >10 % des verdicts), **verbosité** (le juge préfère le plus long >90 % du temps — égaliser les longueurs), et **self-preference** (+10 à +25 % de win-rate pour ses propres sorties — anonymiser, éviter producteur=juge) (Eugene Yan ; Applied LLMs).

**Q8. Puis-je utiliser le même modèle comme générateur et comme juge ?**
Oui *si* la tâche du juge diffère réellement (évaluer une synthèse ≠ la produire), mais c'est risqué dès qu'il évalue ses propres sorties : le self-preference, lié à la perplexité, le pousse vers ce qui lui est familier (Hamel/Shankar ; Self-Preference Bias paper). En cas de doute, anonymise et croise les modèles.

**Q9. Le pairwise est-il toujours meilleur ?**
Non. Son coût est quadratique (n² comparaisons), impraticable pour ranker beaucoup de candidats sans échantillonnage ou méthodes d'agrégation comme PairS. Et sur les tâches objectives, son avantage disparaît (cohérence factuelle : 0,47 vs 0,46 en direct) (Liu et al. ; Eugene Yan).

**Q10. Que faut-il prévoir comme effort et combien d'exemples ?**
Au moins ~100 exemples labellisés pour valider/aligner un juge, plus une maintenance récurrente. On ne construit un juge coûteux que pour des problèmes sur lesquels on itère vraiment, et on garde un humain dans la boucle pour échantillonner et ré-aligner quand le système ou la distribution change (Hamel/Shankar).

## Sources

1. [Hamel Husain — Creating a LLM-as-a-Judge That Drives Business Results](https://hamel.dev/blog/posts/llm-judge/) — *praticien* — processus d'alignement en 7 étapes (expert principal, critiques, shadowing), argument binaire pass/fail vs Likert, exemple Honeycomb (>90 % d'accord en 3 itérations).
2. [Hamel Husain & Shreya Shankar — LLM Evals: Everything You Need to Know (2026)](https://hamel.dev/blog/posts/evals-faq/) — *praticien* — seuil ~100 labels, TPR/TNR sur held-out, juge hors chemin critique (async) vs gating en prod, méfiance vis-à-vis des pass-rates à 100 %.
3. [Eugene Yan — Evaluating the Effectiveness of LLM-Evaluators (aka LLM-as-Judge)](https://eugeneyan.com/writing/llm-evaluators/) — *praticien* — synthèse de ~24 papiers : verbosité >90 %, self-enhancement GPT-4 +10 %/Claude +25 %, position 50–70 %, recall factuel 30–60 %, κ vs % d'accord (80 % / κ=0,62).
4. [What We've Learned From A Year of Building with LLMs (Applied LLMs)](https://applied-llms.org/) — *praticien* — conseils opérationnels : égaliser la longueur, double comparaison avec swap d'ordre, autoriser le tie, CoT, régression sur exemples de prod (68 %→94 %).
5. [Liu et al. — Aligning with Human Judgement: The Role of Pairwise Preference in LLM Evaluators (COLM 2024)](https://arxiv.org/abs/2403.16950) — *académique* — la préférence pairwise s'aligne mieux que le scoring direct ; la calibration seule ne suffit pas ; introduit PairS.
6. [Ye et al. — Justice or Prejudice? Quantifying Biases in LLM-as-a-Judge (CALM)](https://llm-judge-bias.github.io/) — *académique* — taxonomie de 12 biais (position, verbosité, self-enhancement, authority, bandwagon, distraction…) et attaques automatisées « hackant » un juge (citations factices = biais d'autorité).
7. [Self-Preference Bias in LLM-as-a-Judge](https://arxiv.org/pdf/2410.21819) — *académique* — quantifie le self-preference et le lie à la perplexité : le juge favorise les sorties familières (faible perplexité), y compris les siennes.
