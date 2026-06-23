# Adéquation échelle/complexité (YAGNI, anti-over-engineering)

> **En une phrase.** On n'ajoute une techno que lorsqu'une douleur réelle et mesurée la rend incontournable ; le reste du temps, on choisit l'ennuyeux et bien compris, on reporte l'abstraction, et on assume que le risque dominant du projet est l'over-engineering — pas le sous-dimensionnement.

## Pourquoi c'est nécessaire pour TechWatch IA

TechWatch IA est une plateforme de veille au trafic **sporadique**, dont la valeur tient à un seul différenciateur : la **confiance** (chaque affirmation cite sa source vérifiable). Tout l'effort doit donc se concentrer là — évals, guardrails, sourcing — et nulle part ailleurs. Empiler vector DB, fine-tuning, microservices ou cluster permanent « parce que c'est l'archi standard » consommerait notre budget d'attention et d'exploitation sur de la plomberie, au détriment du cœur de valeur. Ce document pose la règle de gouvernance : **prouver la douleur avant d'ajouter la complexité.**

## Le principe

L'idée centrale : **chaque techno non conventionnelle a un coût caché qui se paie en exploitation, pas en construction.** Dan McKinley formalise cela avec les *innovation tokens* — une entreprise dispose d'« environ trois » jetons à dépenser sur des choix technos non éprouvés, et ce stock ne grandit qu'avec la maturité (McKinley). On réserve ces jetons au différenciateur (ici la confiance/sourcing) et on prend du **« boring »** partout ailleurs. « Boring » ne veut pas dire obsolète : c'est une techno dont « les capacités *et surtout* les modes de défaillance sont bien compris » (McKinley) — Postgres, Python, un monolithe modulaire. Le nouveau, lui, traîne des *unknown unknowns* dont le coût réel apparaît en production : « les coûts long terme pour garder un système fiable dépassent de loin les inconvénients rencontrés pendant le build » (McKinley).

La règle d'or avant tout ajout : **« comment résoudrais-je ce problème sans rien ajouter de nouveau ? »** (McKinley). La réponse est rarement « impossible » ; elle est le plus souvent « faisable mais pénible » — et ce seuil de pénibilité doit être *mesuré*, pas imaginé.

Martin Fowler complète avec **YAGNI** (*You Aren't Gonna Need It*) : ne pas construire de capacité pour une feature future *présumée*. Une telle feature porte quatre coûts : **build** (effort pour une feature inutile), **delay** (valeur/feedback perdus en la priorisant), **carry** (la complexité ralentit tout le reste) et **repair** (on l'avait mal devinée six mois plus tôt) (Fowler). Distinction cruciale : « YAGNI s'applique aux capacités, pas à l'effort pour rendre le logiciel facile à modifier » (Fowler) — refactoring, tests, CI/CD et évals **ne sont pas** du YAGNI à fuir ; ce sont eux qui rendent YAGNI *sûr*, car ils maintiennent bas le coût de changer plus tard.

Côté produit, on **fait des choses qui ne scalent pas** au début (Graham) : curation manuelle des sources, validation humaine des synthèses. Côté LLM, on monte les paliers un par un — prompt engineering + long-context **avant** vector DB ou fine-tuning, chaque palier coûtant « un ordre de grandeur » d'effort de plus (applied-llms). Et on retire toujours plus vite une complexité qu'on a refusée d'ajouter qu'une mauvaise abstraction déjà incrustée.

## Exemples concrets

**Exemple 1 — Le corpus React 18→19 tient dans le contexte : on n'ajoute rien.**
La douleur qui justifierait une vector DB (corpus dépassant la fenêtre) n'existe pas encore : notes de version + guide d'upgrade + changelog tiennent en long-context.

```text
# Décision MVP
Question : "Comment résoudre la synthèse de migration SANS vector DB ?"
Réponse  : charger les 3 docs officiels dans le prompt (long-context).
           taille corpus ~ 40K tokens << fenêtre du modèle.
Conclusion : douleur absente => on n'ajoute RIEN.
Condition de sortie : corpus > ~200K tokens OU multi-tenant
                      => alors (et seulement alors) RAG.
```

**Exemple 2 — Détection de version « qui ne scale pas ».**
On démarre par un simple cron qui *poll* les changelogs/registries officiels (npm, blog React), pas un système d'événements/webhooks généralisé. On n'automatisera/étendra que quand le nombre de technos suivies rendra le polling pénible — signal mesuré, pas crainte de scale.

**Exemple 3 — Tableau des *innovation tokens* du projet (gouvernance légère).**

| Choix | Statut | Coût en tokens | Condition de sortie / déclencheur |
|---|---|---|---|
| Monolithe modulaire + workers | boring | 0 | Goulot de scale mesuré sur un module isolé |
| Serverless event-driven (pay-per-use) | boring | 0 | Volume soutenu rendant l'idle non pertinent |
| Long-context | boring | 0 | Corpus > fenêtre / multi-tenant |
| Vector DB / RAG | **token** | 1 | Corpus dépasse le long-context |
| Fine-tuning | **token** | 1 | Prompt prouvé insuffisant + évals en place |
| Multi-agents | **token** | 1 | Workflow déterministe prouvé insuffisant |

**Exemple 4 — Validation humaine en boucle.** Pour les premières synthèses React 19, un ingénieur relit et score *manuellement* chaque affirmation sourcée (Graham). Cette dette assumée alimente les évals — qui automatiseront plus tard le contrôle de fiabilité.

## Subtilités & pièges à éviter

- **YAGNI n'est pas « ne jamais penser au futur ».** Il s'oppose à *construire* pour un futur présumé, tout en exigeant un code malléable. C'est un arbitrage de design, pas un refus de design (Fowler).
- **YAGNI suppose un code malléable.** Dans un code rigide/non testé, reporter une décision devient coûteux : il faut investir dans tests/évals **avant** de pouvoir « se permettre » YAGNI (Fowler).
- **« Boring » ≠ obsolète.** C'est « contraint mais bien compris ». Choisir une techno récente est correct *si* on dépense sciemment un token là où c'est le différenciateur (McKinley).
- **Le coût réel = l'exploitation long terme** (monitoring, mises à jour, on-call), pas l'effort de build initial. Décider sur la TCO, pas la vitesse de démarrage (McKinley).
- **« Do things that don't scale » est temporaire.** Le manuel est une dette à rembourser au signal de PMF ; prévoir le **déclencheur d'automatisation**, sinon on reste bloqué (Graham).
- **Le serverless a un revers.** Pay-per-use est idéal pour un trafic sporadique mais peut devenir cher à fort volume soutenu, et le long-context coûte par token — le « least worst » dépend du profil de charge réel (AWS, applied-llms).
- **La sous-ingénierie volontaire ≠ négligence.** Sécurité, sourcing vérifiable et évals sont le cœur de valeur : ils ne se « YAGNI-isent » jamais. Distinguer plomberie reportable et différenciateur non négociable.
- **DRY vs YAGNI.** Dupliquer temporairement est souvent moins risqué qu'une abstraction prématurée mal devinée ; reporter l'extraction jusqu'au *rule of three* (Shpilt, Fowler).
- **Piège — vector DB / RAG par défaut** : ajouter l'ingestion, les embeddings et la sync pour un besoin non prouvé alors que le long-context suffit.
- **Piège — microservices / Kubernetes prématurés** : éclater un monolithe qui marche, multipliant coûts d'exploitation et modes de défaillance.
- **Piège — interfaces sans seconde implémentation** : un point d'extension jamais utilisé n'est pas neutre, il *gêne* l'évolution (Shpilt).
- **Piège — fine-tuning avant prompt engineering** ou **course au dernier modèle/GPU avant PMF** : « no GPUs before PMF » (applied-llms).
- **Piège — confondre YAGNI et abandon de qualité** : invoquer YAGNI pour zapper tests/évals/sourcing — exactement le différenciateur à ne pas couper.
- **Piège — over-provisioning « au cas où »** : quotas larges et instances permanentes jamais révisés, à contre-emploi du serverless (AWS).

## Checklist d'application

- [ ] Avant d'ajouter une techno : ai-je écrit **comment résoudre le problème sans rien ajouter** ? (McKinley)
- [ ] L'ajout dépense-t-il un *innovation token* — et est-ce sur le différenciateur (la confiance), pas sur la plomberie ?
- [ ] Ai-je une **douleur mesurée** (latence, coût/req, taux d'erreur instrumentés), pas une crainte de scale future ?
- [ ] Pour toute feature « au cas où » : ai-je pesé les **4 coûts** build/delay/carry/repair ? (Fowler)
- [ ] Ai-je vérifié que je ne sacrifie pas refactoring/tests/évals (qui rendent YAGNI sûr) ? (Fowler)
- [ ] Côté LLM : ai-je épuisé **prompt + long-context** avant d'envisager RAG/fine-tuning ? (applied-llms)
- [ ] Toute abstraction attend-elle le **3e cas d'usage réel** (rule of three) ?
- [ ] Build-vs-buy : est-ce une **commodité** (orchestration, file, stockage, parsing) que je devrais acheter plutôt que réimplémenter ?
- [ ] L'ajout est-il **documenté** : pourquoi le stack actuel est insuffisant + la condition de sortie/migration ?
- [ ] Le manuel/non-scalable a-t-il un **déclencheur d'automatisation** explicite ? (Graham)
- [ ] L'infra évite-t-elle l'**over-provisioning** (zéro coût à l'idle, pas de cluster permanent au MVP) ? (AWS)

## 10 questions / réponses

**Q1 — C'est quoi YAGNI, en une phrase ?**
« You Aren't Gonna Need It » : ne construis pas une capacité pour une feature future que tu *présumes* nécessaire. Tant que le besoin n'est pas réel, tu paies du code, de la complexité et du risque pour rien (Fowler).

**Q2 — Qu'est-ce qu'un *innovation token* ?**
Une métaphore de budget : une équipe dispose d'« environ trois » choix technos non conventionnels qu'elle peut se permettre, ce stock grandissant avec la maturité. On les réserve au cœur métier et on prend de l'« ennuyeux » partout ailleurs (McKinley).

**Q3 — « Boring », ça veut dire mauvais ou dépassé ?**
Non : « boring » signifie que les capacités **et surtout les modes de défaillance** sont bien compris (Postgres, Python). Une techno récente peut être le bon choix si on dépense sciemment un token là où c'est le différenciateur (McKinley).

**Q4 — Pourquoi le risque dominant ici est l'over-engineering et pas le sous-dimensionnement ?**
Parce qu'il est rapide d'ajouter de la complexité quand la douleur apparaît, mais lent et risqué de retirer une mauvaise abstraction déjà incrustée. On préfère donc une asymétrie en faveur du simple, quitte à ajouter plus tard.

**Q5 — Mais alors, le refactoring et les tests, ce n'est pas du YAGNI à éviter ?**
Au contraire. Fowler est explicite : « YAGNI s'applique aux capacités, pas à l'effort pour rendre le logiciel facile à modifier ». Refactoring, tests et CI/CD sont les *conditions* qui rendent YAGNI sûr, en maintenant bas le coût de changement (Fowler).

**Q6 — Comment décider d'ajouter une vector DB au projet ?**
On ne l'ajoute pas tant que le corpus tient en long-context. La condition de sortie est explicite : corpus dépassant la fenêtre, dynamisme fort ou multi-tenant. Avant ça, ce serait de l'ingestion/embeddings/sync pour un besoin non prouvé (applied-llms).

**Q7 — Pourquoi commencer par des choses qui ne scalent pas ?**
Parce que le manuel (curation des sources, relecture humaine des synthèses) est la boucle de feedback qui rend le produit *fiable*. C'est une dette volontaire et temporaire, à rembourser quand le signal de PMF arrive, avec un déclencheur d'automatisation prévu (Graham).

**Q8 — Quels sont les coûts d'une feature présumée qu'on construit « au cas où » ?**
Quatre, selon Fowler : **build** (effort gaspillé), **delay** (valeur/feedback retardés), **carry** (complexité qui ralentit toutes les features suivantes) et **repair** (rework quand l'hypothèse initiale était fausse) (Fowler).

**Q9 — En quoi le serverless event-driven sert le right-sizing ?**
La facturation à la requête (Lambda/Step Functions) donne un coût quasi nul à l'idle, ce qui colle à un trafic de veille sporadique et évite l'over-provisioning. Revers : à fort volume soutenu, le pay-per-use peut devenir cher — le bon choix dépend du profil de charge réel (AWS).

**Q10 — L'IA change-t-elle ces principes en 2025 ?**
Le principe tient et se renforce : l'IA est une techno particulièrement gourmande en *innovation tokens* à cause de ses *unknown unknowns* (Brethorst). Empiler vector DB + fine-tuning + multi-agents multiplie les inconnues ; long-context + workflow déterministe est le choix « boring » du moment — « focus on the system, not the model » (applied-llms).

## Sources

1. [Choose Boring Technology — Dan McKinley](https://mcfunley.com/choose-boring-technology) — *praticien reconnu (essai de référence)* — concept des *innovation tokens* (~3 par équipe), définition de « boring » (capacités **et** modes de défaillance bien compris), règle « résoudre le problème sans rien ajouter », et primauté du coût d'exploitation long terme sur le coût de build.
2. [Yagni (bliki) — Martin Fowler](https://martinfowler.com/bliki/Yagni.html) — *praticien reconnu (référence canonique)* — définition autoritative de YAGNI, les 4 coûts (build, delay, carry, repair), et la distinction « YAGNI vise les capacités présumées, pas l'effort de rendre le code modifiable ».
3. [Do Things that Don't Scale — Paul Graham](https://www.paulgraham.com/ds.html) — *praticien reconnu (essai de référence)* — justification du manuel/non-scalable au début (recrutement et sur-engagement avec les premiers users) comme boucle de feedback qui rend le produit bon.
4. [What We Learned from a Year of Building with LLMs (Part III: Strategy) — Yan, Bischof, Frye, Husain, Liu, Shankar](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-iii-strategy/) — *praticiens reconnus (applied-llms.org / O'Reilly)* — right-sizing des applis LLM : « no GPUs before PMF », « focus on the system, not the model », prompt engineering d'abord, chaque palier ≈ un ordre de grandeur de plus, build-vs-buy.
5. [Premature Infrastructure is the Root of All Evil — Michael Shpilt](https://michaelscodingspot.com/premature-infrastructure-is-evil/) — *praticien* — extension de la « premature optimization » à l'infra : interfaces sans seconde implémentation, patterns/hiérarchies prématurés, architectures pour features prédites jamais livrées ; renvoie à YAGNI + KISS et au refactoring continu.
6. [Cost Optimization Pillar — AWS Well-Architected, Serverless Applications Lens](https://docs.aws.amazon.com/wellarchitected/latest/serverless-applications-lens/cost-optimization.html) — *doc officielle (AWS)* — cadre de right-sizing de l'infra : facturation à la requête (zéro coût à l'idle), éviter l'over-provisioning, ajuster offre et demande sur du serverless event-driven.
7. [Choose Boring Technology, Revisited (2025) — Aaron Brethorst](https://www.brethorsting.com/blog/2025/07/choose-boring-technology,-revisited/) — *praticien (relecture récente 2025)* — réactualisation des *innovation tokens* à l'ère IA/LLM : le principe tient, et les nouvelles technos (dont l'IA) consomment encore plus de tokens à cause de leurs *unknown unknowns*.
