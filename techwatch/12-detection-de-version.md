# Détection de version (polling, ETags, webhooks)

> **En une phrase.** Pour savoir de façon fiable qu'une techno est passée de la v18 à la v19, on interroge périodiquement l'API des releases GitHub (polling cron), on évite de re-scanner inutilement grâce aux requêtes conditionnelles (ETag / `If-None-Match`), on compare la version reçue à un curseur persistant avec une vraie logique SemVer, et on garde les webhooks pour plus tard comme optimisation temps réel — jamais comme unique filet.

## Pourquoi c'est nécessaire pour TechWatch IA

TechWatch IA promet une synthèse de migration **fiable et sourcée** déclenchée au bon moment : il faut donc détecter une nouvelle version **sans la rater** et **sans la dupliquer**. La détection de version est l'étape 1 du workflow déterministe : c'est elle qui décide « il y a un vrai delta (React 18 → 19), je lance la génération » plutôt que de brûler du quota et du budget LLM sur du bruit. Comme le différenciateur du produit est la confiance, on privilégie une approche robuste (polling cron + curseur, modèle Dependabot/Renovate) où l'on peut prouver qu'aucune release n'a été manquée, plutôt qu'une approche temps réel fragile.

## Le principe

L'idée de base : un **worker planifié (cron)** interroge à intervalle régulier l'endpoint des releases d'un dépôt GitHub, par exemple `GET /repos/{owner}/{repo}/releases`. C'est exactement le modèle d'un bot de mise à jour de dépendances comme Renovate, qui s'exécute sur planning et interroge les registres, sans détection temps réel par défaut ([Renovate Docs](https://docs.renovatebot.com/bot-comparison/)).

Trois mécanismes rendent ce polling économe et fiable :

1. **Requêtes conditionnelles (ETag / Last-Modified).** À chaque réponse, GitHub renvoie un en-tête `ETag` (et `Last-Modified`). On le stocke, puis on le renvoie au poll suivant via `If-None-Match` (ou `If-Modified-Since`). Si rien n'a changé, GitHub répond `304 Not Modified` **sans corps** — et, point capital, **un 304 ne consomme pas le rate limit primaire** quand on est authentifié ([GitHub best practices](https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api)). On ne « paie » donc que les vrais changements.

2. **Authentification.** Sans token : 60 requêtes/heure. Avec un token : **5 000 requêtes/heure**. Pour surveiller beaucoup de technos, une GitHub App donne un quota par installation (base 5 000, jusqu'à 12 500 req/h) ([Rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)).

3. **Curseur + SemVer.** On persiste par techno un curseur : `{ dernier tag SemVer vu, etag, release_id }`. Quand la réponse contient un tag **supérieur** au curseur selon la précédence SemVer 2.0.0 (et non une comparaison de chaînes), on déclenche la synthèse, puis on avance le curseur. SemVer définit `MAJOR.MINOR.PATCH`, ignore le build metadata pour l'ordre, et place une pré-release avant la release correspondante (`1.0.0-rc.1 < 1.0.0`) ([SemVer 2.0.0](https://semver.org/)).

Les **webhooks** (événement `release`, actions `published`/`created`/`edited`) sont une optimisation ultérieure : temps réel, légers, sans coût de quota — mais ils peuvent être manqués, donc on les combine toujours à un polling de réconciliation ([Webhook events](https://docs.github.com/en/webhooks/webhook-events-and-payloads)).

## Exemples concrets

**1. Détecteur React (polling conditionnel + curseur).**

```ts
// Cron horaire. Curseur persistant par techno.
type Cursor = { lastTag: string; etag: string | null; releaseId: number | null };

async function pollReact(cursor: Cursor) {
  const res = await fetch(
    "https://api.github.com/repos/facebook/react/releases?per_page=100",
    {
      headers: {
        Authorization: `Bearer ${TOKEN}`,        // 5 000 req/h au lieu de 60
        Accept: "application/vnd.github+json",
        ...(cursor.etag ? { "If-None-Match": cursor.etag } : {}),
      },
    }
  );

  if (res.status === 304) return cursor;          // rien de neuf, 0 quota consommé

  const releases = await res.json();              // triées par date, page de 100 max
  const newer = releases
    .filter(r => !r.draft && semverGt(r.tag_name, cursor.lastTag)) // vraie compa SemVer
    .sort((a, b) => semverCmp(a.tag_name, b.tag_name));

  for (const r of newer) {
    if (r.id === cursor.releaseId) continue;      // idempotence : déjà traité
    enqueueMigrationSynthesis(r);                 // ex. v18.3.1 -> v19.0.0 => MAJOR
  }

  const top = newer.at(-1);
  return top
    ? { lastTag: top.tag_name, etag: res.headers.get("ETag"), releaseId: top.id }
    : { ...cursor, etag: res.headers.get("ETag") };
}
```

**2. Démonstration du gain ETag.** Surveiller 50 dépôts toutes les 5 min = 600 req/h. Sans cache, les 600 comptent. Avec `If-None-Match`, seuls les ~10 % réellement modifiés (≈ 60) consomment du quota ; le reste renvoie `304` gratuit. On le prouve en regardant `x-ratelimit-remaining` avant/après ([GitHub best practices](https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api)).

**3. Pré-releases React.** `GET /repos/facebook/react/releases/latest` renvoie la dernière release **stable** (non-prerelease, non-draft) et ignore donc une RC de React 19. Pour anticiper la migration avant la GA, on utilise `List releases` et on filtre `prerelease === true` pour capter `19.0.0-rc.1` ([Releases API](https://docs.github.com/en/rest/releases/releases)).

**4. Webhook + réconciliation.** Un webhook `release.published` donne la détection temps réel ; un cron quotidien rejoue `List releases` depuis le curseur pour rattraper toute livraison manquée.

## Subtilités & pièges à éviter

- **ETag vs Last-Modified.** Un praticien Renovate/Dependabot a observé que l'ETag GitHub peut changer alors que la ressource est identique (faux `200`), tandis que `Last-Modified` reste stable. `If-Modified-Since` peut être plus fiable pour ne pas gaspiller de quota ([Jamie Magee](https://jamiemagee.co.uk/blog/making-the-most-of-github-rate-limits/)).
- **Cache ETag par token et par page.** Un nouveau token d'installation de GitHub App (TTL ≤ 1 h) invalide les ETags en cache : prévoir le « cold start » du cache au renouvellement de token.
- **`/releases/latest` trie par `created_at`**, pas par date de publication : une release rétro-datée ou re-taguée peut tromper la logique « la plus récente ». Croiser avec le tag SemVer plutôt que se fier à l'ordre de l'API ([Releases API](https://docs.github.com/en/rest/releases/releases)).
- **Releases ≠ tags Git.** Un tag sans release associée n'apparaît PAS dans l'API Releases. Si une techno ne publie que des tags, poller l'API Tags (ou `git ls-remote --tags`).
- **Comparaison lexicographique = bug.** `'1.0.0-alpha10'` passe avant `'1.0.0-alpha2'`, et `'9.0.0'` après `'10.0.0'`. Utiliser une bibliothèque SemVer conforme 2.0.0 ; le build metadata (`+xxx`) doit être totalement ignoré pour l'ordre ([SemVer 2.0.0](https://semver.org/)).
- **Tags non conformes.** Beaucoup de projets utilisent `v18.2.0`, du CalVer, ou des suffixes maison : la normalisation doit être tolérante et **journaliser** les versions non parseables au lieu de planter.
- **Pré-releases.** Le curseur doit savoir s'il suit le canal stable seulement ou inclut les RC/canary/beta, sinon on rate (ou on spamme avec) les pré-releases.
- **Rate limit secondaire avant le primaire.** Les limites « 900 points/min » et « 90 s CPU / 60 s réel » peuvent se déclencher en burst même avec du quota horaire restant. Surveiller `retry-after`, pas seulement `x-ratelimit-remaining` ([Rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)).
- **Ne pas ignorer `retry-after` / `x-ratelimit-reset`.** Retenter en boucle serrée est vu comme un abus et peut faire bannir temporairement le token. Si `remaining == 0`, attendre `x-ratelimit-reset` (epoch UTC) ; si `retry-after` est présent, respecter exactement ce délai.
- **Pas de fan-out massif.** Maximum 100 requêtes concurrentes : préférer une file séquentielle à un appel parallèle par dépôt.
- **GraphQL n'a ni ETag ni 304** et possède un budget séparé (points/min). On peut répartir les appels REST/GraphQL pour augmenter le quota global, mais on perd le cache conditionnel côté GraphQL.
- **Webhooks non garantis.** Une livraison peut être manquée, retardée ou dupliquée : ils ne dispensent jamais d'un polling de réconciliation, qui est le différenciateur de fiabilité du produit.

## Checklist d'application

- [ ] Polling via **cron** (modèle Dependabot/Renovate), webhooks repoussés comme optimisation.
- [ ] Requêtes **authentifiées** (PAT ou GitHub App) — jamais de polling anonyme à 60 req/h.
- [ ] Stocker l'**ETag/Last-Modified** et le renvoyer (`If-None-Match`/`If-Modified-Since`) ; traiter `304` comme « rien de neuf » sans coût de quota.
- [ ] Choisir l'endpoint : `/releases/latest` (stable) vs `List releases` + filtre `prerelease` (canal RC) ; ou API Tags si la techno ne fait que des tags.
- [ ] `per_page=100` et arrêt dès qu'on retombe sur une version déjà vue.
- [ ] **Curseur persistant** par techno : `{ lastTag, etag, releaseId }`.
- [ ] Comparaison via une **bibliothèque SemVer 2.0.0** (build metadata ignoré, pré-release < release) ; parsing tolérant + log des tags non conformes.
- [ ] **Idempotence** : dédup sur `release_id`/tag pour ne pas re-générer une synthèse.
- [ ] Backpressure : respecter `x-ratelimit-reset` et `retry-after`, **backoff exponentiel** sur 5xx et rate limit secondaire.
- [ ] **File séquentielle** (1 worker), pas de fan-out parallèle massif.
- [ ] Si webhooks ajoutés : garder le **poll de réconciliation** comme filet de sécurité.

## 10 questions / réponses

**Q1. Pourquoi commencer par du polling et pas directement par des webhooks ?**
Parce que le polling cron est l'approche pragmatique et prouvablement fiable : c'est le modèle de Dependabot/Renovate, qui interrogent les registres sur planning sans temps réel par défaut ([Renovate Docs](https://docs.renovatebot.com/bot-comparison/)). Les webhooks sont une optimisation : ils peuvent être manqués, donc ils ne suffisent pas seuls pour une promesse de fiabilité.

**Q2. Qu'est-ce qu'une requête conditionnelle et pourquoi est-ce gratuit ?**
On renvoie l'ETag (ou la date) de la réponse précédente via `If-None-Match` (ou `If-Modified-Since`). Si rien n'a changé, GitHub répond `304 Not Modified` sans corps, et ce `304` **ne décompte pas** du rate limit primaire quand on est authentifié ([GitHub best practices](https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api)). On ne paie donc que les vrais changements.

**Q3. Quels sont les quotas exacts à connaître pour dimensionner le cron ?**
60 req/h en anonyme, 5 000 req/h avec un token, et une GitHub App va de 5 000 à 12 500 req/h par installation. Côté limites secondaires : 100 requêtes concurrentes max, 900 points/min en REST, 90 s de CPU par 60 s réelles ([Rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)).

**Q4. `/releases/latest` ou `List releases` : lequel poller ?**
`GET /releases/latest` renvoie la dernière release **stable** (non-prerelease, non-draft) — idéal pour le canal stable. Pour capter les RC/beta (ex. React 19 avant la GA), utiliser `List releases` puis filtrer `prerelease === true` côté client ([Releases API](https://docs.github.com/en/rest/releases/releases)).

**Q5. Pourquoi ne pas comparer les versions comme des chaînes ?**
Parce que l'ordre lexicographique se trompe : `'9.0.0'` passe après `'10.0.0'` et `'alpha10'` avant `'alpha2'`. SemVer 2.0.0 compare les identifiants numériques numériquement, ignore le build metadata, et place la pré-release avant la release (`1.0.0-rc.1 < 1.0.0`) ([SemVer 2.0.0](https://semver.org/)).

**Q6. À quoi sert le curseur exactement ?**
C'est l'état persistant par techno qui matérialise « ce que j'ai déjà vu » : typiquement `{ lastTag, etag, releaseId }`. On compare chaque réponse au curseur pour ne déclencher la synthèse que sur un **vrai delta**, puis on l'avance — c'est aussi ce qui garantit qu'on ne re-traite pas une version connue.

**Q7. Comment rendre la détection idempotente ?**
En dédupliquant sur l'identifiant de release (ou le tag) : un re-poll ou une livraison webhook dupliquée ne doit pas relancer une synthèse déjà produite. Sans cela, on paie deux fois le coût LLM et on génère du bruit utilisateur.

**Q8. Que faire quand on touche le rate limit ?**
Si `x-ratelimit-remaining` vaut 0, attendre `x-ratelimit-reset` (epoch UTC) ; si l'en-tête `retry-after` est présent (limite secondaire), respecter exactement ce délai et appliquer un backoff exponentiel. Ne jamais retenter en boucle serrée, ce qui est vu comme un abus ([GitHub best practices](https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api)).

**Q9. ETag ou Last-Modified — y a-t-il un piège ?**
Oui : un praticien Renovate/Dependabot a observé que l'ETag GitHub peut changer alors que la ressource est identique (faux `200`), alors que `Last-Modified` reste stable. Dans ce cas `If-Modified-Since` économise mieux le quota ([Jamie Magee](https://jamiemagee.co.uk/blog/making-the-most-of-github-rate-limits/)). Attention aussi : l'ETag est mis en cache par token, donc un renouvellement de token GitHub App provoque un cold start.

**Q10. Comment intégrer proprement les webhooks plus tard ?**
Avec le pattern « webhook + poll de réconciliation » : le webhook `release.published` donne le temps réel et ne coûte pas de quota ([Webhook events](https://docs.github.com/en/webhooks/webhook-events-and-payloads)), mais un cron de réconciliation rejoue `List releases` depuis le curseur pour rattraper toute livraison manquée. Comme les webhooks ne garantissent pas la livraison, le polling reste le filet de sécurité.

## Sources

1. [Best practices for using the REST API — GitHub Docs](https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api) — *officielle* — recommande les requêtes conditionnelles (`If-None-Match`/`If-Modified-Since`), les requêtes sérielles, la gestion de `retry-after` / `x-ratelimit-remaining` / `x-ratelimit-reset`, et confirme que `304` ne décompte pas du rate limit primaire.
2. [Rate limits for the REST API — GitHub Docs](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api) — *officielle* — chiffres exacts : 60 req/h anonyme, 5 000 req/h authentifié, GitHub App 5 000 → 12 500 req/h, limites secondaires (100 concurrentes, 900 points/min REST, 90 s CPU / 60 s réel).
3. [Semantic Versioning 2.0.0 — semver.org](https://semver.org/) — *officielle* — spécification SemVer : `MAJOR.MINOR.PATCH`, pré-releases, build metadata ignoré pour la précédence, règles de comparaison (numérique vs ASCII, pré-release < release).
4. [Releases REST API — GitHub Docs](https://docs.github.com/en/rest/releases/releases) — *officielle* — endpoints `List releases` et `Get latest release` ; `/releases/latest` renvoie la release non-prerelease/non-draft triée par `created_at` ; les releases diffèrent des tags Git.
5. [Webhook events and payloads (release event) — GitHub Docs](https://docs.github.com/en/webhooks/webhook-events-and-payloads) — *officielle* — référence des événements webhook dont l'événement `release` (actions `published`/`created`/`edited`), base de l'optimisation temps réel ultérieure.
6. [Bot comparison — Renovate Docs / Dependabot](https://docs.renovatebot.com/bot-comparison/) — *officielle* — modèle de scheduling : un job planifié exécute Renovate régulièrement (polling des registres), sans détection temps réel par webhook par défaut.
7. [Making the most of GitHub rate limits — Jamie Magee](https://jamiemagee.co.uk/blog/making-the-most-of-github-rate-limits/) — *praticien* — retour terrain Dependabot/Renovate : `304` gratuit, budgets REST/GraphQL séparés, `per_page=100`, et caveat ETag instable → préférer `If-Modified-Since`.
