# Collecte API-first & extraction de contenu

> **En une phrase.** Toujours demander la version et ses notes à l'API officielle du registre (~80 % des cas), ne scraper qu'en dernier recours via Firecrawl, et convertir/nettoyer en Markdown sourcé avant le LLM — car « garbage in, garbage out » : la qualité de l'extraction borne la qualité de la synthèse.

## Pourquoi c'est nécessaire pour TechWatch IA

TechWatch IA vend de la **confiance** : chaque affirmation de la synthèse de migration doit citer une source vérifiable. Or une synthèse n'est jamais meilleure que ses entrées. Comme environ 80 % des versions sont exposées par une API officielle (GitHub Releases, npm, PyPI, Maven Central), interroger ces API en premier donne des données structurées, datées et stables, sans la fragilité du scraping. Pour les 20 % restants (guides de migration en page HTML, blogs d'annonce), un fallback scraping respectueux puis une extraction propre conditionnent directement la fiabilité du livrable.

## Le principe

**API-first par défaut.** Pour détecter une nouvelle version, on interroge l'API canonique du registre avant tout scraping : `GET /repos/{owner}/{repo}/releases/latest` et `/tags/{tag}` sur GitHub, `GET /{package}` (champ `dist-tags.latest`) sur le registre npm, `pypi.org/pypi/<pkg>/json` sur PyPI, ou `maven-metadata.xml` sur Maven Central. Ces réponses sont des données structurées : on y lit des identifiants stables (`tag_name`, `published_at`, `prerelease`, `draft`) qui permettent de décider de façon déterministe si une version est « GA » et doit déclencher le workflow — en ignorant drafts et pre-releases sauf demande explicite.

**Récupérer la version ET son contenu au même endroit.** Quand l'API le permet, on évite un aller-retour vers le HTML : le champ `body` d'une GitHub Release contient déjà les release notes en Markdown. Pas besoin de scraper la page correspondante.

**Authentifier pour ne pas se faire limiter.** Un appel GitHub non authentifié plafonne vers ~60 requêtes/heure ; avec un token, on monte à 5 000 req/h. Il faut en plus gérer le *secondary rate limiting* avec un backoff exponentiel et respecter les en-têtes `Retry-After` / `X-RateLimit-Reset` ([GitHub REST API — Releases](https://docs.github.com/en/rest/releases/releases)).

**Le scraping est un plan B.** On ne scrape (via Firecrawl) que lorsqu'aucune API n'expose le contenu. Le fallback doit respecter la RFC 9309 : lire et mettre en cache `robots.txt` (24 h max), appliquer la précédence allow/disallow « most-octets », limiter le débit, et identifier clairement son User-Agent.

**Nettoyer et convertir en Markdown.** Pour le contenu d'article, un extracteur heuristique déterministe (Readability.js, Trafilatura) isole le texte principal ; la conversion HTML→Markdown réduit le bruit et le coût en tokens d'environ 80 % ([Firecrawl](https://www.firecrawl.dev/blog/scrape-a-website-to-markdown)). On conserve la provenance (URL, date de fetch, état du cache) pour que chaque fragment reste citable.

## Exemples concrets

**1. Détection React 18→19, 100 % API-first.**

```python
import httpx

# Étape A — détecter la dernière version GA via npm (pas de scraping)
pkg = httpx.get("https://registry.npmjs.org/react").json()
latest = pkg["dist-tags"]["latest"]          # ex. "19.0.0"

# Étape B — récupérer les release notes (markdown) via GitHub Releases
rel = httpx.get(
    f"https://api.github.com/repos/facebook/react/releases/tags/v{latest}",
    headers={"Authorization": f"Bearer {GITHUB_TOKEN}"},  # 60 -> 5000 req/h
).json()

source = {
    "version": rel["tag_name"],
    "published_at": rel["published_at"],
    "is_ga": not rel["prerelease"] and not rel["draft"],
    "body_markdown": rel["body"],            # notes déjà en markdown
    "fetched_at": "2026-06-23T10:00:00Z",
    "source_url": rel["html_url"],
}
```

Zéro scraping, provenance datée et citable.

**2. Fallback Firecrawl sur le guide de migration.** Le guide « Upgrading to React 19 » (react.dev) est une page rendue côté navigateur. On le scrape avec `only_main_content: true` (supprime nav, sidebars, bannières cookies, footers) et `wait_for` pour laisser le JS s'hydrater, on obtient du Markdown LLM-ready, puis on **extrait en JSON schema-locked** (schéma Pydantic) les champs stables `{breaking_change, avant, après, codemod}` — reproductibles d'un run à l'autre ([Firecrawl Scrape](https://www.firecrawl.dev/blog/mastering-firecrawl-scrape-endpoint)).

**3. Détection d'extraction douteuse.** Sur la même page de blog d'annonce, faire tourner Readability.js *et* Trafilatura, comparer la longueur et la densité de liens du Markdown produit, et flagguer tout désaccord > 20 % comme « extraction douteuse » à re-vérifier — une garde concrète contre le *garbage in*.

## Subtilités & pièges à éviter

- **`latest` n'est pas universel.** Sur npm, `dist-tags.latest` peut pointer un patch d'une ancienne majeure, pas la plus haute version semver ; sur GitHub, `/releases/latest` se base sur la date (et exclut drafts/prereleases), pas sur le semver. Ne pas confondre « latest » et « plus récent en semver ».
- **`body` vs `CHANGELOG.md`.** Le `body` d'une release est rédigé par un humain (qualité variable) ; certains projets mettent les vraies notes dans un `CHANGELOG.md` du repo. Il faut parfois croiser les deux.
- **Ne pas scraper ce qu'une API expose** : DOM fragile, plus lent, soumis inutilement à `robots.txt` et aux anti-bots.
- **Ne pas envoyer de HTML brut au LLM** : coût en tokens ~5x, signal dilué par la navigation/pubs, hallucination accrue. Nettoyer puis convertir en Markdown d'abord.
- **Vélocité ≠ volume.** La cause n°1 de ban IP est la vitesse, pas le volume : sérialiser par domaine, espacer les requêtes (crawl-delay), identifier son product token.
- **Readability échoue sur les SPA** : on récupère un shell HTML vide et on « résume du vide ». Détecter le contenu manquant et basculer en rendu navigateur réel (`wait_for`). La garde `isProbablyReaderable()` évite aussi de tourner sur pages d'accueil/listes/login ([Readability.js](https://webcrawlerapi.com/blog/mozilla-readability-algorithm-readabilityjs)).
- **Ne pas remplacer l'heuristique par « un LLM qui parse le HTML »** : l'étude SIGIR 2023 montre que sur les pages les plus complexes, les heuristiques (Readability médiane 0.970) sont *plus* robustes — en plus d'être déterministes et moins chères ([Bevendorff et al.](https://chuniversiteit.nl/papers/comparison-of-web-content-extraction-algorithms)).
- **`robots.txt` n'est pas un mur d'accès** : il régule les crawlers, pas l'accès humain ni les API publiques documentées (qui restent soumises aux Terms of Service et au rate limiting).
- **Le Markdown pivot perd parfois de l'information** : tableaux complexes, code multi-langage, callouts et notes de bas de page se dégradent. Vérifier que les *breaking changes* en tableau survivent à la conversion.
- **Le prompt libre produit des noms de champs instables** entre runs : pour un workflow déterministe et évaluable, le schéma fixe est quasi obligatoire.
- **Caching à double tranchant** : un `max_age` trop long peut masquer une version fraîchement publiée. TTL court sur la détection, long sur l'extraction de contenu historique.
- **Extraire sans provenance** (URL + date + ancre) casse le différenciateur « fiable et vérifiable » : impossible de faire citer ses sources à la synthèse.
- **Traiter une extraction bruitée comme valide** : si footers/related-articles ont fui dans le texte, la synthèse sera plausible mais fausse, sans signal d'alerte. La baseline d'hallucination reste de 5–10 % même en simple résumé ([applied-llms.org](https://applied-llms.org/)).

## Checklist d'application

- [ ] Interroger l'API officielle du registre **avant** tout scraping (GitHub / npm / PyPI / Maven).
- [ ] Lire le contenu source au même endroit que la version (`body` de la release) quand c'est possible.
- [ ] Décider GA via `tag_name` + `published_at` + `prerelease`/`draft` ; ignorer drafts et pre-releases par défaut.
- [ ] Authentifier les appels GitHub (token) et gérer backoff + `Retry-After` / `X-RateLimit-Reset`.
- [ ] N'activer le scraping qu'en fallback documenté, quand aucune API n'expose le contenu.
- [ ] Côté fallback : lire/cacher `robots.txt` (≤ 24 h), respecter allow/disallow (most-octets), 4xx = autorisé / 5xx = tout interdit, parsing ≥ 500 KiB.
- [ ] Limiter le débit : sérialiser par domaine, crawl-delay, User-Agent/product token explicite.
- [ ] Extraire le contenu d'article avec un extracteur heuristique (Readability.js / Trafilatura).
- [ ] Convertir systématiquement en Markdown (format pivot) avec `only_main_content` / reader-mode.
- [ ] Utiliser une extraction JSON schema-locked (Pydantic) pour les champs stables.
- [ ] Détecter les SPA non hydratées et basculer en rendu navigateur (`wait_for`).
- [ ] Stocker la provenance bout-en-bout : `{url, fetched_at, cache_state}` par fragment.
- [ ] Flagguer ou rejeter une source mal extraite plutôt que de la résumer.
- [ ] Mettre en cache (ETag/`If-None-Match` côté API, `max_age` côté Firecrawl) avec un TTL adapté à l'étape.

## 10 questions / réponses

**Q1. Pourquoi privilégier l'API plutôt que de scraper la page de release ?**
Parce que l'API renvoie des données structurées, stables et datées (`tag_name`, `published_at`, `body`), alors que le HTML peut changer à tout moment et tombe sous `robots.txt` et les anti-bots. Scraper ce qu'une API expose est plus fragile, plus lent et inutilement risqué ([GitHub REST API](https://docs.github.com/en/rest/releases/releases)).

**Q2. Comment détecter qu'une nouvelle version est sortie sans scraper ?**
On interroge l'API du registre : `dist-tags.latest` sur npm, `pypi.org/pypi/<pkg>/json` sur PyPI, `/releases/latest` ou `/tags/{tag}` sur GitHub. On compare au dernier état connu et on déclenche le workflow si la version a changé ([npm Registry API](https://github.com/npm/registry/blob/main/docs/REGISTRY-API.md)).

**Q3. Faut-il scraper la page de release pour avoir les notes ?**
Non, dans la plupart des cas : le champ `body` d'une GitHub Release contient déjà les release notes en Markdown. On récupère version et contenu au même endroit, sans aller-retour HTML ([GitHub REST API](https://docs.github.com/en/rest/releases/releases)).

**Q4. Pourquoi authentifier les appels GitHub ?**
Sans token, on plafonne à ~60 requêtes/heure ; avec un token, on passe à 5 000 req/h. Cela évite de se faire limiter en pleine exécution du workflow. Il faut aussi gérer le *secondary rate limiting* avec un backoff et respecter `Retry-After` ([GitHub REST API](https://docs.github.com/en/rest/releases/releases)).

**Q5. Attention au piège de `latest` : que faut-il savoir ?**
`latest` n'est pas toujours la plus haute version semver. Sur npm, `dist-tags.latest` peut désigner un patch d'une ancienne majeure ; sur GitHub, `/releases/latest` se base sur la date, pas le semver. Il faut valider le semver soi-même quand l'ordre compte ([npm Registry API](https://github.com/npm/registry/blob/main/docs/REGISTRY-API.md)).

**Q6. Quelles règles respecter quand on tombe en fallback scraping ?**
La RFC 9309 : mettre `robots.txt` en cache (≤ 24 h), appliquer la précédence allow/disallow par le match « le plus d'octets », traiter un 4xx comme « autorisé » et un 5xx comme « tout interdit », et parser au moins 500 KiB. Le matching du product token est insensible à la casse ([RFC 9309](https://datatracker.ietf.org/doc/html/rfc9309)).

**Q7. Pourquoi convertir le HTML en Markdown avant le LLM ?**
Parce que le Markdown réduit le coût en tokens d'environ 80 % (analyse Cloudflare : 16 180 tokens HTML vs 3 150 en Markdown) et que les modèles ont une meilleure « Markdown Awareness ». Moins de bruit, moins de coût, moins d'hallucination ([Firecrawl](https://www.firecrawl.dev/blog/scrape-a-website-to-markdown)).

**Q8. Quel extracteur choisir pour le texte d'un article ?**
Un extracteur heuristique déterministe (Readability.js, Trafilatura) plutôt qu'un gros modèle neuronal : l'étude SIGIR 2023 mesure une médiane de 0.970 pour Readability et montre que les heuristiques sont plus robustes sur les pages complexes ([Bevendorff et al.](https://chuniversiteit.nl/papers/comparison-of-web-content-extraction-algorithms)).

**Q9. Que faire quand Readability renvoie une page presque vide ?**
C'est typiquement une SPA dont le contenu est injecté par JavaScript : Readability ne voit qu'un shell vide. Il faut détecter ce contenu manquant et basculer vers un rendu navigateur réel (Firecrawl `wait_for`) qui hydrate le DOM avant extraction ([Readability.js](https://webcrawlerapi.com/blog/mozilla-readability-algorithm-readabilityjs)).

**Q10. Pourquoi préférer une extraction schema-locked et conserver la provenance ?**
Le prompt libre produit des noms de champs instables d'un run à l'autre, ce qui casse un workflow déterministe et évaluable ; un schéma Pydantic fixe les champs (version, date, breaking changes). Conserver `{url, fetched_at, cache_state}` permet à chaque affirmation de citer sa source — c'est ce qui rend la synthèse vérifiable, puisque la baseline d'hallucination reste de 5–10 % ([applied-llms.org](https://applied-llms.org/) ; [Firecrawl Scrape](https://www.firecrawl.dev/blog/mastering-firecrawl-scrape-endpoint)).

## Sources

1. [GitHub REST API — Releases](https://docs.github.com/en/rest/releases/releases) — *doc-officielle* — endpoints list/latest/by-tag, champs structurés (`tag_name`, `body`, `published_at`, `prerelease`, `draft`), rate limiting et secondary rate limiting.
2. [npm Registry API — REGISTRY-API.md](https://github.com/npm/registry/blob/main/docs/REGISTRY-API.md) — *doc-officielle* — packument (`GET /{package}`), métadonnée par version, `dist-tags` ; modèle de référence pour la dernière version d'un paquet.
3. [RFC 9309 — Robots Exclusion Protocol](https://datatracker.ietf.org/doc/html/rfc9309) — *rfc* — règles formelles du crawler : cache 24 h, précédence most-octets, 4xx autorisé / 5xx interdit, parsing ≥ 500 KiB, matching case-insensitive du product token.
4. [An Empirical Comparison of Web Content Extraction Algorithms (Bevendorff et al., SIGIR 2023)](https://chuniversiteit.nl/papers/comparison-of-web-content-extraction-algorithms) — *académique* — les extracteurs heuristiques (Readability médiane 0.970, Trafilatura, Goose3) battent les modèles neuronaux en robustesse sur les pages complexes.
5. [Mozilla Readability Algorithm (Readability.js) expliqué](https://webcrawlerapi.com/blog/mozilla-readability-algorithm-readabilityjs) — *praticien* — heuristiques de Readability.js, garde `isProbablyReaderable()`, et cas d'échec (JS non rendu, listes, DOM cassé).
6. [Mastering Firecrawl's Scrape Endpoint](https://www.firecrawl.dev/blog/mastering-firecrawl-scrape-endpoint) — *doc-officielle* — fallback à rendu navigateur, `only_main_content`, sortie Markdown LLM-ready, extraction JSON schema-locked (Pydantic), `wait_for`, caching `max_age`.
7. [What We Learned from a Year of Building with LLMs](https://applied-llms.org/) — *praticien* — principe « garbage in, garbage out » et nécessité d'identifier les sources en sortie pour permettre la vérification.
8. [How To Scrape A Website To Markdown For LLMs And AI Agents](https://www.firecrawl.dev/blog/scrape-a-website-to-markdown) — *praticien* — gain de la conversion HTML→Markdown : 16 180 tokens HTML vs 3 150 en Markdown (−80 %, analyse Cloudflare), benchmark MDEval / « Markdown Awareness ».
