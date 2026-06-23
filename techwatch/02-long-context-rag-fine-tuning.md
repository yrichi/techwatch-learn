# Long-context, RAG ou fine-tuning : quand choisir quoi

> **En une phrase.** Commence par le prompting, puis mets tout le corpus dans le prompt tant qu'il tient (long-context) ; ne passe au RAG que quand le corpus devient gros, dynamique ou cloisonné, et ne fine-tune qu'en dernier recours pour figer un style ou une syntaxe — jamais pour injecter des faits.

## Pourquoi c'est nécessaire pour TechWatch IA

Le corpus d'une transition de version (notes de version, guide de migration, changelog d'une techno comme React 18→19) est **borné et statique** : il tient presque toujours sous le seuil où le RAG devient utile. Choisir le long-context « d'abord » nous évite une dette d'infrastructure (vector DB, chunking, embeddings à maintenir) qui n'améliorerait pas la qualité au MVP. Ce choix sert aussi notre différenciateur — la confiance : avec tout le corpus dans le prompt, chaque affirmation de la synthèse peut citer sa source exacte et rester vérifiable manuellement. Ce document fixe l'arbre de décision qui justifie « zéro vector DB au MVP ».

## Le principe

Trois approches répondent à trois besoins **différents** ; les confondre est l'erreur la plus coûteuse.

- **RAG (Retrieval-Augmented Generation)** change ce que le modèle **voit à l'instant T** : on récupère des documents pertinents et on les injecte dans le prompt. C'est l'outil des connaissances fraîches, de la traçabilité et du cloisonnement multi-tenant.
- **Le fine-tuning** change **comment le modèle se comporte** : style, format de sortie, syntaxe d'un langage de niche (un DSL, une structure de migration). Il n'apprend pas fiablement des **faits** (Husain).
- **Le long-context** = tout mettre dans le prompt, sans étape de recherche, quand le corpus est petit et borné.

L'arbre de décision par défaut suit une complexité croissante :

1. **Prompting + few-shot d'abord.** Mesure ce que ça donne avant toute infrastructure.
2. **Si le corpus tient sous ~200K tokens (≈500 pages), long-context + prompt caching.** Anthropic est explicite : sous ce seuil, « inutile de faire du RAG », on met toute la base dans le prompt ; le prompt caching réduit la latence de plus de 2× et le coût jusqu'à 90 % (Anthropic).
3. **Si le corpus est gros, dynamique ou multi-tenant, RAG.** Le coût d'inférence du long-context croît **linéairement** avec la longueur : tout inclure sur du volume devient économiquement absurde (applied-llms).
4. **Fine-tuning seulement en dernier**, pour le style/la syntaxe d'un domaine de niche, jamais pour des faits frais.

La **règle des 90 %** encadre la dernière marche : si le prompting t'amène à 90 % du résultat, le fine-tuning ne vaut probablement pas l'investissement (applied-llms). Et un prérequis dur : impossible de fine-tuner efficacement sans système d'éval (Husain). Enfin, ce qui compte n'est pas la longueur brute du contexte mais sa **pertinence** : un long contexte bruité dégrade la réponse (lost-in-the-middle). Choisis l'approche la plus simple qui passe les évals, et itère — réserve la complexité au moment où la mesure prouve qu'elle est nécessaire.

## Exemples concrets

**Exemple 1 — MVP TechWatch React 18→19 (long-context, zéro vector DB).**
On agrège trois documents officiels bornés directement dans le prompt :

```text
[PRÉFIXE CACHÉ — corpus stable, réutilisé sur toutes les requêtes]
<source id="blog-react19">Blog officiel React 19 (texte intégral)</source>
<source id="migration">Guide de migration officiel (texte intégral)</source>
<source id="changelog">Changelog React 19 (texte intégral)</source>

[PARTIE VARIABLE — la question]
Produis une synthèse de migration. Pour CHAQUE affirmation, cite
la source via son id et la section, ex. [migration §"useRef"].
```

Le préfixe (le corpus) est mis en cache : on ne paie le plein tarif qu'une fois, puis on fait varier seulement la question. La sortie structurée force la traçabilité — chaque ligne renvoie à `id + section`, ce qu'on peut vérifier à la main contre le contexte fourni.

**Exemple 2 — Scénario de repli RAG (si un corpus explose, ex. migration d'un gros framework backend).**
On bascule seulement quand la mesure le justifie. Chunking par **unité sémantique** (1 entrée de changelog = 1 chunk), contexte préfixé (`version + module`), recherche **hybride** :

```text
candidats = RRF( BM25(query, ~top-100), embeddings(query, ~top-100) )
top20     = rerank(candidats)[:20]   # qui mérite la fenêtre de contexte ?
réponse   = LLM(query + top20)
```

Le BM25 capte les tokens exacts — `useEffect`, `react-dom/client`, les numéros de version — là où les embeddings échouent.

**Exemple 3 — Contre-exemple fine-tuning.** On NE fine-tune PAS : la connaissance change à chaque release (se périmerait), on a besoin de citations vérifiables, et on n'a pas (encore) d'évals de domaine. Le fine-tuning ne serait envisagé que pour **figer le FORMAT** de la synthèse, et seulement avec un harnais d'éval en place (Husain).

## Subtilités & pièges à éviter

- **Le seuil de ~200K tokens dépend du modèle et évolue.** Des fenêtres de 1M+ existent, mais « tenir techniquement » n'est pas « optimal » : la pertinence prime sur la longueur (lost-in-the-middle, dilution de l'attention).
- **Long-context et RAG ne sont pas exclusifs.** On peut faire un retrieval grossier (sélectionner 3 docs pertinents) puis tout mettre en long-context. Le **RAG par résumés** est un point milieu efficace — il égale quasiment le long-context, alors que le chunking naïf décroche (Li et al. 2025).
- **Coût linéaire vs coût fixe.** Le long-context coûte linéairement par requête ; le RAG a un coût fixe d'infra (index, embeddings, reranker) + maintenance. Sur faible volume avec caching, le long-context est souvent moins cher tout compris.
- **Choix selon la NATURE de la requête (Li et al.) :** long-context fort en QA factuel structuré sur corpus stable ; RAG fort en dialogue / requêtes ouvertes. Pas seulement la taille du corpus.
- **Piège — sauter au RAG par réflexe** alors que le corpus tient en contexte : complexité prématurée qui ralentit l'itération sans gagner en qualité.
- **Piège — fine-tuner pour injecter des faits** : anti-pattern ; le modèle n'apprend pas fiablement les faits, ils se périment, et on perd la traçabilité.
- **Piège — tout-embeddings sans BM25** : on échoue précisément sur versions, acronymes et IDs, les tokens critiques d'une veille techno.
- **Piège — chunking à longueur fixe sans contexte** : « le hook a été supprimé » sans dire quel hook ni quelle version est inutilisable. Préférer unités sémantiques + contexte préfixé.
- **Piège — lancer RAG/fine-tuning sans évals** : sans mesure on ne justifie ni l'investissement ni la détection de régression.
- **Piège — bourrer la fenêtre « au cas où »** : plus de contexte n'est pas mieux ; du contexte *pertinent* l'est.
- **Piège — oublier le prompt caching** : refacturer le même corpus plein tarif à chaque requête rend le long-context faussement « trop cher ».
- **Piège — empiler reranking + contextual retrieval + hybride sur un MVP à petit corpus** : optimiser un pipeline RAG sophistiqué avant d'avoir prouvé qu'un simple long-context échoue.

## Checklist d'application

- [ ] J'ai d'abord testé un **prompt simple + few-shot** et mesuré le résultat.
- [ ] J'ai estimé la taille du corpus en tokens ; **< ~200K** → long-context (Anthropic).
- [ ] Le corpus stable est mis en **prompt caching** (préfixe), seule la question varie.
- [ ] La sortie est **structurée pour citer ses sources** (id + section), vérifiable à la main.
- [ ] Je n'ajoute du RAG **que si** le corpus dépasse le budget, devient dynamique ou multi-tenant.
- [ ] Si RAG : chunking par **unité sémantique** + contexte préfixé (version + module).
- [ ] Si RAG : recherche **hybride BM25 + embeddings** fusionnée par RRF (pour versions/acronymes/IDs).
- [ ] Si RAG : **reranking** activé seulement si les évals montrent que le top-k brut rate des chunks.
- [ ] Je n'envisage le **fine-tuning** que pour le style/format, **et** avec un harnais d'éval en place.
- [ ] Chaque décision d'ajout de complexité est **justifiée par une mesure** (évals), pas par réflexe.

## 10 questions / réponses

**Q1. Quelle est la différence de fond entre RAG, fine-tuning et long-context ?**
Le RAG change ce que le modèle *voit* (faits, traçabilité) ; le fine-tuning change *comment* il se comporte (style, syntaxe, format) ; le long-context met tout le corpus dans le prompt quand il est petit et borné. Ne pas confondre injection de connaissances et apprentissage de comportement (applied-llms).

**Q2. Par quoi commencer concrètement ?**
Par le prompting et le few-shot, en mesurant le résultat avant toute infra. C'est la séquence de complexité croissante : prompting → long-context → RAG → fine-tuning (O'Reilly Radar).

**Q3. À partir de quelle taille de corpus le RAG devient-il utile ?**
Sous ~200K tokens (≈500 pages), inutile : on met toute la base dans le prompt. Au-delà, on passe à une solution de retrieval scalable (Anthropic).

**Q4. Pourquoi le long-context est-il pertinent pour une transition de version ?**
Parce que le corpus (blog officiel, guide de migration, changelog) est borné et statique, donc tient typiquement sous le seuil. Le long-context bat souvent le RAG sur le QA factuel d'un corpus stable (Li et al. 2025).

**Q5. Le long-context ne coûte-t-il pas trop cher ?**
Son coût croît linéairement avec la longueur, mais le prompt caching réduit la latence de plus de 2× et le coût jusqu'à 90 % en mettant le corpus stable en préfixe caché et en ne variant que la question (Anthropic). Sur un corpus réutilisé, c'est souvent moins cher qu'un RAG complet.

**Q6. Quand le RAG l'emporte-t-il franchement ?**
Quand la connaissance change vite, exige des citations/traçabilité difficiles à tenir en long-context, ou doit être cloisonnée (multi-tenant via index séparés). Aussi sur les requêtes ouvertes / le dialogue (Li et al. 2025).

**Q7. Pourquoi ne pas se reposer uniquement sur les embeddings ?**
Parce que les vector embeddings ne résolvent pas magiquement la recherche : ils échouent sur les noms, acronymes, IDs et numéros de version. On combine BM25 (matchs exacts) + embeddings (synonymes, fautes) fusionnés par Reciprocal Rank Fusion (O'Reilly Radar).

**Q8. Qu'est-ce que le « contextual retrieval » et quand l'utiliser ?**
On préfixe chaque chunk d'un court contexte (~50–100 tokens) généré par le LLM avant l'embedding, pour le rendre auto-suffisant. Les gains s'empilent : Contextual Embeddings −35 % d'échecs, +BM25 −49 %, +reranking −67 % (5,7 % → 1,9 %), pour ~1,02 $/M tokens avec caching (Anthropic). Réservé aux gros corpus réutilisés — sur un petit corpus borné, c'est de l'over-engineering.

**Q9. Quand le fine-tuning est-il légitime, et quand ne l'est-il pas ?**
Légitime pour ancrer un format de sortie ou une syntaxe de niche (Honeycomb query language, ReChat Lucy) ; illégitime pour injecter des faits frais (peu fiable, se périme, perte de traçabilité). Et règle dure : impossible de fine-tuner efficacement sans système d'éval (Husain).

**Q10. Comment éviter de sur-architecturer le pipeline ?**
Choisir l'approche la plus simple qui passe les évals et n'ajouter de la complexité (RAG hybride, reranking) qu'au moment où la mesure prouve qu'elle est nécessaire. Pas de vector DB tant que le corpus tient en contexte ; le reranking et le contextual retrieval sont des arbitrages qualité/latence/coût, pas des gains gratuits (applied-llms).

## Sources

1. [Introducing Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval) — *official-docs* — fixe le seuil de décision (~200K tokens ≈ 500 pages, sous lequel le RAG est inutile), les gains du prompt caching (latence −2×, coût −90 %) et la pile RAG chiffrée (Contextual Embeddings −35 %, +BM25 −49 %, +reranking −67 %, 5,7 %→1,9 %).
2. [Enhancing RAG with contextual retrieval — Claude Cookbook](https://platform.claude.com/cookbook/capabilities-contextual-embeddings-guide) — *official-docs* — implémentation de référence (code) : génération auto du contexte par chunk (~1,02 $/M tokens avec caching), fusion embeddings + BM25 par Reciprocal Rank Fusion, reranking.
3. [What We Learned from a Year of Building with LLMs](https://applied-llms.org/) — *practitioner* — règle des « 90 % » du fine-tuning, coût linéaire du long-context vs retrieval, et les trois dimensions de qualité de retrieval (pertinence MRR/NDCG, densité, récence) + recherche hybride.
4. [What We Learned from a Year of Building with LLMs (Part I) — O'Reilly Radar](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-i/) — *practitioner* — version éditée : « commencer par le prompting avant RAG/fine-tuning », « les embeddings ne résolvent pas magiquement la recherche », recommandation BM25 + hybride pour noms/acronymes/IDs.
5. [Is Fine-Tuning Still Valuable? — Hamel Husain](https://hamel.dev/blog/posts/fine_tuning_valuable.html) — *practitioner* — contrepoint nuancé : le fine-tuning sert la syntaxe/style d'un domaine de niche, pas l'injection de faits frais ; prérequis dur « impossible de fine-tuner efficacement sans système d'éval ».
6. [Long Context vs. RAG for LLMs: An Evaluation and Revisits (Li et al., 2025)](https://arxiv.org/abs/2501.01880) — *academic* — étude comparative : long-context fort en QA factuel/Wikipedia, RAG fort en dialogue/requêtes ouvertes ; le retrieval par résumés égale le long-context, le chunking naïf décroche ; rôle central de la pertinence du contexte.
