# Monolithe modulaire & bounded contexts (DDD)

> **En une phrase.** Construis d'abord un **monolithe modulaire** organisé par domaine métier (bounded contexts), avec des frontières logiques gratuites et *enforced* par outillage — puis n'extrais un service physique que contre un gain **mesurable** (latence, coût, scaling, sécurité), jamais par esthétique.

## Pourquoi c'est nécessaire pour TechWatch IA

TechWatch IA est un MVP exploratoire : on ne connaît pas encore les vraies frontières entre Détection, Crawling, Synthèse, Vérification et Publication. Figer ces frontières derrière un réseau dès le jour 1 (microservices) reviendrait à parier sur un découpage qu'on n'a pas encore validé, tout en payant le coût de la distribution sans le bénéfice. La stratégie retenue — **un monolithe modulaire + 2-3 workers** — permet de découvrir les frontières en codant, d'isoler *seulement* les deux coutures qui le justifient vraiment (le **Crawling**, I/O bloquant + surface SSRF à sandboxer ; la **Génération IA**, latence/coût + rate-limiting propre), et de garder tout le reste in-process. Nos « innovation tokens » vont au vrai différenciateur — la synthèse sourcée et vérifiable — pas à l'orchestration distribuée.

## Le principe

**« Modulaire » et « distribué » sont deux axes orthogonaux.** On peut — et on doit souvent — avoir une forte modularité *sans* distribution. C'est la thèse centrale de Shopify et de Dan McKinley : les frontières d'un système ne requièrent pas d'appels réseau. Une frontière logique coûte presque rien (un dossier, une interface, une règle de lint) ; une frontière physique coûte cher (sérialisation, cohérence éventuelle, déploiements multiples, debugging distribué, sécurité inter-services).

**MonolithFirst (Fowler).** On ne peut pas découper proprement un système qu'on ne comprend pas encore. Mieux vaut commencer par un monolithe — soigneusement modulaire — pour *découvrir* les frontières, puis décomposer graduellement. Le refactoring d'une fonctionnalité entre deux modules est trivial dans un monolithe (une PR) ; entre deux services, c'est un projet. Un mauvais module se redécoupe ; un mauvais microservice se renégocie.

**Organiser par domaine, pas par couche.** Le geste clé de la *componentization* Shopify a été de réassigner ~6000 classes par domaine métier (orders, billing, shipping…) plutôt que par couche technique (controllers/models). Chaque composant expose une **API publique explicite** et garde la **propriété exclusive de ses données** : pas de jointure cross-module, pas d'association traversante vers les tables internes d'un autre module.

**Bounded Context (DDD, Fowler).** Là où le **vocabulaire change**, une frontière apparaît. Le même mot recouvre des modèles différents selon le contexte (la *polysémie*). « La total unification du modèle de domaine n'est ni faisable ni rentable » : on préfère des modèles par contexte + une **couche anti-corruption** (traduction) plutôt qu'un modèle unique partagé qui recréerait du couplage.

**Couture prête-à-extraire (seams, Newman).** Dès le jour 1, on appelle les modules extractibles via une **interface (port)** avec des **DTO sérialisables** — même quand l'appel est in-process. Le jour où on passe en worker/queue, on change l'implémentation *derrière* le port, pas les appelants.

## Exemples concrets

**1. Carte des modules de TechWatch IA**

```
            in-process (frontières LOGIQUES)
   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │  Detection   │──▶│  Synthesis   │──▶│ Verification │──▶ Publishing
   │ (releases)   │   │ (migration)  │   │ (citations)  │
   └──────┬───────┘   └──────┬───────┘   └──────────────┘
          │ port             │ port
          ▼                  ▼
   ╔══════════════╗   ╔══════════════╗
   ║   Crawling   ║   ║ Génération IA║   coutures PRÊTES-À-EXTRAIRE
   ║ (I/O, SSRF)  ║   ║ (latence/coût)║  → appel via interface + DTO,
   ╚══════════════╝   ╚══════════════╝     bascule queue sans toucher l'appelant
```

Chaque module possède ses données ; les flèches `──▶` pleines sont des appels in-process directs, les `║` doubles signalent une couture sérialisable extractible en worker.

**2. Polysémie de « source » (React 18 → 19)**

Le même mot, deux modèles selon le contexte — d'où l'anti-corruption layer :

```
# Bounded context CRAWLING
Source = { url: "https://react.dev/blog/...react-19",
           http_status: 200,
           robots_hash: "a1b2…",
           fetched_at: "2026-06-23T…" }

# Bounded context SYNTHESIS  (après traduction)
Source = { claim: "useEffect double-invoke en StrictMode",
           anchor: "#strict-mode-double-rendering",
           confidence: 0.92 }
```

Forcer un `Source` unifié partagé recréerait du couplage sémantique entre Crawling et Synthesis. On traduit l'un en l'autre à la frontière.

**3. ADR — pourquoi extraire la Génération IA mais PAS la Vérification**

```
Décision : Génération IA = worker (queue) ; Vérification = in-process.
Critère IA     : p95 latence > 8 s ET besoin de rate-limit/coût propre
                 → scaling indépendant justifié (cf. storefront Shopify).
Critère Vérif. : CPU léger, pas d'I/O externe, change avec Synthesis
                 → couplage de domaine ⇒ l'extraire ajouterait du réseau
                   au couplage sans bénéfice. Reste in-process.
Seuil de revue : ré-évaluer si file d'attente IA > 50 jobs en p95.
```

## Subtilités & pièges à éviter

- **Les frontières logiques sont quasi gratuites ; les physiques coûtent cher.** Ne payer le coût de la distribution (réseau, sérialisation, cohérence éventuelle, debug distribué) que contre un bénéfice concret.
- **Sans enforcement automatique, la frontière n'existe pas.** Convention seule = `big ball of mud` à terme : sous la pression d'une deadline, un import vers l'interne d'un autre module passe et la modularité se dissout silencieusement (constat explicite Fowler/Govind).
- **Découvrir les bons bounded contexts est difficile, même pour des experts** (Fowler). D'où l'intérêt de commencer monolithe : le coût d'un mauvais découpage reste celui d'une PR.
- **Extraire un service ne supprime pas le couplage de domaine.** Si deux modules changent toujours ensemble, les séparer physiquement ne fait qu'ajouter de la latence réseau au couplage. Le bon signal d'extraction est l'**indépendance d'évolution/scaling**, pas la propreté du diagramme.
- **L'isolation runtime ne nécessite pas forcément un service.** Shopify isole les tenants par *database pods*, pas par découpage de service. Pour nous, sandboxer le Crawling peut passer par un process/worker isolé plutôt qu'un microservice complet.
- **Une couture « prête-à-extraire » impose dès le jour 1 des DTO sérialisables et zéro état partagé caché.** Sinon l'extraction « facile » se révèle un gros refactoring le moment venu.
- **Piège — démarrer en microservices (`distributed monolith`)** : on fige des frontières inconnues et tout refactoring devient transverse (dénoncé par Fowler et Newman).
- **Piège — découper par couche technique** (un service « API », un « DB-access », un worker « utils ») : chaque feature traverse tous les services → couplage maximal.
- **Piège — partager les tables entre modules** (jointures cross-module, FKs partout) : c'est le couplage le plus dur à défaire et rend toute extraction future quasi impossible.
- **Piège — sur-extraire** : transformer chaque module logique en service. Complexité distribuée maximale pour une charge MVP qui tiendrait sur un seul process (gaspillage d'innovation tokens).
- **Exception au MonolithFirst** : si l'équipe a *déjà* une expérience forte du domaine et des frontières évidentes, démarrer multi-services peut se justifier. Pour un MVP exploratoire comme TechWatch IA, le monolithe modulaire reste le pari sûr.

## Checklist d'application

- [ ] Modules nommés par **domaine métier** (Detection, Crawling, Synthesis, Verification, Publishing), pas par couche technique.
- [ ] Chaque module expose une **API publique** explicite ; l'interne n'est jamais importé directement par un autre.
- [ ] **Propriété exclusive des données** par module : aucune jointure ni FK cross-module.
- [ ] Frontières placées **là où le vocabulaire change** (polysémie « source », « version »…) + anti-corruption layer aux jonctions.
- [ ] **Règle de lint/CI** qui échoue si un module importe l'interne d'un autre (équivalent Packwerk/Wedge).
- [ ] Les deux coutures extractibles (**Crawling**, **Génération IA**) appelées via **port + DTO sérialisables**, même in-process.
- [ ] **Un seul** déploiement + datastore au départ ; pas plus de 2-3 workers.
- [ ] Critère d'extraction **mesurable** documenté en ADR (latence p95, coût, taille de file, scaling, isolation SSRF).
- [ ] **Communication asynchrone** (events/queue) prévue aux coutures extractibles pour absorber les pics de latence IA.
- [ ] **Ownership** documenté : un domaine = un propriétaire clair (même en solo).
- [ ] Métriques de santé : score d'isolation par module, nombre de violations cross-module en CI, *blast radius* d'un changement.

## 10 questions / réponses

**Q1 — C'est quoi un monolithe modulaire ?**
Un seul déployable (un process, un datastore) découpé *à l'intérieur* en modules à frontières strictes, organisés par domaine métier. On obtient l'essentiel des bénéfices de découplage des microservices sans la complexité réseau (Shopify Engineering).

**Q2 — Pourquoi ne pas démarrer directement en microservices ?**
Parce qu'on ne connaît pas encore les vraies frontières : on les fige derrière un réseau et tout refactoring devient transverse. Fowler recommande « MonolithFirst » — découvrir les frontières en codant avant de payer le « MicroservicePremium » (martinfowler.com/bliki/MonolithFirst.html).

**Q3 — Modulaire et distribué, c'est la même chose ?**
Non, ce sont deux axes orthogonaux. On peut avoir une forte modularité *sans* aucune distribution : les frontières logiques ne requièrent pas d'appels réseau (Shopify ; McKinley).

**Q4 — Qu'est-ce qu'un bounded context concrètement ?**
Une zone où un terme a un sens unique et cohérent. Là où le vocabulaire change, une frontière apparaît : « source » côté Crawling (URL + statut HTTP) n'est pas le même objet que « source » côté Synthesis (citation vérifiable). Fowler : la total unification du modèle « n'est ni faisable ni rentable » (martinfowler.com/bliki/BoundedContext.html).

**Q5 — Pourquoi interdire les jointures cross-module ?**
Parce que partager les tables est le couplage le plus dur à défaire : il rend toute extraction future quasi impossible et tout changement de schéma transverse. Chaque module doit posséder exclusivement ses données et exposer une API (Shopify Engineering).

**Q6 — Comment empêcher l'érosion des frontières ?**
Par l'outillage, pas la discipline. Une règle de lint/CI (façon Packwerk/Wedge) doit échouer dès qu'un module importe l'interne d'un autre. Sans cela, sous la pression des deadlines, les raccourcis « urgents » dissolvent la modularité (Fowler/Govind, linking-modular-arch.html).

**Q7 — Quand extraire un module en service physique ?**
Seulement contre un signal *mesurable* : latence, coût, scaling indépendant ou isolation de sécurité. Chez Shopify, le *storefront rendering* est extrait parce qu'il est étroit + très haut trafic (gain de ressources massif), pas par esthétique (Milanović, Inside Shopify's Modular Monolith).

**Q8 — Pourquoi extraire la Génération IA mais pas la Vérification dans le projet ?**
La Génération IA a une latence/coût propres et un besoin de rate-limiting → scaling indépendant justifié. La Vérification est légère et évolue avec la Synthèse : l'extraire ajouterait du réseau à un couplage de domaine existant. Extraire un service ne supprime pas le couplage de domaine (Newman, Monolith to Microservices).

**Q9 — Que sont les « innovation tokens » et en quoi est-ce lié ?**
McKinley : « chaque entreprise dispose d'environ **trois** innovation tokens », à dépenser sur le cœur métier, pas sur l'infra. Choisir un monolithe modulaire + long-context (zéro vector DB) économise ces tokens pour le vrai différenciateur — la traçabilité/vérifiabilité des affirmations (mcfunley.com/choose-boring-technology).

**Q10 — La modularité, ça rapporte vraiment, chiffres à l'appui ?**
Oui, sur un effort continu. L'étude de cas de Fowler/Govind rapporte un cycle time passé de 17 à 10,3 jours en moyenne (**~-40 %**) et un temps d'intégration de **90 jours à 5 jours** (×18). Mais l'article avertit : la modularité demande un effort soutenu et ne doit cibler que de vrais problèmes métier (martinfowler.com/articles/linking-modular-arch.html).

## Sources

1. [MonolithFirst — Martin Fowler](https://martinfowler.com/bliki/MonolithFirst.html) — *praticien reconnu* — principe de construire un monolithe d'abord pour découvrir les frontières avant de payer le « MicroservicePremium » ; refactoring trivial dans un monolithe vs douloureux entre services.
2. [BoundedContext — Martin Fowler](https://martinfowler.com/bliki/BoundedContext.html) — *praticien reconnu (DDD)* — référence canonique du bounded context, polysémie, et « la total unification du modèle de domaine n'est ni faisable ni rentable ».
3. [Linking Modular Architecture to Development Teams — Martin Fowler (Govind et al.)](https://martinfowler.com/articles/linking-modular-arch.html) — *praticien reconnu (étude de cas)* — encapsulation/abstraction/optionalité, érosion des frontières, lien Conway, équipe platform ; résultats chiffrés ~-40 % cycle time (17 → 10,3 j), intégration 90j → 5j.
4. [Deconstructing the Monolith — Shopify Engineering](https://shopify.engineering/deconstructing-monolith-designing-software-maximizes-developer-productivity) — *doc d'ingénierie (étude de cas)* — componentization par domaine (~6000 classes réassignées), API publique par composant, propriété exclusive des données, outillage Wedge en CI.
5. [Inside Shopify's Modular Monolith — Milan Milanović](https://newsletter.techworld-with-milan.com/p/inside-shopifys-modular-monolith) — *praticien (synthèse technique)* — signaux d'extraction (storefront rendering : narrow + très haut trafic), isolation par database pods, Packwerk/Tapioca, frontières logiques vs séparation physique.
6. [Choose Boring Technology — Dan McKinley](https://mcfunley.com/choose-boring-technology) — *praticien reconnu* — « innovation tokens » (~3 par entreprise), et « les coûts long terme de maintenir un système fiable dépassent largement les inconvénients de construction ».
7. [Monolith to Microservices — Sam Newman](https://samnewman.io/books/monolith-to-microservices/) — *praticien reconnu (livre de référence)* — décomposition incrémentale, recherche des « seams », bounded contexts comme premiers candidats de services, garder le nombre de services minimal.
