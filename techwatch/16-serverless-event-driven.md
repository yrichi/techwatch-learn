# Serverless event-driven & scale-to-zero

> **En une phrase.** Pour une charge épisodique et tolérante à la latence comme TechWatch IA, on déclenche des workers serverless par événement (event-based compute) câblés en architecture découplée (event-driven), de sorte qu'on ne paie rien entre deux releases et que le cold start occasionnel reste négligeable face à une synthèse qui dure plusieurs secondes.

## Pourquoi c'est nécessaire pour TechWatch IA

TechWatch ne tourne pas en continu : il dort jusqu'à ce qu'une nouvelle version sorte (React 18→19), puis doit produire une synthèse de migration. Cette charge est **bursty et imprévisible** — quelques dizaines de synthèses par mois, concentrées autour des releases — ce qui est exactement le profil où le serverless gagne : zéro coût à l'idle, scale automatique au pic. Et comme personne n'attend une synthèse en 200 ms (on tolère plusieurs secondes, voire minutes), on peut laisser les workers **scale-to-zero** et accepter le cold start occasionnel plutôt que de payer un serveur allumé 24/7. Le découplage event-driven sert aussi la fiabilité du projet : chaque étape du workflow 9 étapes communique par événements versionnés et tracés, condition d'une donnée vérifiable.

## Le principe

Il faut distinguer deux axes **orthogonaux** souvent confondus (Alex DeBrie) :

- **Event-driven architecture** = *comment les services communiquent* : de façon asynchrone et découplée, via des événements. Le producteur (« React 19 est sorti ») ne connaît pas le consommateur et n'attend pas de résultat précis.
- **Event-based compute** = *comment le compute est provisionné* : une instance (ex. AWS Lambda) n'existe que pour traiter un événement, puis disparaît. Tout usage de Lambda est event-based compute, mais ce n'est pas forcément event-driven.

Une architecture event-driven se structure en **3 parties** (AWS Well-Architected) : **event sources** (le déclencheur : un poller détecte une release), **event routers** (qui acheminent : EventBridge route par règles, SNS fait du fan-out, SQS bufferise), et **event destinations** (les workers de synthèse). On ne couple jamais producteur et consommateur par appel API direct.

Choisir le bon transport est central :

- **EventBridge** : bus event-driven multi-consommateurs, routage par règles. Ajoute un peu de latence (peu importe ici).
- **SNS** : fan-out vers plusieurs abonnés indépendants.
- **SQS** : file point-à-point, **message-driven** (le producteur cible un consommateur précis). Idéale comme **tampon** devant un worker pour absorber les bursts et respecter le rate limit de l'API LLM.
- **API Gateway** : request-response synchrone, à réserver à l'UI temps réel — surtout pas devant une synthèse LLM.

Le **scale-to-zero** signifie qu'aucune instance ne persiste : les workers doivent être **stateless** (état externalisé dans S3/DynamoDB), **idempotents** (livraison at-least-once → tout événement peut être rejoué), et protégés par **retries + Dead Letter Queue**. Le **cold start** (phase INIT : provisioning, runtime, code, dépendances) affecte moins de 1 % des requêtes en régime établi (AWS), mais beaucoup plus sur un trafic intermittent — sans gravité ici.

## Exemples concrets

**1. Pipeline TechWatch de bout en bout**

```
[cron/poller] détecte React 19
      │  publie un événement
      ▼
[EventBridge]  event "release.detected" { releaseId, techno:"react", version:"19.0.0" }
      │  règle de routage
      ▼
[SQS]  tampon : lisse le burst, respecte le rate limit de l'API LLM
      │  consommé une à la fois
      ▼
[Worker scale-to-zero]  exécute le workflow 9 étapes de synthèse
      │  écrit le résultat
      ▼
[S3 / DynamoDB]  synthèse sourcée + notification
```

Entre deux releases, **aucune ressource n'est allumée** : coût d'idle nul.

**2. Idempotence au rejeu (pseudocode du handler)**

```python
def handle(event):
    key = f"{event['releaseId']}:{event['version']}"   # clé d'idempotence
    if store.exists(key):                              # déjà traité ?
        return  # doublon → on ignore, pas de second appel LLM facturé
    store.put(key, status="processing")
    synthese = run_workflow_9_etapes(event)            # appels LLM
    store.put(key, status="done", result=synthese)
```

Si l'événement `release.detected` pour React 19 est livré deux fois (at-least-once), une seule synthèse est produite : pas de double facturation LLM.

**3. Cold start vs durée de synthèse**

Un worker Node « à froid » paie ~300–800 ms d'INIT ; « à chaud » l'init disparaît. Rapporté à une synthèse qui prend plusieurs secondes à plusieurs minutes, le cold start est **négligeable** — justification chiffrée du choix scale-to-zero plutôt que provisioned concurrency.

## Subtilités & pièges à éviter

- **Message-driven ≠ event-driven** : SQS est asynchrone mais le producteur cible un consommateur précis ; ce n'est pas le découplage d'EventBridge (Alex DeBrie). Utile pour chaîner des workers, à ne pas confondre.
- **Scale-to-zero n'est gratuit qu'au repos** : sous burst, le scale-up déclenche de **multiples cold starts simultanés** — le pire moment. Le tampon SQS sert justement à lisser ce pic.
- **Cold start < 1 % en régime établi, mais bien plus en trafic intermittent** (AWS) : une techno veillée rarement = quasi toujours à froid. Sans gravité ici car personne n'attend 200 ms.
- **Provisioned concurrency = anti-pattern pour du batch tolérant** : elle élimine le cold start mais réintroduit le coût du toujours-allumé — on paierait pour ne pas attendre alors qu'on accepte d'attendre.
- **EventBridge ajoute de la latence** : Well-Architected recommande SNS/SQS quand la latence est critique. Ici, peu importe.
- **At-least-once rend l'idempotence non-optionnelle** : sans clé d'idempotence, un rejeu = double synthèse + double coût LLM.
- **Limite dure de 15 min** : une synthèse long-context sur gros corpus peut la dépasser → découper en étapes (Step Functions) plutôt qu'une fonction monolithique.
- **Piège — API synchrone < 200 ms devant le worker** : cold start + temps LLM = timeout. La génération doit être asynchrone (déclenchée, résultat poll/notifié).
- **Piège — couplage par appels API directs en chaîne** (A attend B attend C) : la disponibilité s'effondre et la latence s'additionne. Découpler par bus/queue.
- **Piège — omettre DLQ/retries** : un événement empoisonné (source 404, schéma invalide) bloque la file ou se rejoue à l'infini, et perd silencieusement des releases.
- **Piège — pool de connexions DB dans la fonction** : sous burst, des centaines d'instances épuisent les connexions et font tomber la base. Utiliser un proxy (RDS Proxy) ou un store sans connexion persistante (DynamoDB).
- **Piège — dépendances lourdes / runtime compilé** pour un worker rarement appelé : cold start aggravé là où il pèse déjà le plus.
- **Piège — polling/setTimeout pour « attendre »** une étape : on paie du compute à ne rien faire. Remplacer par un déclencheur d'événement / Step Functions / cron.

## Checklist d'application

- [ ] Charge confirmée **épisodique/bursty** et **tolérante à la latence** (sinon, réévaluer serverless vs always-on).
- [ ] Producteurs et consommateurs **découplés** par bus/queue, jamais par appel API direct synchrone.
- [ ] **Tampon SQS** devant les workers pour absorber les bursts et respecter le rate limit LLM.
- [ ] Chaque handler **idempotent** via clé `releaseId + version`.
- [ ] **Retries + DLQ** configurés sur chaque consommateur.
- [ ] Workers **stateless** : état externalisé (S3 / DynamoDB), aucune connexion DB persistante dans la fonction.
- [ ] Cold start réduit **par le code d'abord** : runtime interprété, tree-shaking, lazy-loading, libs inutiles supprimées.
- [ ] **Mémoire tunée** (AWS Lambda Power Tuning) : arbitrer coût vs latence d'init/exécution.
- [ ] **Pas de provisioned concurrency** sur les workers batch ; SnapStart si runtime Java/Python/.NET.
- [ ] **Contrats d'événements versionnés** (JSONSchema / OpenAPI 3 via EventBridge Schema Registry).
- [ ] **Tracing distribué** (X-Ray / OpenTelemetry) bout-en-bout.
- [ ] Étapes longues (> 15 min) **découpées** en orchestration (Step Functions).
- [ ] **Point de bascule de coût** calculé (pay-per-use vs toujours-allumé) avant de figer l'archi.

## 10 questions / réponses

**Q1. C'est quoi le serverless en une phrase ?**
Un modèle où l'on exécute du code en réponse à des événements sans gérer de serveur, facturé à l'usage et capable de descendre à zéro instance quand il n'y a rien à faire. On ne provisionne pas de capacité à l'avance : la plateforme crée une instance par événement puis la détruit.

**Q2. Quelle différence entre event-driven et event-based compute ?**
Event-driven décrit *comment les services communiquent* (asynchrone, découplé, par événements) ; event-based compute décrit *comment le compute est provisionné* (une instance par événement, ex. Lambda). Ce sont deux axes orthogonaux : un worker peut être event-based sans que l'architecture soit pleinement event-driven (Alex DeBrie).

**Q3. Quelles sont les 3 parties d'une architecture event-driven ?**
Les event sources (déclencheurs, ex. une release détectée), les event routers (EventBridge, SNS, SQS qui acheminent), et les event destinations (les workers). On ne couple jamais directement source et destination (AWS Well-Architected).

**Q4. EventBridge, SNS ou SQS : lequel choisir ?**
EventBridge pour du routage event-driven multi-consommateurs par règles ; SNS pour du fan-out vers des abonnés indépendants ; SQS pour bufferiser en point-à-point (message-driven). Pour TechWatch, SQS sert de tampon devant les workers afin de lisser les bursts et respecter le rate limit de l'API LLM (Alex DeBrie).

**Q5. Pourquoi mettre une file SQS devant les workers ?**
Parce que sous burst, le scale-up déclenche plusieurs cold starts simultanés et risque de saturer l'API LLM en pic. La file absorbe l'à-coup et libère les messages à un rythme contrôlé, respectant les quotas du fournisseur plutôt que de tout lancer d'un coup.

**Q6. Qu'est-ce qu'un cold start et combien de requêtes affecte-t-il ?**
C'est le temps de la phase INIT (provisioning, runtime, chargement du code, dépendances) quand une nouvelle instance est créée. En régime établi il affecte moins de 1 % des requêtes, mais ce chiffre s'effondre sur un trafic intermittent — sans gravité ici car la synthèse n'est pas attendue en 200 ms (AWS Compute Blog).

**Q7. Comment réduit-on les cold starts concrètement ?**
D'abord par le code : runtime interprété (Node/Python s'initialisent vite), tree-shaking, lazy-loading, suppression des libs inutiles. Ensuite par le tuning mémoire — plus de mémoire = plus de CPU = init plus rapide. Enfin SnapStart (Java/Python/.NET) restaure un snapshot de l'environnement initialisé (AWS Compute Blog).

**Q8. Pourquoi ne PAS utiliser provisioned concurrency ici ?**
Parce qu'elle pré-chauffe des instances payées à l'heure même inutilisées : c'est utile pour un chemin à latence garantie, mais un anti-pattern pour du batch tolérant. On paierait du compute idle 24/7 pour éviter une attente qu'on accepte — autant laisser scale-to-zero (AWS Compute Blog).

**Q9. Pourquoi l'idempotence est-elle obligatoire ?**
Parce que la livraison en event-driven est at-least-once : tout événement peut être rejoué. Sans clé d'idempotence (`releaseId + version`), un rejou produirait une seconde synthèse et un second appel LLM facturé. La clé garantit qu'un événement déjà traité est ignoré.

**Q10. Quand le serverless devient-il un mauvais choix ?**
Pour une API synchrone à budget < 200 ms, des jobs > 15 min, des connexions persistantes, un débit soutenu, un debugging difficile ou des dépendances lourdes (Render). Au-delà d'un seuil de charge constante, le toujours-allumé devient moins cher : il faut calculer ce point de bascule avant de figer l'architecture.

## Sources

1. [Understanding and Remediating Cold Starts: An AWS Lambda Perspective](https://aws.amazon.com/blogs/compute/understanding-and-remediating-cold-starts-an-aws-lambda-perspective/) — *official-docs* — décomposition de la phase INIT, chiffre clé « < 1 % des requêtes affectées », comparaison provisioned concurrency vs SnapStart, et effet de la mémoire sur le cold start.
2. [Event-Driven Architectures vs. Event-Based Compute in Serverless Applications](https://www.alexdebrie.com/posts/event-driven-vs-event-based/) — *practitioner* — distinction event-driven (communication) vs event-based compute (provisionnement) ; classement EventBridge/SNS (event-driven) vs SQS (message-driven) vs API Gateway (request-response).
3. [Event-driven architectures — Serverless Applications Lens (AWS Well-Architected)](https://docs.aws.amazon.com/wellarchitected/latest/serverless-applications-lens/event-driven-architectures.html) — *official-docs* — les 3 parties (sources / routers / destinations), contrats de schéma (OpenAPI 3, JSONSchema via Schema Registry), tracing X-Ray, et l'avertissement sur la latence d'EventBridge.
4. [When to Avoid Using Serverless Functions](https://render.com/articles/when-to-avoid-using-serverless-functions) — *practitioner* — les six cas où serverless est un mauvais choix et le point de bascule de coût vers le toujours-allumé.
5. [Cold Start Latency in Serverless Computing: A Systematic Review, Taxonomy, and Future Directions](https://arxiv.org/html/2310.08437v2) — *academic* — taxonomie des mitigations cold start (cache/préchargement, runtime WASM/microVM/unikernel/conteneur, snapshot/checkpoint-restore, function fusion) et arbitrage startup vs isolation.
6. [Transitioning to event-driven architecture — AWS Serverless Developer Guide](https://docs.aws.amazon.com/serverless/latest/devguide/serverless-transition.html) — *official-docs* — couplage faible producteur/consommateur, communication asynchrone via bus/queues, déploiement indépendant des services.
