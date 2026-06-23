# Notifications : fan-out asynchrone & idempotence

> **En une phrase.** Quand une synthèse est publiée, TechWatch IA émet *un seul* événement métier `SummaryPublished`, puis des workers calculent les N destinataires et envoient les notifications — le tout conçu pour qu'un crash ne re-notifie jamais les abonnés déjà servis, grâce au pattern *transactional outbox* et à une idempotence portée par la clé `(summary_id, subscriber_id)`.

## Pourquoi c'est nécessaire pour TechWatch IA

Le différenciateur du projet, c'est la **confiance** : une synthèse de migration (React 18→19, etc.) doit arriver aux bonnes personnes, une fois, sans spam ni perte. Or notifier N abonnés depuis 1 synthèse est une opération longue, faillible et soumise aux pannes du provider d'email. Si on couple naïvement « publier la synthèse » et « envoyer les emails », un crash en plein fan-out laisse l'état incohérent : synthèse publiée mais personne notifié, ou pire, reprise qui re-notifie tout le monde. Découpler l'événement `SummaryPublished` du calcul des destinataires et de l'envoi, puis rendre chaque envoi **idempotent**, transforme ce processus fragile en pipeline fiable et rejouable.

## Le principe

Le point de départ est le problème du **dual-write** : on voudrait, en une opération atomique, *commiter la synthèse en base* ET *publier l'événement vers le broker*. Mais une base de données et un broker de messages sont deux systèmes distincts — il n'existe pas de transaction commune. Un crash entre les deux casse la cohérence. Le **transactional outbox pattern** résout ce dual-write : on écrit la synthèse **et** une ligne d'événement dans la **même transaction DB**. Si la transaction *rollback*, aucun événement n'est émis ; si le crash survient après le commit, un *relay* (poller ou CDC) lira l'outbox et publiera l'événement — il n'est jamais perdu (AWS Prescriptive Guidance).

Conséquence assumée : la livraison est **at-least-once**, pas **exactly-once**. L'outbox, SNS standard et SQS standard garantissent *au moins une fois*. Les doublons ne sont donc pas un bug à éliminer en transit, mais un tradeoff intégré : toute la fiabilité repose sur des **consommateurs idempotents**. C'est la combinaison « at-least-once côté émetteur + idempotence côté récepteur » qui *simule* l'exactly-once.

Côté découplage, le producteur **possède le schéma** de l'événement, le consommateur **possède la sémantique de l'effet** et doit être idempotent (AWS EDA). Concrètement : un worker « calcul destinataires » liste les N abonnés et crée N jobs ; un worker « envoi » appelle le provider. L'idempotence se matérialise par une table *inbox* / `processed_events` (insert-if-not-exists sur l'`event_id`) et par une clé **par destinataire** `(summary_id, subscriber_id)` avec contrainte `UNIQUE`. Ainsi, une reprise après crash ne touche **que** les abonnés non encore servis. Enfin, l'envoi réel doit gérer les pannes provider avec un **backoff exponentiel capé + jitter** (AWS Builders' Library, Stripe), des **retries bornés** basculant vers une **Dead-Letter Queue**, et — quand le provider le supporte — une **clé d'idempotence** propagée jusqu'à lui pour ne pas renvoyer un email déjà parti.

## Exemples concrets

**1. Outbox transactionnel : persister la synthèse + l'événement dans la même transaction**

```sql
BEGIN;
  INSERT INTO summaries (id, tech, version, body)
    VALUES ('sum-42', 'react', '19', '...');
  -- Même transaction : si ça rollback, aucun événement n'est émis.
  INSERT INTO outbox (id, aggregate_id, type, payload, occurred_on)
    VALUES ('evt-abc', 'sum-42', 'SummaryPublished',
            '{"summary_id":"sum-42","tech":"react"}', now());
COMMIT;
```

```sql
-- Relay : plusieurs workers en parallèle, sans double-traitement, dans l'ordre.
SELECT * FROM outbox
  WHERE published_at IS NULL
  ORDER BY occurred_on
  FOR UPDATE SKIP LOCKED
  LIMIT 100;
-- publier vers le broker, puis UPDATE outbox SET published_at = now() ...
```

`FOR UPDATE SKIP LOCKED` permet à plusieurs processeurs de piocher des lots disjoints ; `ORDER BY occurred_on` préserve l'ordre (Milan Jovanović).

**2. Consommateur idempotent (table `processed_events`) — le rejeu est un no-op**

```sql
-- L'événement SummaryPublished#evt-abc arrive deux fois (rejeu SQS).
INSERT INTO processed_events (event_id) VALUES ('evt-abc')
  ON CONFLICT (event_id) DO NOTHING;
-- Si 0 ligne insérée → déjà traité → on ne refait PAS le fan-out.
```

**3. Fan-out idempotent par destinataire — un crash reprend où il s'est arrêté**

Scénario React 18→19 : le worker « calcul destinataires » crée N jobs avec clé `(summary_id, subscriber_id)`. Le worker « envoi » crashe après l'abonné #3.

```sql
-- Clé d'unicité par destinataire : pose la preuve d'envoi atomiquement.
INSERT INTO notifications (summary_id, subscriber_id, status)
  VALUES ('sum-42', 'sub-7', 'sending')
  ON CONFLICT (summary_id, subscriber_id) DO NOTHING;  -- déjà servi → skip
```

À la reprise, les abonnés 1..3 ont déjà une ligne → ignorés ; seuls 4..N sont traités. **Aucun re-spam.**

**4. Envoi avec backoff + jitter et clé d'idempotence provider**

```python
# deduplication-id stable propagé jusqu'au provider quand il le supporte.
dedup_id = f"summary-{summary_id}:subscriber-{subscriber_id}"  # ex. summary-42:subscriber-7
for attempt in range(MAX_RETRIES):            # borné (5–8), sinon -> DLQ
    try:
        provider.send(to=email, idempotency_key=dedup_id, body=html)
        break
    except TransientError:
        delay = min(CAP, BASE * 2 ** attempt)  # backoff exponentiel capé
        sleep(delay + random.uniform(0, delay))  # + jitter (anti thundering herd)
```

Sur une panne Postmark touchant 500 notifications React 19, le jitter étale la reprise au lieu de re-saturer le provider à l'instant précis où il récupère.

## Subtilités & pièges à éviter

- **Idempotent ≠ exactly-once.** On n'élimine pas les doublons en transit : on rend leur *effet* sans conséquence. (AWS Prescriptive Guidance)
- **Un email parti n'est pas annulable.** Les *foreign state mutations* (envoi via SES/Postmark) ne sont pas *rollback-ables* : il faut persister la preuve d'envoi (message-id provider) atomiquement avec le marquage « envoyé », sinon un retry renvoie l'email. (brandur.org)
- **L'échec indéterminé est le piège central.** Un timeout / reset de connexion ne dit pas si l'email est parti. Sans clé d'idempotence côté provider, retenter risque le doublon, ne pas retenter risque la perte ; la clé lève l'ambiguïté. (Stripe)
- **Granularité de la clé = arbitrage.** Trop grossière (par synthèse) → un crash re-notifie tout le monde ; trop fine sans verrou → courses concurrentes. `(summary_id, subscriber_id)` + `UNIQUE` est le bon niveau.
- **Réutiliser une clé pour un contenu différent est une erreur, pas un succès silencieux** : Stripe vérifie que les paramètres rejoués correspondent à l'originale. (Stripe)
- **Backoff sans jitter = thundering herd** : tous les envois échoués retentent au même instant et re-tuent le provider. (AWS Builders' Library)
- **Amplification des retries** : ne retenter qu'à **une seule couche**. Retenter à chaque niveau (SDK + worker + queue) multiplie la charge (ex. 3 retries sur 5 couches = 243×). (AWS Builders' Library)
- **Pas de borne ni de DLQ** : un message empoisonné (adresse invalide, payload corrompu) retenté à l'infini bloque la file et brûle le quota provider — il faut l'isoler en dead-letter.
- **Clé purgée trop tôt → doublon possible** sur un retry tardif ; **jamais purgée → table qui enfle** et performances qui se dégradent. Définir une fenêtre de rétention.
- **Confier l'idempotence uniquement au provider** : SES n'offre pas de vraie clé d'idempotence d'envoi côté API comme Stripe. La déduplication doit d'abord vivre **côté application** (inbox + clé par destinataire).
- **Polling vs CDC** : le polling est simple mais ajoute latence et charge DB ; le CDC (log tailing, DynamoDB Streams) réduit la charge et capte l'ordre nativement, au prix de complexité opérationnelle.
- **L'ordre n'est garanti que si on le force** : les standard queues ne le préservent pas ; FIFO / sequence numbers oui, mais au prix du débit. Pour des notifications de veille, l'ordre *par abonné* suffit généralement.
- **Stores non-ACID** (ex. Mongo sans transactions) ne garantissent pas l'atomicité outbox : choisir un store ACID pour les tables outbox/inbox.
- **Jobs orphelins** (worker mort à mi-chemin) : prévoir un process *completer* qui pousse les états « in-progress » à terme, sinon ils y restent éternellement. (brandur.org)

## Checklist d'application

- [ ] La synthèse **et** la ligne `outbox` (`SummaryPublished`) sont écrites dans **la même transaction DB**.
- [ ] Un **relay** lit l'outbox avec `FOR UPDATE SKIP LOCKED` + `ORDER BY occurred_on` et marque les lignes publiées.
- [ ] Les consommateurs sont **idempotents** via une table `processed_events` / inbox (insert-if-not-exists sur `event_id`).
- [ ] Le fan-out utilise une clé **par destinataire** `(summary_id, subscriber_id)` avec contrainte `UNIQUE`.
- [ ] L'`event_id` / une clé de déduplication stable **voyage dans le payload** jusqu'au provider quand il le supporte.
- [ ] Le worker d'envoi applique **backoff exponentiel capé + jitter**.
- [ ] Les retries sont **bornés** (5–8) et **une seule couche** retente ; les échecs définitifs partent en **DLQ**.
- [ ] La preuve d'envoi (message-id provider) est persistée **atomiquement** avec le statut « envoyé ».
- [ ] Une **politique de rétention** purge les lignes outbox publiées et les clés d'idempotence après une fenêtre définie.
- [ ] Un **completer** récupère les jobs orphelins « in-progress ».
- [ ] L'`event_id` est **tracé de bout en bout** (producteur → broker → worker → provider) pour diagnostiquer doublons et latences.

## 10 questions / réponses

**Q1. C'est quoi le « fan-out » dans ce contexte ?**
C'est la diffusion d'**un** événement métier (1 synthèse publiée) vers **N** destinataires (les abonnés). Au lieu d'envoyer les N emails dans le même processus, on publie un événement `SummaryPublished`, puis des workers découplés calculent les destinataires et envoient. Le producteur possède le schéma de l'événement, le consommateur possède la sémantique de l'effet (AWS EDA).

**Q2. Pourquoi ne pas simplement « commiter la synthèse puis appeler le provider d'email » ?**
Parce que c'est un **dual-write naïf** : un crash entre le commit DB et l'appel provider laisse la synthèse publiée sans notification (ou l'inverse). La base et le provider sont deux systèmes sans transaction commune. Le transactional outbox corrige exactement ce trou (AWS Prescriptive Guidance).

**Q3. Comment fonctionne le transactional outbox concrètement ?**
On écrit la synthèse **et** une ligne d'événement dans la **même transaction**. Si elle rollback, aucun événement n'est émis. Après commit, un *relay* (polling de la table ou CDC sur le log) lit l'outbox et publie vers le broker. L'événement n'est donc jamais perdu et survit aux crashes (AWS Prescriptive Guidance).

**Q4. Pourquoi viser at-least-once plutôt qu'exactly-once ?**
L'exactly-once distribué est extrêmement coûteux, voire illusoire. L'outbox, SNS standard et SQS standard offrent **at-least-once** : des doublons sont possibles par conception. On les neutralise non pas en les empêchant, mais en rendant les consommateurs **idempotents** (AWS Prescriptive Guidance).

**Q5. Comment rend-on un consommateur idempotent ?**
On stocke l'`event_id` traité dans une table *inbox* / `processed_events` et on vérifie son existence **avant** d'appliquer l'effet (`INSERT ... ON CONFLICT DO NOTHING`). Si l'ID existe déjà, on no-op. On conserve ces IDs pendant une fenêtre de rétention définie (AWS Prescriptive Guidance, Milan Jovanović).

**Q6. Pourquoi la clé d'idempotence doit-elle être `(summary_id, subscriber_id)` et pas juste `summary_id` ?**
Parce que la granularité est un arbitrage. Une clé par synthèse marque la synthèse « notifiée » globalement : un crash en milieu de fan-out fait recommencer **tout** le monde au redémarrage et spamme les abonnés déjà servis. La clé **par destinataire** (+ contrainte `UNIQUE`) fait reprendre uniquement les abonnés 4..N (brandur.org).

**Q7. À quoi sert le jitter dans le backoff, et le backoff seul ne suffit-il pas ?**
Le backoff exponentiel capé espace les tentatives (`min(cap, 2^n)`). Mais si 500 envois échouent en même temps (panne provider), ils retentent **tous au même instant** : c'est le *thundering herd* qui re-sature le provider. Le jitter ajoute un délai aléatoire qui **étale** la reprise (AWS Builders' Library, Stripe).

**Q8. Qu'est-ce que l'« amplification des retries » et comment l'éviter ?**
Si chaque couche d'un appel (SDK provider + worker + queue) retente indépendamment, les retries se **multiplient** : 3 retries sur 5 couches = 243× la charge. La règle : ne retenter qu'à **une seule couche** de la stack, et désactiver les retries des autres (AWS Builders' Library).

**Q9. Que faire d'un message qui échoue toujours (adresse invalide, payload corrompu) ?**
Le retenter à l'infini bloque la file et brûle le quota provider. Il faut **borner** le nombre de tentatives (typiquement 5–8) puis basculer le message vers une **Dead-Letter Queue** pour inspection et replay manuel, libérant la file pour le reste.

**Q10. L'idempotence côté provider remplace-t-elle l'idempotence côté application ?**
Non. Tous les providers n'offrent pas de vraie clé d'idempotence d'envoi (Stripe oui via `Idempotency-Key` ; SES non au niveau API). De plus, un échec indéterminé (timeout) ne dit pas si l'email est parti — d'où une clé côté provider quand c'est possible. Mais la déduplication doit **d'abord** vivre côté application (inbox + clé par destinataire) ; la clé provider est une protection complémentaire (Stripe, brandur.org).

## Sources

1. [Transactional outbox pattern — AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html) — *official-docs* — résolution du dual-write, consommateurs idempotents, ordre via timestamp/sequence, non-envoi en cas de rollback, et les deux implémentations (table pollée vs CDC/streams).
2. [Designing robust and predictable APIs with idempotency — Stripe](https://stripe.com/blog/idempotency) — *practitioner* — clés d'idempotence côté client (header `Idempotency-Key`), mise en cache du résultat (succès comme échec) pour rejouer la même réponse, backoff exponentiel + jitter contre le thundering herd.
3. [Implementing Stripe-like Idempotency Keys in Postgres — brandur.org](https://brandur.org/idempotency-keys) — *practitioner* — foreign state mutations, phases atomiques, recovery points, staging transactionnel des notifications, verrou (`locked_at`), `SERIALIZABLE` et process *completer* pour les requêtes orphelines.
4. [Timeouts, retries, and backoff with jitter — Amazon Builders' Library](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/) — *official-docs* — ne retenter que si l'opération est idempotente, backoff capé, jitter, bornage des tentatives, et éviter l'amplification des retries (retenter à une seule couche).
5. [Best practices for implementing event-driven architectures — AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/best-practices-for-implementing-event-driven-architectures-in-your-organization/) — *official-docs* — découplage producteur/broker/consommateur, propriété du schéma vs sémantique de l'effet, idempotence des consommateurs, observabilité centralisée.
6. [Implementing the Outbox Pattern — Milan Jovanović](https://www.milanjovanovic.tech/blog/implementing-the-outbox-pattern) — *practitioner* — polling outbox avec `SELECT ... FOR UPDATE SKIP LOCKED`, batch sizing, `ORDER BY occurred_on`, déduplication côté consommateur (at-least-once) et alternative CDC (transaction log tailing).
