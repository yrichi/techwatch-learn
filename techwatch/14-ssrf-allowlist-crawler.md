# Sécurité du crawler : SSRF & allowlist

> **En une phrase.** Pour que TechWatch IA reste *fiable et vérifiable*, son crawler ne doit jamais aller chercher une URL arbitraire : on n'autorise qu'une liste finie de domaines officiels (allowlist), on épingle l'IP résolue à la connexion (DNS pinning) et on coupe les redirections — sinon une attaque SSRF peut transformer le crawler en relais vers le réseau interne ou l'endpoint de métadonnées cloud `169.254.169.254`.

## Pourquoi c'est nécessaire pour TechWatch IA

Le différenciateur du projet, c'est la **confiance** : chaque affirmation d'une synthèse de migration (React 18→19, etc.) cite sa source. Or le crawler reçoit ses cibles d'un pipeline piloté par LLM, et un LLM peut produire une URL inattendue, empoisonnée ou injectée. Si le crawler suit aveuglément cette URL, il devient un vecteur de **SSRF (Server-Side Request Forgery)** : le serveur émet une requête vers une destination contrôlée par l'attaquant, potentiellement interne. Comme TechWatch IA ne consomme que des **sources officielles connues d'avance**, il est le cas idéal pour une **allowlist statique versionnée** — un contrôle central qui borne le risque tout en garantissant que les sources citées sont bien celles qu'on prétend.

## Le principe

Le SSRF est classé **A10 du OWASP Top 10 (2021)** : l'application est manipulée pour émettre une requête vers une ressource non prévue, en contournant les contrôles réseau. La parade la plus robuste pour un consommateur de sources connues est l'**allowlist**, pas la **blocklist**. OWASP est explicite : *« Deny-lists are bypass-prone. Prefer allow-lists. »* Une blocklist (« interdire telle IP, tel domaine ») se contourne par encodage, IPv6-mapped ou rebinding DNS ; elle n'est réservée qu'aux cas où la destination est dynamique et inconnue (webhooks utilisateur), ce qui n'est pas notre cas.

Concrètement, le crawler ne doit **jamais accepter d'URL brute** issue du LLM ou de l'utilisateur. Il reçoit une **clé de source** (ex. `"react"`) et **reconstruit** l'URL côté serveur depuis un catalogue de sources autorisées. La validation s'enchaîne ensuite : (1) **schéma** — `https` uniquement, ports 80/443, on rejette `file://`, `gopher://`, `ftp://`, `data://` ; (2) **host** — comparaison d'**égalité stricte** contre un ensemble fini de domaines canoniques, jamais un `endsWith(".github.com")` naïf qui laisserait passer `react.dev.evil.com` ; (3) **composants piégés** — rejet du `userinfo` (`user@host`), des fragments, des doubles encodages, normalisation avant comparaison.

Mais valider l'URL ne suffit **jamais seul**. Le piège classique est le **TOCTOU** (time-of-check / time-of-use) : on valide l'IP du host, puis on redonne le *hostname* à la lib HTTP, qui le **re-résout** — et un serveur DNS malveillant (TTL=0) renvoie alors `169.254.169.254`. La correction se situe à la **couche connexion** : **DNS pinning**. On résout une fois, on valide **toutes** les IP renvoyées contre les plages interdites (loopback `127.0.0.0/8`, `::1` ; RFC1918 `10/8`, `172.16/12`, `192.168/16` ; link-local `169.254.0.0/16` et `fe80::/10` ; métadonnées cloud `169.254.169.254`, `metadata.google.internal`), puis on **se connecte directement à l'IP validée** (en gardant le hostname dans l'en-tête `Host` et le SNI). On **désactive les redirections** : un `3xx` vers `169.254.169.254` casserait sinon toute la validation amont. Enfin, tout passe par **un seul module `secureFetch`** obligatoire — un point de contrôle unique et auditable.

## Exemples concrets

**1. Catalogue d'allowlist versionné (le crawler ne voit jamais d'URL libre)**

```ts
// sources.config.ts — versionné en git, revu en PR
export const SOURCES = {
  react: {
    hosts: ["react.dev", "raw.githubusercontent.com"],
    // chemins précis : pas tout github.com, pas les gists arbitraires
    pathPrefixes: ["/blog/", "/facebook/react/"],
  },
} as const;

// Le pipeline LLM fournit la CLÉ "react", jamais une URL.
function buildReleaseUrl(key: "react"): string {
  // URL reconstruite côté serveur → impossible d'injecter un host arbitraire
  return "https://react.dev/blog/2024/04/25/react-19";
}
```

**2. `secureFetch` avec DNS pinning (Node / undici)**

```ts
import { lookup } from "node:dns/promises";
import { request } from "undici";

async function secureFetch(url: string) {
  const u = new URL(url);
  if (u.protocol !== "https:") throw new Error("schéma interdit");
  if (u.username || u.password) throw new Error("userinfo interdit");
  if (!ALLOWED_HOSTS.has(u.hostname)) throw new Error("host hors allowlist"); // égalité stricte

  // Résoudre UNE fois, valider TOUTES les IP, puis épingler.
  const records = await lookup(u.hostname, { all: true });
  for (const { address } of records) {
    if (isBlockedIp(address)) throw new Error(`IP interne refusée: ${address}`); // 127/8, RFC1918, 169.254/16, ::1, fc00::/7, IPv4-mapped...
  }
  const pinnedIp = records[0].address;

  return request(`https://${pinnedIp}${u.pathname}`, {
    headers: { host: u.hostname }, // Host conservé → SNI/cert OK
    maxRedirections: 0,            // redirections coupées
    bodyTimeout: 5_000, headersTimeout: 5_000,
    // + borner taille de réponse et content-type attendu (text/html, application/json)
  });
}
// Règle ESLint : interdire l'import direct de fetch/axios/undici ailleurs.
```

**3. Test d'éval de sécurité dédié** — un faux serveur de rebinding renvoie d'abord une IP publique (validation OK) puis `169.254.169.254`. Le test prouve qu'**aucune note de migration React n'est jamais générée** à partir d'une réponse provenant de l'IMDS : le fetch doit échouer **au pinning**, pas après avoir lu le corps.

## Subtilités & pièges à éviter

- **Valider l'URL/IP ne suffit jamais seul.** Sans DNS pinning ni egress restreint, le rebinding DNS contourne tout : *« any defense that resolves a hostname, checks the result, and then hands the hostname back to an HTTP library to resolve again is vulnerable to rebinding »* (Behrad Taher). Le fix est à la **connexion**, pas au parsing.
- **`endsWith(".github.com")` est insuffisant** : `github.com.attacker.com` passe. Préférer l'**égalité stricte** sur un ensemble fini de hosts canoniques.
- **Allowlist trop large = retour au risque.** Autoriser `github.com` entier ouvre gists et contenus arbitraires. Cibler des sous-domaines/chemins précis (`raw.githubusercontent.com/facebook/react`, `react.dev/blog/`).
- **TOCTOU (advisory Craft CMS)** : valider puis laisser la lib re-résoudre le hostname est exactement le pattern exploité par le rebinding.
- **Suivre les redirections automatiquement** sans re-validation : un `3xx` vers un host interne ou `169.254.169.254` casse toute la validation. Couper, ou re-valider host + IP à **chaque** saut. Attention aux redirections inter-protocoles (`https→http`).
- **Sécurité non centralisée (advisory Flowise)** : un module qui importe directement `node-fetch`/`axios`/`undici` contourne le wrapper. L'enforcement doit être **unique et obligatoire** (lint + revue).
- **Métadonnées cloud incomplètes** : ne bloquer que `169.254.169.254` mais oublier `metadata.google.internal`, l'IPv6 `fd00:ec2::254`, le CGNAT `100.64.0.0/10` et l'ULA `fc00::/7`.
- **Une seule IP validée** : le DNS peut renvoyer plusieurs A/AAAA — valider **tous** les enregistrements. Et ne pas valider l'IP puis se connecter par hostname.
- **Formats d'IP alternatifs** : décimal `2130706433`, octal, hex `0x7f.1`, IPv4-mapped `::ffff:169.254.169.254` — **normaliser avant** toute comparaison de plages.
- **IMDSv2 réduit mais ne supprime pas le risque** : token + PUT requis, mais un proxy ouvert peut encore atteindre l'IMDS — d'où `HttpPutResponseHopLimit=1` (2 en conteneur) et le blocage réseau de `169.254.169.254`.
- **Egress IPv6 oublié** : beaucoup de configs ne filtrent que l'IPv4 et laissent `fd00:ec2::254` joignable.
- **Compter sur la seule validation applicative** : sans contrôle réseau d'egress, aucune défense en profondeur si l'app a un bug.

## Checklist d'application

- [ ] Le crawler reçoit une **clé de source**, jamais une URL brute ; l'URL est **reconstruite** depuis un catalogue versionné.
- [ ] Schéma restreint à **`https`**, ports **443/80** ; `file/gopher/ftp/data` rejetés.
- [ ] Host validé par **égalité stricte** contre un ensemble fini de domaines canoniques (pas d'`endsWith`/regex permissive).
- [ ] `userinfo`, fragments, doubles encodages rejetés ; **normalisation** avant comparaison.
- [ ] **Toutes** les IP résolues validées contre : loopback, RFC1918, link-local v4/v6, CGNAT, multicast, ULA `fc00::/7`, IPv4-mapped.
- [ ] Métadonnées cloud bloquées : `169.254.169.254`, `metadata.google.internal`, `fd00:ec2::254`.
- [ ] **DNS pinning** : résolution unique, connexion à l'IP validée, `Host`/SNI conservés.
- [ ] **Redirections désactivées** (ou re-validation host+IP à chaque saut).
- [ ] **Un seul `secureFetch`** ; import direct de `fetch`/`axios`/`undici` interdit (règle ESLint + revue).
- [ ] Timeouts, **taille de réponse max**, content-type borné (`text/html`, `application/json`).
- [ ] **Log** de chaque fetch : URL demandée, IP résolue, verdict allowlist, code HTTP.
- [ ] Egress **deny-by-default** sur les workers (Security Group / NetworkPolicy / proxy), IPv4 **et** IPv6.
- [ ] Sur AWS : **IMDSv2** (`HttpTokens=required`), hop limit 1 (2 en conteneur).
- [ ] **Résolveurs DNS publics** côté crawler (ils ignorent les noms d'infra interne).

## 10 questions / réponses

**Q1. C'est quoi le SSRF, en deux phrases ?**
Le Server-Side Request Forgery, c'est quand on manipule une application pour qu'elle émette une requête réseau vers une destination non prévue — souvent interne. C'est la catégorie A10 du Top 10 OWASP 2021, qui recommande un egress *deny-by-default* en plus des contrôles applicatifs (OWASP Top 10 A10).

**Q2. Pourquoi une allowlist plutôt qu'une blocklist ?**
Parce qu'une blocklist est contournable par encodage, IPv6 ou rebinding DNS : OWASP écrit explicitement *« Deny-lists are bypass-prone. Prefer allow-lists. »* TechWatch IA ne consomme que des sources officielles connues d'avance, c'est donc le cas idéal pour une allowlist statique versionnée (OWASP SSRF Cheat Sheet).

**Q3. Pourquoi ne pas accepter directement l'URL produite par le LLM ?**
Parce que cette URL peut être empoisonnée ou injectée, et les parseurs d'URL divergent et sont contournables. On accepte une clé de source (`"react"`) et on reconstruit l'URL côté serveur depuis un catalogue, ce qui rend impossible l'injection d'un host arbitraire (OWASP SSRF Cheat Sheet).

**Q4. Pourquoi `host.endsWith(".github.com")` est-il dangereux ?**
Parce qu'il laisse passer `github.com.attacker.com` et des sous-domaines non maîtrisés. Il faut une **égalité stricte** sur un ensemble fini de hosts canoniques, pas une comparaison par sous-chaîne (Stytch).

**Q5. Quelles plages d'IP faut-il bloquer après résolution ?**
Loopback `127.0.0.0/8` et `::1`, RFC1918 (`10/8`, `172.16/12`, `192.168/16`), link-local `169.254.0.0/16` et `fe80::/10`, CGNAT `100.64.0.0/10`, multicast `224.0.0.0/4`, ULA `fc00::/7`, et les IPv4-mapped en IPv6. OWASP liste ces plages comme minimum à refuser (OWASP SSRF Cheat Sheet).

**Q6. C'est quoi le DNS rebinding et pourquoi la validation d'URL ne l'arrête pas ?**
On valide l'IP d'un hostname, puis la lib HTTP **re-résout** ce hostname et obtient cette fois `169.254.169.254` : c'est le TOCTOU. Comme le dit Behrad Taher, *« any defense that resolves a hostname, checks the result, and then hands the hostname back to an HTTP library to resolve again is vulnerable to rebinding »* (Behrad Taher).

**Q7. Comment corrige-t-on ce TOCTOU ?**
Par le **DNS pinning** : résoudre une fois, valider l'IP, puis se connecter **directement à l'IP** en gardant le hostname dans l'en-tête `Host`. La correction est à la couche connexion, pas au parsing ; un proxy d'egress (type Smokescreen) ou des règles réseau renforcent la défense (Behrad Taher, Stytch).

**Q8. Pourquoi désactiver les redirections ?**
Parce qu'une réponse `3xx` vers `169.254.169.254` ou un host interne contourne toute la validation initiale, même si l'URL de départ était saine. OWASP demande de *« disable the support for the following of the redirection »* dans le client HTTP ; sinon, re-valider host **et** IP à chaque saut (OWASP SSRF Cheat Sheet).

**Q9. Pourquoi un seul `secureFetch` obligatoire ?**
Parce que si un module importe directement `node-fetch`/`axios`/`undici`, il contourne le wrapper sécurisé — c'est exactement l'anti-pattern de l'advisory Flowise. Un point de contrôle unique est un invariant de sécurité auditable, à imposer par règle de lint et revue (advisory Flowise).

**Q10. La validation applicative suffit-elle, ou faut-il du réseau ?**
Elle ne suffit pas : un bug applicatif laisserait passer. Il faut une défense en profondeur — egress *deny-by-default* sur les workers (NetworkPolicy / Security Group / proxy), couvrant IPv4 **et** IPv6, plus IMDSv2 forcé et hop limit 1 sur AWS. L'advisory Craft CMS prouve qu'une protection métadonnées seule peut être contournée par rebinding (advisory Craft CMS, OWASP Top 10 A10).

## Sources

1. [OWASP — Server Side Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html) — *doc-officielle* — référence canonique : allowlist vs blocklist (*« Deny-lists are bypass-prone. Prefer allow-lists »*), plages IP/métadonnées à bloquer, désactivation des redirections, validation du schéma.
2. [OWASP Top 10 (2021) — A10 Server-Side Request Forgery (SSRF)](https://owasp.org/Top10/2021/A10_2021-Server-Side_Request_Forgery_(SSRF)/) — *doc-officielle* — catégorisation officielle du risque, contrôles réseau (egress deny-by-default) et applicatifs ; ancrage de menace et vocabulaire partagé.
3. [DNS Rebinding Attacks Against SSRF Protections — Behrad Taher](https://behradtaher.dev/DNS-Rebinding-Attacks-Against-SSRF-Protections/) — *praticien* — explique le TOCTOU et les fixes corrects : DNS pinning (connexion directe à l'IP validée), proxy d'egress, contrôles réseau.
4. [Securing Identity APIs Against SSRF — Stytch](https://stytch.com/blog/securing-identity-apis-against-ssrf/) — *praticien* — retour d'expérience : allowlist domaines+protocoles+ports, blocage RFC1918/link-local/ULA, résolveurs DNS publics, client HTTP custom validant à la connexion, defense-in-depth.
5. [Craft CMS — Cloud Metadata SSRF Protection Bypass via DNS Rebinding (GHSA)](https://github.com/craftcms/cms/security/advisories/GHSA-gp2f-7wcm-5fhx) — *advisory* — cas réel : une protection métadonnées contournée par rebinding ; preuve que la validation d'URL seule ne suffit pas.
6. [Flowise — SSRF Protection Bypass via Direct node-fetch/axios Usage (GHSA)](https://github.com/FlowiseAI/Flowise/security/advisories/GHSA-qqvm-66q4-vf5c) — *advisory* — anti-pattern : sécurité SSRF centralisée mais contournée par import direct de `node-fetch`/`axios` ; justifie le point de contrôle central unique.
