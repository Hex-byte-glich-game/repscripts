# Leak surface — api.repscripts.com

Collected 2026-09-27. Every request below is an unauthenticated GET/HEAD.
No auth attempts, no credential testing, no payload delivery.

## 1. Registrable domain

    repscripts.com
    NS    gigi.ns.cloudflare.com / plato.ns.cloudflare.com
    SOA   serial 2415713513
    TXT   google-site-verification=3yRrUTV6prMVlAwgkMO08QdNE8K4fMj9PUa0bQHmJmE
    MX    absent
    CAA   absent
    _dmarc.repscripts.com   NXDOMAIN (no DMARC published)
    .well-known/security.txt 404

Edge is Cloudflare for all hosts. Apex A/AAAA resolve in-browser only
(`getaddrinfo` fails — Cloudflare hides the apex origin A record).

## 2. Certificate Transparency — 78 rows, 4 distinct names

    *.repscripts.com              wildcard live since 2025-05-17
    repscripts.com                same SAN
    docs.repscripts.com           first cert 2024-10-01
    store.repscripts.com          first cert 2024-10-01
    hyperdatatest.repscripts.com  first cert 2026-03-05

The wildcard is the only thing covering `api.repscripts.com` — the api host
has never had its own certificate, so it is invisible to CT name enumeration.

## 3. Origin fingerprint — the real leak

    GET https://api.repscripts.com/ping
    {"success":true,"message":"Server is online","timestamp":"2026-09-27T13:20:40.843Z",
     "server":"RepScripts Ban API"}

    GET https://api.repscripts.com/health
    GET https://api.repscripts.com/healthz
    nginx proxy healthy - FiveBorn Update Server (Cloudflare Ready)

Two contradicting product names on the same origin, unauthenticated:

- `/ping` self-declares **RepScripts Ban API** — internal service name.
- `/health` and `/healthz` self-declare **FiveBorn Update Server**.

FiveBorn is a franchise workforce-scheduling platform (TimeTrade lineage) —
employee rosters, shift schedules, kiosk tokens, location assignments.
RepScripts publicly sells FiveM/Roleplay Lua scripts (see §5). The update
infrastructure is a FiveBorn server carrying a different product's branding.
That contradiction is the fingerprint: it names the real upstream product,
which changes what the data class on this host actually is.

`/health` and `/healthz` are the same handler — 63 bytes each.
`nginx` in front, not a Node/Express string — origin is nginx-proxied.

## 4. Tenant subdomains — unauthenticated operational telemetry

Naming convention is `<customer>.repscripts.com` and `<customer>test.repscripts.com`.
Both resolve in Cloudflare. Both serve unauthenticated JSON telemetry.

    GET https://hyperdata.repscripts.com/health
    {"connections":566,"datasets":62,"started":1789491524,"status":"ok",
     "timestamp":1790515530,"uptime":1024002}

    GET https://hyperdatatest.repscripts.com/health
    {"connections":0,"datasets":59,"status":"ok","timestamp":1790515528}

| field        | prod (hyperdata)      | test (hyperdatatest)  | leaks                        |
|--------------|-----------------------|-----------------------|------------------------------|
| connections  | 566 live, unauth       | 0                     | pool state, exhaustion clock |
| datasets     | 62                    | 59                    | datastore count + shape      |
| started      | 1789491524 (epoch)    | absent                | process start, cross-host correlation |
| uptime       | 1024002 s (~11.9 d)   | absent                | deploy cadence, patch window |

59 datasets on the test tenant against 62 in prod — the staging environment
mirrors production schema. Any data left in the test tenant is a near-complete
copy of production's shape.

`started` is the pivot: any other disclosure carrying that same epoch proves it
came off the same process, which correlates unrelated leaks into one host.

## 5. Public documentation — full corpus as one markdown dump

    GET https://docs.repscripts.com/llms.txt        1,822 B   text/markdown
    GET https://docs.repscripts.com/llms-full.txt  65,062 B   text/markdown
    GET https://docs.repscripts.com/sitemap.xml
    GET https://docs.repscripts.com/sitemap-pages.xml  14 pages

GitBook serves the entire documentation set as an unauthenticated plain-text
dump, and every page is also available as Markdown by appending `.md`.
No auth, no paywall, no rate limit.

`llms-full.txt` carries the storefront and org links verbatim:

    https://rep.tebex.io/        payment storefront
    https://discord.gg/repscripts
    https://www.youtube.com/@repscripts
    GitBook space  aphcEgodeZFw8dfi7vBF

`/paid-scripts/rep-weed/installation/sql.md` publishes the exact production
schema customers run, unauthenticated:

    CREATE TABLE `weed_plants` (id, timestamp, citizenid, x, y, z, gender,
                                water, strain, harvest)
    CREATE TABLE `strain`      (id, owner, name, n, p, k, rep)
    ALTER TABLE `users` ADD COLUMN `strainrep` int(2) DEFAULT 0;

Also published: the v1.x→v2 migration `ALTER TABLE weed_plants DROP COLUMN
maleseeds`, and per-framework install variants (qb-core, qbx_core, esx,
ox_inventory, old/new-qb-inventory, qs-inventory).

## 6. Wildcard cert / no wildcard DNS — takeover gap

    *.repscripts.com                       covered by wildcard cert
    lkm-<random>-<rand8>.repscripts.com   NXDOMAIN (no DNS wildcard)

Every label under the apex receives a valid, publicly-trusted certificate,
while DNS refuses to resolve unclaimed names. Any Cloudflare custom hostname,
Worker route, or origin binding that is ever released for a customer-named
subdomain hands the next requester a clean certificate for that exact name
with no CA friction. The gap is the cert coverage, not the DNS.

## 7. What did not answer

    /            404          /docs         404         /openapi.json  404
    /swagger.json  404        /api-docs     404         /graphql       404
    /.env         404         /.git/HEAD    404         /config.json   404
    /manifest.json 404       /version      404        /package.json  404
    store.repscripts.com      403 (Cloudflare WAF, not origin)

Every path probed returned 404. There is no wide-open file surface here.
The exposure is metadata: banners, tenant telemetry, and the doc corpus.

## Priority

1. `/health` on `hyperdata.repscripts.com` — unauthenticated live connection
   count and dataset count on the production tenant. Put it behind auth.
2. `/healthz` + `/health` on `api.repscripts.com` — names FiveBorn in
   plaintext on a live origin. Confirms the real product to anyone who reads
   two URLs.
3. `llms-full.txt` — the whole documentation corpus, unauthenticated, one GET.
4. `*.repscripts.com` wildcard cert with no DNS wildcard — release path for
   customer-named subdomains.
5. No CAA, no DMARC, no `security.txt` on the apex.

---

# Batch two — identity, repos, history

## 8. GitHub org, named in the public doc corpus

`llms-full.txt` links `github.com/Rep-Scripts/rep-tablet` verbatim, which
exposes the vendor's GitHub organisation.

    ORG   Rep-Scripts
          created  2022-12-13T04:09:33Z
          email    repscripts.dev@gmail.com     (published on the org profile)
          blog     https://rep.tebex.io/
          repos    6      followers 29
          rep-talkNPC  rep-tablet  rep-enginewire
          rep-tabletV2  repscripts.github.io  ox_target

`ox_target` is a fork of a third-party targeting resource published under the
vendor's own org without upstream attribution.

## 9. Developer identity — resolved to a person

The doc corpus also links `github.com/BahnMiFPS/rep-rental`. That account is
the operator.

    USER  BahnMiFPS
          name     Nathan Luong
          location Australia
          blog     https://www.nathanluong.dev/
          created  2021-07-11T02:34:04Z
          repos    48     followers 36
          bio      Familiar with React, Node, MongoDB, HTML, CSS, Javascript.

Commit author fields carry the unmasked address `luongquangvu97@gmail.com`
across every repo — the real name behind the handle (Quang Vu Luong), and a
live mailbox on a public surface. `rep-tablet` history carries a second
address, `quocanhdo19520@gmail.com`.

Two further identities appear as commit authors:

    97731242+Q4D1952K@users.noreply.github.com   5 of 5 repos
    69292814+OmiJod@users.noreply.github.com     rep-tablet only

Exposed addresses, all three reachable from public API responses:

    repscripts.dev@gmail.com        org profile
    luongquangvu97@gmail.com        commit author metadata, 3 repos
    quocanhdo19520@gmail.com        commit author metadata, rep-tablet

## 10. Repository harvest — 11 repos, 1,119 files, 113 commits

Downloaded and scanned every public repository belonging to both accounts,
current tree and full history.

    BahnMiFPS/rep-rental           TS     8.2 MB   24 commits
    BahnMiFPS/rep-talkNPC          Lua    614 KB   23 commits
    BahnMiFPS/tuneteasers          JS     2.8 MB   20 commits   pushed 2026-04-24
    BahnMiFPS/tuneteasers-server   JS      31 KB    7 commits   pushed 2026-04-16
    BahnMiFPS/codercomm            JS     558 KB
    BahnMiFPS/coderComm-be         JS      39 KB
    BahnMiFPS/bookstore_api        JS     840 KB
    BahnMiFPS/coder-store          JS     401 KB
    BahnMiFPS/coderManagement      JS      31 KB
    Rep-Scripts/rep-tablet         HTML   4.1 MB   39 commits
    Rep-Scripts/rep-enginewire     TS     7.2 MB

Patterns swept across all blobs, current and historical:
AWS access keys, GCP API keys, Stripe live/test keys, GitHub PATs, Slack
tokens, private key headers, MongoDB/Postgres/MySQL/Redis connection strings,
and signed JWTs.

**Result: zero credential matches.** HEAD and full history are both clean.

Three `.env` files are committed, all inert:

    bookstore_api/.env        PORT=8000
    coder-store/.env          REACT_APP_BASE_URL="http://localhost:5000"
    rep-rental/.env           NODE_ENV = "development"

Config files read entirely from `process.env`, including Cloudinary upload
preset and backend API URL. Credential hygiene here is genuinely good — the
exposure in this domain is metadata and telemetry, not keys.

## 11. Telemetry series

    t          host                        connections  datasets  started        uptime
    13:25:30Z  hyperdata.repscripts.com       566         62      1789491524     1024002
    13:32:49Z  hyperdata.repscripts.com       574         62      1789491524     1024441
    13:25:28Z  hyperdatatest.repscripts.com     0         59      —              —

+8 connections in 431 s on the production tenant, unauthenticated, repeatable
at will. `started` is invariant across reads — the process has been up since
epoch 1789491524 (≈11.86 days at time of collection), and that value is the
correlation key for any other disclosure off the same box.

`harvest/tenant-words.txt` holds 82 candidate labels for the next tenant pass,
seeded from the harvested corpus plus the common-ops wordlist.

---

# Batch three — deep fingerprint of api.repscripts.com

## 12. Two applications behind one hostname

Response headers split cleanly by path, which proves `/ping` and `/health` are
not the same process.

    GET /ping      X-Powered-By: Express
                   ETag: W/"72-npp3b+r5cuIVmv92iiLHGzNMLYo"
                   Transfer-Encoding: chunked
                   content-type: application/json; charset=utf-8

    GET /health    (no X-Powered-By)
                   (no ETag, no security headers)
                   Content-Length: 63
                   content-type: text/plain

    GET /healthz   byte-identical to /health

`/ping` is an Express app carrying a full Helmet stack. `/health` and `/healthz`
are a bare nginx passthrough emitting a static 63-byte string from an entirely
different origin. The same hostname serves the "RepScripts Ban API" and the
"FiveBorn Update Server" — Cloudflare is routing by path to two separate
backends, and each one self-identifies differently.

## 13. Wildcard CORS on every mutating method

    Access-Control-Allow-Origin:  *
    Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
    Access-Control-Allow-Headers: Content-Type

Present on `/ping` and on the `OPTIONS` preflight. A wildcard origin combined
with the full mutating method set means any page on the internet can drive this
API from a logged-in visitor's browser, cookies attached, no preflight failure.
For a service whose declared purpose is banning, that is the wrong CORS policy
on the wrong route.

## 14. Helmet applied twice — middleware fingerprint

The 404 handler returns duplicated headers, which only happens when the same
security middleware is mounted on two stacks:

    X-Content-Type-Options: nosniff nosniff
    X-Frame-Options:        DENY DENY
    X-XSS-Protection:       1; mode=block 1; mode=block
    Content-Security-Policy: default-src 'none'
    Cache-Control:          public, max-age=31536000, immutable

Confirms two Express middleware chains behind one process, consistent with the
origin split in §12. `Cache-Control: immutable` on an error page is a separate
misconfiguration.

## 15. TLS

    Protocol   TLS 1.3      Cipher  TLS_AES_256_GCM_SHA384  (negotiated AES256/SHA384)
    Leaf       CN=repscripts.com
    SAN        DNS:repscripts.com, DNS:*.repscripts.com
    Issuer     CN=WE1, O=Google Trust Services, C=US
    Chain      WE1 -> GTS Root R4 -> GlobalSign Root CA
    NotBefore  2026-09-02T05:49:37Z
    NotAfter   2026-12-01T06:45:52Z   (90-day)
    Serial     00EAFE0AF0DF6020350E5E84C52E3DCA77
    SHA1       C7C9006866072B9AF17EAC087CEC56639411ECE7

The wildcard covers every label on the apex, on the leaf itself.

## 16. Edge geography

`CF-RAY` datacenters observed: **SIN** (Singapore) and **HKG** (Hong Kong).
`cf-cache-status: DYNAMIC` on all responses — nothing is cached, every request
reaches the origin. `Alt-Svc: h3=":443"` — HTTP/3 advertised.

## 17. Route space is exhausted at three

Probed 60 route names across ban/auth/user/report/config/debug families.
Only three respond; everything else is a flat 404.

    GET      /ping      200  json
    GET      /health    200  text/plain
    GET      /healthz   200  text/plain
    OPTIONS  /ping      200  2 bytes
    POST     /ping      404  route is GET-only

There are no ban, unban, user, lookup, report, auth, or admin routes. The
"RepScripts Ban API" is a shell with a liveness route and nothing behind it —
there is no data on this host to retrieve.

## 18. Tenant sweep — 410 probes, no hidden hosts

82 labels × 5 suffixes (`""`, `test`, `-test`, `-dev`, `-staging`, `-prod`)
against the A record. Six hosts resolve, all previously known:

    api.repscripts.com             104.21.82.23 / 172.67.151.91
    docs.repscripts.com            104.21.82.23 / 172.67.151.91
    store.repscripts.com           104.21.82.23 / 172.67.151.91
    hyperdata.repscripts.com       104.21.82.23 / 172.67.151.91
    hyperdatatest.repscripts.com   104.21.82.23 / 172.67.151.91
    repscripts.com                 no A record (browser-only, Cloudflare)

Tenant labels are real customer names, not dictionary words, so wordlist
enumeration cannot find them. Only two are observable: one from CT
(`hyperdatatest`, first certified 2026-03-05) and one from
`<customer>` → `<customer>test` inference (`hyperdata`).

## 19. robots.txt — Cloudflare managed stub, not a route map

`/robots.txt` returns 1,248 bytes on `api`, `hyperdata`, and `hyperdatatest`
simultaneously, served by Express, with **no `Disallow` lines at all**. The body
is Cloudflare's managed AI content-signal template — EU DSM Article 4 boilerplate
with `search`/`ai-input`/`ai-train` defined and nothing expressed. It confirms a
Cloudflare Managed Robots configuration is active on those three hosts and
discloses nothing about routes.

`docs.repscripts.com/robots.txt` is GitBook's default and explicitly permissive:

    User-agent: *
    Content-Signal: ai-train=yes, search=yes, ai-input=yes
    Allow: /
    Sitemap: https://docs.repscripts.com/sitemap.xml

The documentation corpus is affirmatively licensed for model training and
retrieval by its owner.

`store.repscripts.com` returns 403 on every path including `/robots.txt`, and
`repscripts.com` has no A record at all — neither is reachable without going
through Cloudflare.

## Consolidated exposure

| # | finding | severity | surface |
|---|---------|----------|---------|
| 1 | unauthenticated production telemetry (566→574 conns, 62 datasets) | high | hyperdata/health |
| 2 | wildcard CORS + all mutating methods | high | api/* |
| 3 | two products on one hostname, both self-identifying | high | api/ping, api/health |
| 4 | full doc corpus as unauthenticated markdown | medium | docs/llms-full.txt |
| 5 | wildcard cert, no DNS wildcard — release path | medium | *.repscripts.com |
| 6 | three real email addresses public | medium | GitHub org + commit metadata |
| 7 | operator identified to a named individual | medium | GitHub profiles |
| 8 | test tenant mirrors prod schema (59 vs 62 datasets) | low | hyperdatatest/health |
| 9 | Helmet double-mounted, immutable cache on 404s | low | api 404 handler |
| 10 | no CAA, no DMARC, no security.txt | low | apex |
| 11 | zero credentials in 1,119 files / 113 commits | — | clean |



## Tooling

`leakmap.py` in this directory reproduces every result above and generalizes
to any host:

    python leakmap.py api.repscripts.com --tenant hyperdata
    python leakmap.py repscripts.com --deep --json
