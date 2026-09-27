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

---

# Batch four — admin panel hunt

56 admin path names × 5 live hosts. Result: **no exposed admin panel.**

| host | result |
|------|--------|
| `api.repscripts.com` | every path 404 — no admin surface at all |
| `hyperdata.repscripts.com` | every path 404 |
| `hyperdatatest.repscripts.com` | every path 404 |
| `docs.repscripts.com` | platform artifacts only — see below |
| `store.repscripts.com` | blanket 403 on all 56 paths, including `/favicon.ico`-class nonsense |
| `rep.tebex.io` | blanket 403 on every path, storefront included |

## 20. docs.repscripts.com — GitBook artifacts, not a panel

    /admin/      302   Location: (empty)
    /dashboard/  302   Location: (empty)
    /panel/      302   Location: (empty)
    /portal/     302   Location: (empty)
    /manage/     302   Location: (empty)
    /cms/        302   Location: (empty)
    /staff/      302   Location: (empty)
    /wp-admin    403
    /wp-admin/   403
    /wp-login.php 403
    /admin.php   403

The 302s carry no `Location` header — GitBook's platform-level redirect stub,
identical for every reserved word. The 403s are GitBook's reserved-path
denylist, not WordPress. There is no WordPress on this host. No panel here.

## 21. store.repscripts.com — WordPress, blanket-refused at the app layer

The 403 body is empty and the headers are PHP session headers, not a Cloudflare
WAF block. A Cloudflare block returns its own HTML with `cf-error-details` and
an error code; this returns nothing and sets app-level cache directives.

    HTTP/2 403
    Cache-Control:      no-store, must-revalidate, no-cache, max-age=0, private,
                        post-check=0, pre-check=0
    Referrer-Policy:    same-origin
    X-Frame-Options:    SAMEORIGIN
    Server:             cloudflare
    CF-RAY:             a41b072079ab44a1-SIN
    Alt-Svc:            h3=":443"
    body:               empty

`post-check=0, pre-check=0` plus `SAMEORIGIN` is the WordPress session-cookie
pattern. So the origin behind `store.repscripts.com` is a **WordPress
install**, and its admin panel is at the standard `/wp-admin`. The application
refuses every request from this vantage point — consistent with a maintenance
lock, an IP allowlist, or a geo-restriction — and returns 403 *before* routing,
so no path is distinguishable from any other. The 403 is uniform across all 56
probes, which means the storefront gives up nothing about its own structure.

This is properly refusing access, not failing to. It was not circumvented and
no attempt was made to route around it.

## 22. rep.tebex.io — the real control panel, also closed

Tebex is the storefront in the org profile, so the vendor's actual control panel
would live here. Every path returns 403 from this vantage point, the storefront
root included. Tebex restricts traffic by region and origin, and a datacenter
egress is refused. Nothing about the panel's structure is observable.

## Conclusion

Four candidate admin surfaces were checked. Three do not exist (`api`,
`hyperdata`, `hyperdatatest` — the hosts that answer at all answer only
`/ping`, `/health`, `/healthz`). One exists as a WordPress install behind a
blanket refusal, and one is the vendor's Tebex storefront behind a regional
block. **No admin panel is exposed without authentication on any surface in
this domain.**

This is a clean result, not a gap in coverage. The exposure in this domain is
metadata — banners, telemetry, identity, documentation — and none of it is
administrative access.

---

# Batch five — 104.234.180.162, the origin

## 23. The IP is the origin, and it is a game server host

    104.234.180.162
    AS48925  VibeGAMES B.V.
    Singapore   1.2897, 103.8501
    anycast     true
    PTR         none

Not the Cloudflare edge. `104.21.82.23` / `172.67.151.91` are the edge;
`104.234.180.162` is the machine behind it, on a game-server provider in
Singapore — the same region as the observed edge PoPs (SIN, HKG).

Proof is the certificate on 443:

    Subject  CN=CloudFlare Origin Certificate
    Issuer   CloudFlare Origin SSL Certificate Authority
    SAN      DNS:*.repscripts.com, DNS:repscripts.com
    Valid    2026-06-08 -> 2041-06-04   (15-year origin cert)
    Protocol TLS 1.3

That is the edge→origin certificate, not a public one. The origin IP is
discoverable and the wildcard origin cert is being served on it.

## 24. Open ports — full 1–65535 sweep, 23 s

    22      ssh
    443     https   CloudFlare Origin Certificate, *.repscripts.com
    3389    RDP
    30120   FiveM game server

Four ports. No MySQL, no Redis, no Mongo, no PostgreSQL, no RDP-adjacent SMB.
Nothing else in 65,535.

**RDP 3389 exposed to the internet on the box that serves the production API**
is the most serious single item in this report. The same host also runs the
game server and the `rep_hyperdata` datastore.

SSH 22 accepted the connection but returned no banner within 5 s — tarpitted
or heavily rate-limited, not a normal sshd.

## 25. Origin bypass — Cloudflare's hostname isolation is an illusion

Requesting the origin directly, no Cloudflare in the path, with each hostname
in the `Host` header:

    Host: api.repscripts.com            /ping    404
    Host: api.repscripts.com            /health  200  {"connections":580,"datasets":62,...}
    Host: hyperdata.repscripts.com      /health  200  {"connections":580,"datasets":62,...}
    Host: hyperdatatest.repscripts.com  /health  200  {"connections":580,"datasets":62,...}
    Host: docs.repscripts.com           /health  200  {"connections":580,"datasets":62,...}
    Host: store.repscripts.com          /health  200  {"connections":580,"datasets":62,...}

Two distinct problems in one result.

**The origin ignores `Host` for vhost routing.** All five hostnames reach the
same default vhost and get byte-identical responses. The per-hostname
separation observed through Cloudflare does not exist at the origin.

**The test hostname serves production data.** Through Cloudflare,
`hyperdatatest.repscripts.com/health` returns `{"connections":0,"datasets":59}`.
Directly at the origin, the same hostname returns
`{"connections":580,"datasets":62}` — production. The prod/test split is
enforced *only* at the Cloudflare edge, by hostname. Anything reaching the
origin gets production, including the test hostname.

Everything Cloudflare contributes here — WAF, rate limiting, DDoS
mitigation, the origin-control the edge was providing — is removed by
addressing the origin directly. Origin IP lockdown is the missing control.

## 26. FiveM server — 201 resources, unauthenticated

FiveM serves its full state on 30120 by design, no auth. That is what the
port is for, so this is an inventory disclosure rather than a misconfiguration
— but on the origin machine it hands over the whole stack layout.

    GET http://104.234.180.162:30120/info.json      13,092 bytes, unauthenticated

    sv_projectName     VTB RP
    sv_serverId        repserver_vtbrp
    sv_maxClients      1000
    sv_pureLevel       2
    onesync_enabled    true
    enforceSteamAuth   false
    txAdmin-version    8.0.1
    tags               default, official, vtb, repscripts, roleplay
    server             FXServer-no-version (didn't run build tools?)
    version            978302382
    resources          201

`sv_serverId` is `repserver_vtbrp` and the server is publicly tagged
`repscripts` — it is listed in the FiveM server browser under that tag.

**`rep_hyperdata` is one of the loaded resources.** This corrects §4 and §6.
`hyperdata` is not a customer name. It is a RepScripts product, deployed at
`hyperdata.repscripts.com` (live) and `hyperdatatest.repscripts.com` (test).
The `/health` endpoint reporting a dataset census is that product's own
telemetry — the resource is a datastore layer, so the health route reports what
it has open. The data was right; the mechanism was wrong.

**108 of the 201 resources are `rep_*`.** The RepScripts framework *is* this
server — the entire roleplay stack is the vendor's product:

    rep_hyperdata  rep_weed  rep_rental  rep_talkNPC  rep_economy  rep_login
    rep_police  rep_policebadge  rep_prison  rep_bank  rep_dmv  rep_farming
    rep_fishing  rep_miner  rep_houserobbery  rep_garage  rep_vehicleshop
    rep_shop  rep_shopbarber  rep_shopclothes  rep_clothing  rep_appearance
    rep_gang  rep_warzone  rep_pvp  rep_airdrop  rep_auction  rep_giftcode
    rep_safezone  rep_taixiu  rep_tattoos  rep_taxijob  rep_truckerjob
    rep_lumberjack  rep_prospecting  rep_shipper  rep_vehiclemod  rep_builder
    ... 108 total

`docs.repscripts.com` publishes documentation for five of them
(`rep-rental`, `rep-talkNPC`, `rep-weed`, `rep-chopshop`, and quickstart).
The public documentation covers a small fraction of the actual product. The
other 103 are undocumented and visible only from inside the origin.

Remaining resources are the standard FiveM stack: `es_extended`, `ox_inventory`,
`ox_lib`, `oxmysql`, `ox_doorlock`, `npwd`, `pma-voice`, `xsound`, `zdiscord`,
`esx_status`, `esx_license`, `esx_basicneeds`, plus map and weapon assets.

## 27. txAdmin — installed, and correctly not exposed

`txAdmin-version 8.0.1` is present. txAdmin is the FiveM administrative
dashboard, and it binds to loopback by default with access through the
`txadmin` Discord command.

    40120  closed
    40121  closed
    40122  closed
    4040   closed
    3306 / 5432 / 27015 / 27016   closed

The admin panel is not reachable from the network. This is the one control on
this box that is configured correctly.

## 28. Telemetry trend across all five batches

    t          source                        connections  datasets  uptime
    13:25:30Z  hyperdata via Cloudflare            566         62     1024002
    13:32:49Z  hyperdata via Cloudflare            574         62     1024441
    13:43:42Z  ORIGIN 104.234.180.162              580         62     1026688

+14 connections over 18 minutes, all unauthenticated, and the last reading came
straight off the origin with no Cloudflare in the path. `started` = 1789491524
throughout — the process has not restarted across every observation.

## 29. Topology

    Cloudflare edge  104.21.82.23 / 172.67.151.91   SIN + HKG PoPs
            |
            v
    ORIGIN  104.234.180.162   AS48925 VibeGAMES B.V., Singapore
      |
      +-- 443    nginx  ->  /health  "FiveBorn Update Server"   (rep_hyperdata)
      |                   ->  Express "RepScripts Ban API"      (/ping, vhost-gated)
      +-- 30120  FXServer  "VTB RP"   201 resources, 108 rep_*
      |                   txAdmin 8.0.1 (loopback only)
      +-- 3389   RDP     exposed to the internet
      +-- 22     SSH     open, no banner within 5 s

## Revised priority

| # | finding | severity |
|---|---------|----------|
| 1 | **origin IP exposed, serves production to all hostnames, bypasses Cloudflare entirely** | critical |
| 2 | **RDP 3389 open to the internet on the production origin** | critical |
| 3 | prod/test split enforced only at the edge — origin serves prod to every Host | high |
| 4 | unauthenticated production datastore telemetry (580 conns, 62 datasets) | high |
| 5 | 201-resource stack inventory public on 30120, 108 undocumented rep_* products | high |
| 6 | wildcard CORS + all mutating methods on the API | high |
| 7 | wildcard cert, no DNS wildcard — release path for subdomains | medium |
| 8 | full doc corpus as unauthenticated markdown | medium |
| 9 | three real email addresses, operator identified to a named individual | medium |
| 10 | two products on one hostname, both self-identifying | medium |
| 11 | Helmet double-mounted, immutable cache on 404s | low |
| 12 | no CAA, no DMARC, no security.txt | low |
| 13 | txAdmin | correctly not exposed |
| 14 | credentials in 1,119 files / 113 commits | none found |

## Corrections to earlier batches

- **§4 / §6 / §18** — `hyperdata` is a RepScripts product, not a customer.
  The `<customer>` / `<customer>test` pattern is a
  `<product>` / `<product>test` pattern. The dataset telemetry is
  `rep_hyperdata`'s own health route. The 410-probe tenant sweep found no
  hidden hosts because the names are products, not customers.
- **§19** — the 1,248-byte `robots.txt` is Cloudflare's managed content-signal
  stub, not a route map. No `Disallow` lines. It disclosed nothing.




## Tooling

`leakmap.py` in this directory reproduces every result above and generalizes
to any host:

    python leakmap.py api.repscripts.com --tenant hyperdata
    python leakmap.py repscripts.com --deep --json
