# security-brief watcher

Lightweight news watcher for the daily security briefing page.
Goal: keep the page fresh with **zero missed news** and **~zero agent usage**.

## How it works

`check_feeds.py` (Python 3, stdlib only) polls RSS / JSON / HTML sources with
plain HTTP fetches, compares items against a persistent seen-set
(`seen.json`), and prints **only the new items** as JSON.

- No browser, no LLM, no JS rendering. A poll is just HTTP + parsing.
- Conditional GET (`If-Modified-Since` / `If-None-Match`) so unchanged feeds
  return HTTP 304 with (near-)zero body.
- Only items published in the last **3 days** are candidates. Older items are
  never "new" (feeds list newest first), so they are skipped, never stored,
  and never re-reported. This keeps `seen.json` small (~300 KB, ~500 items)
  instead of ballooning to the full history of every feed.
- Broad/general feeds (Yonhap, ZDNet Korea, etnews, Hacker News) are
  keyword-filtered to security terms so the briefing is not flooded with
  non-security news. Dedicated security feeds are unfiltered.
- Items matching a tracked ongoing incident (Optimizely, Java4Shell, rERP,
  MetaM, Fugaku, N-able, …) are tagged in the `ongoing` field so the rebuild
  can file them under "key issues / ongoing incidents".

It does **NOT** rebuild the page. See "Update trigger" below.

## Pipeline strengthening — 2026-11-19 (NGINX2Root miss fix)

The 2026-11-19 NGINX2Root incident was collected in the feed but never
promoted to a key issue: the static `ONGOING_KEYWORDS` list had no
NGINX2Root/nginx/123007 entry, so all 6 NGINX articles got `ongoing=[]`
while Java4Shell articles in the same batch got 2 tags. The fix:

1. **Auto-detect ongoing incidents** (`auto_detect_ongoing()`): when 3+
   items (new this run + seen in the last 24 h) share the same CVE ID
   (e.g. `CVE-2026-123007`), the same digit-token (e.g. `nginx2root`,
   `java4shell`), or the same distinctive word (e.g. `nginx`, `apache`),
   they are auto-tagged as ongoing. Multi-source requirements apply
   (CVE: ≥3 items any sources; digit-token: ≥3 from ≥2 sources; plain
   word: ≥3 from ≥3 sources), word groups need CVE/token anchoring
   (≥2 members also carry a CVE ID or digit-token), and a stopword list
   blocks generic words (`chrome`, `microsoft`, `malicious`, `exploited`,
   …). A frequency cap (`AUTO_WORD_MAX=50`) prevents a word that appears
   in >50 items from auto-tagging. Korean transliterations are aliased
   (`엔진엑스` → `nginx`) so Korean titles group with English ones.
   The static `ONGOING_KEYWORDS` list is kept as-is.
2. **Tagger word-boundary fix** (`tag_ongoing()`): English keywords now
   match on word boundaries (regex `\b…\b`), so `Tor` no longer
   false-positives on `Actor` or `Factory`. Numeric CVE suffixes
   (`114938`, `123007`) and Korean keywords still use substring matching
   (titles contain them as `CVE-2026-123007` or inside Korean text).
   Verified: `Actor` → no tag, `Factory` → no tag, `Tor network…` →
   `['Tor']`, Java4Shell → `['Java4Shell', '114938']`.
3. **Breaking detection** (`is_breaking()`): an item is flagged
   `breaking` if (a) CVSS ≥ 9.0, (b) it was auto-tagged as ongoing, or
   (c) its title contains `critical`, `zero-day`, `RCE` (word-boundary)
   or `긴급`, `비상`, `치명적` (substring). Output JSON now includes
   `breaking_count` and `breaking_items`; each item has a `breaking`
   boolean. The hook prompt tells the updater to prioritize breaking
   items (feed top + key-issues section when ongoing-tagged).
4. **Feed expansion 56 → 82** (+26, all verified live before adding):
   SentinelOne, Google Security Blog, Kaspersky Securelist, Recorded
   Future, SANS ISC, The DFIR Report, Chrome Releases, CISA
   Cybersecurity Advisories, Volexity, Ars Technica (keyword-filtered),
   Elastic Blog (keyword-filtered), Snyk Blog, Deep Instinct, Cynet,
   Arctic Wolf, Red Canary, Orca Security, Huntress RSS, Flashpoint,
   Aqua, Sysdig, Imperva, Avast (keyword-filtered), Heimdal, Cofense,
   Acunetix. No working domestic (Korean) feeds were found — NoCut News,
   IT Chosun, Digital Daily, iNews24 all failed (malformed XML or 403);
   domestic stays at 10. Feed counts now: total 82 (kisa 1, cve 9,
   domestic 10, intl 61, ransomware 1).
5. **Polling 30 → 15 min**: hook `security-brief-watch`
   `poll_interval_secs` 1800 → 900 (06:00–24:00 KST window unchanged,
   enforced by the hook script).
6. **Daily key-issue review cron** (`security-brief-daily-review`,
   daily 06:00 KST, review-only): flags key issues with no new article
   for 3+ days (demote/remove candidate) and incidents with 5+ feeds in
   the last 24 h but no key-issue panel (promotion candidate). Reports
   only — it never modifies the page.

Verified 2026-11-19 (final full run): **82/82 feeds OK, 0 failed**,
89 new items, 12 breaking, 14 with ongoing tags (Java4Shell, ShadowCloud,
OPTIMJACK, nginx, apache, CVE-2026-123007 — all meaningful, no noise
tags). Task-required 6-title test: 5/6 NGINX2Root titles auto-tagged
(`nginx`); the 6th (The Register, title contains no NGINX/CVE token)
is flagged breaking but cannot be grouped by title alone. All 6 are
breaking. A poll now takes ~70–130 s wall clock (sequential network
waits; CPU < 2 s).

## Usage

```bash
cd ~/workspace/security-brief
# first run: mark everything currently in the feeds as seen
python3 watcher/check_feeds.py --seed
# poll: print only new items as JSON (exit 0 = new items, 2 = none, 1 = error)
python3 watcher/check_feeds.py [--out watcher/new_items.json]
# only some categories
python3 watcher/check_feeds.py --categories cve,domestic
```

Output JSON: `{checked_at, feeds_ok, feeds_failed, new_count, new_items[]}`.
Each new item: `{source, category, title, link, published, key, ongoing[]}`.

Verified 2026-11-18: seed = 498 items, 26 feeds OK, 0 failed; two
consecutive polls after seeding report **0 new items** (exit 2). A poll takes
~25–60 s wall clock (sequential network waits; CPU < 1 s).

**Feed expansion 2026-11-18 09:15–09:20:** 26 → **56 feeds** (+30).
Three verification runs after the expansion: run 1 = 56 OK / 1 failed
(NSHC 403 — removed), 119 new items from the new feeds; run 2 = 55 OK /
1 failed (GitHub Advisories 403, transient shared-IP rate limit), 1 new
item (genuine Yonhap story); run 3 = **56 OK / 0 failed, 1 new item**
(a brand-new NVD CRITICAL CVE, CVE-2026-131852, CVSS 9.8). A poll now
takes ~60–95 s wall clock (sequential network waits; CPU < 1 s).
`seen.json` grew 498 → 619 items (~324 KB) — still small.

## Feed inventory (all verified live 2026-11-19)

82 feeds, 5 categories (KISA 1, CVE 9, domestic 10, international 61, ransomware 1). Every URL below was fetched with curl and returned
real content; nothing here is guessed. The table below lists the original 56 feeds from the 2026-11-18 expansion; the 26 feeds added in the 2026-11-19 expansion (all international) are defined in `check_feeds.py` (verified 2026-11-19: 82/82 feeds OK, 0 failed).

### KISA / KrCERT — 1 feed (no RSS; HTML list scrape)
| Source | URL | Notes |
|---|---|---|
| KISA 보안공지 | https://www.boho.or.kr/kr/bbs/list.do?bbsId=B0000133&menuNo=205020 | Parses `nttId` + title from the list page (15 items/page). Detail URL built as `.../view.do?...&nttId={id}`. KISA pages give **no CVSS** — scores come from vendor/secondary sources. |

### Critical CVEs (CVSS 9.0+) — 9 feeds
| Source | URL | Notes |
|---|---|---|
| CISA KEV | https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json | Full catalog (~1,800 entries, 1.8 MB). Only entries with `dateAdded` in the last 3 days are candidates. |
| NVD API | https://services.nvd.nist.gov/rest/json/cves/2.0/?resultsPerPage=100&pubStartDate={3d ago}&pubEndDate={now}&cvssV3Severity=CRITICAL | No API key needed (limit 5 req/30 s; we make 1 req/run). Returns ~56 CRITICAL CVEs for a 2-day window. A 429 is treated as "skip this run", not a crash. |
| CISA Advisories **(new)** | https://www.cisa.gov/uscert/ncas/alerts.xml | Legacy US-CERT alerts feed (10 items). Low frequency (latest 2026-10-19 at verification) — monthly-ish. Cheap to poll (conditional GET). |
| Exploit-DB **(new)** | https://www.exploit-db.com/rss.xml | 50 items. Exploit write-ups (RCE etc.). Latest 2026-11-11 at verification — a few per week. |
| GitHub Advisories **(new)** | https://api.github.com/advisories?per_page=20 | JSON API, 20 newest advisories. Parser keeps **critical/high only** (medium/low would flood the CVE section). Unauthenticated limit is 60 req/hour **per shared egress IP** — it 403s when the shared IP's quota is used up (observed 50/60 used, 1 transient 403 in verification). The watcher skips the feed that run, no crash; conditional GET (ETag) means 304s do not consume quota. A GitHub token would fix it, but that needs a secret — out of scope. |
| Ubuntu Security Notices **(new)** | https://ubuntu.com/security/notices/rss.xml | 10 items. Ubuntu CVE fix notices (USN). |
| Wordfence **(new)** | https://www.wordfence.com/blog/feed/ | 10 items. WordPress vulnerability reports. Feed is large (~3.8 MB full fetch) — conditional GET keeps repeat polls cheap. Latest 2026-11-12 at verification. |

### Korean domestic — 10 feeds
| Source | URL | Notes |
|---|---|---|
| Boannews (보안뉴스) | https://cdn.boannews.com/rss/gn_rss_allArticle.xml | Dedicated security outlet. |
| Dailysecu (데일리시큐) | https://cdn.dailysecu.com/rss/gn_rss_allArticle.xml | Dedicated security outlet. |
| Yonhap society | https://www.yna.co.kr/rss/society.xml | Keyword-filtered (security incidents are society news, but so are local stories). |
| Yonhap news | https://www.yna.co.kr/rss/news.xml | Keyword-filtered (general news). |
| ZDNet Korea | https://feeds.feedburner.com/zdkorea | Keyword-filtered (general IT). |
| etnews AI/SW | http://rss.etnews.com/04.xml | Keyword-filtered (IT/SW section). |
| ASEC (AhnLab) **(new)** | https://asec.ahnlab.com/ko/feed | AhnLab's threat-intel blog (Korean). Recent items: Java4Shell advisory, dark-web reports, APT trends. (The `/feed` English path 404s; the `/ko/feed` path works.) |
| ITWorld Korea **(new)** | https://www.itworld.co.kr/feed/ | Keyword-filtered (general IT). 20 items, recent. |
| CNET Korea **(new)** | https://www.cnet.co.kr/feed/ | Keyword-filtered (general IT). 15 items, recent — but 0/15 matched security keywords at verification (games/gadgets/promos). Long-tail safety net for big security stories; near-zero yield most days. |
| Datanet **(new)** | https://www.datanet.co.kr/rss/allArticle.xml | Keyword-filtered (general IT). 50 items, recent; 8 matched at verification. |

### International — 61 feeds
| Source | URL | Notes |
|---|---|---|
| BleepingComputer | https://www.bleepingcomputer.com/feed/ | |
| The Hacker News | https://thehackernews.com/feeds/posts/default | |
| SecurityWeek | https://www.securityweek.com/feed/ | |
| The Record | https://therecord.media/feed | |
| Krebs on Security | https://krebsonsecurity.com/feed/ | |
| Dark Reading | https://www.darkreading.com/rss.xml | (Sub-feeds `/threat-intelligence/rss.xml`, `/attacks-breaches/rss.xml` are 403 — main feed only.) |
| Security Affairs | https://securityaffairs.com/feed | |
| Infosecurity Magazine | https://www.infosecurity-magazine.com/rss/news/ | |
| Help Net Security | https://www.helpnetsecurity.com/feed/ | |
| The Register (security) | https://www.theregister.com/security/headlines.atom | |
| Schneier on Security | https://www.schneier.com/feed/ | |
| Threatpost | https://threatpost.com/feed/ | |
| DataBreaches.Net | https://www.databreaches.net/feed/ | |
| Malwarebytes | https://www.malwarebytes.com/blog/feed | |
| Qualys | https://blog.qualys.com/feed | |
| Hacker News (YC) | https://news.ycombinator.com/rss (keyword-filtered; general tech) | |
| The Cyber Express **(new)** | https://thecyberexpress.com/feed/ | 10 items, recent. |
| CyberScoop **(new)** | https://cyberscoop.com/feed/ | 10 items, recent. |
| Cybersecurity Dive **(new)** | https://www.cybersecuritydive.com/feeds/news/ | 10 items, recent. |
| CSO Online **(new)** | https://www.csoonline.com/feed/ | 20 items, recent. |
| ZDNet security **(new)** | https://www.zdnet.com/topic/security/rss.xml | Keyword-filtered (security topic on a general tech site — some items are general tech). 25 items. |
| Wired security **(new)** | https://www.wired.com/feed/category/security/latest/rss | 20 items, recent. Dedicated security section. |
| ESET WeLiveSecurity **(new)** | https://www.welivesecurity.com/feed/ | 100 items (large feed; recency filter drops old ones). |
| Palo Alto Unit42 **(new)** | https://unit42.paloaltonetworks.com/feed/ | 15 items, recent. Threat research. |
| CrowdStrike **(new)** | https://www.crowdstrike.com/en-us/blog/feed/ | 10 items, recent. |
| HackRead **(new)** | https://hackread.com/feed/ | Keyword-filtered (mixed security news + guides/promos). 10 items. |
| TechCrunch security **(new)** | https://techcrunch.com/category/security/feed/ | Keyword-filtered (security section of a general tech site). 20 items. |
| Security Boulevard **(new)** | https://securityboulevard.com/feed/ | Keyword-filtered (mixed vendor/sponsored content). 10 items. |
| Microsoft Security Blog **(new)** | https://www.microsoft.com/en-us/security/blog/feed/ | 10 items, recent. |
| Google Project Zero **(new)** | https://googleprojectzero.blogspot.com/feeds/posts/default | 10 items. Latest 2026-11-13 at verification (outside the 3-day window, so 0 new until they publish again). Low frequency, high value. |
| Cisco Talos **(new)** | https://blog.talosintelligence.com/feed/ | 15 items. Latest 2026-11-13 at verification (outside the 3-day window). Low frequency, high value. |
| Check Point Research **(new)** | https://research.checkpoint.com/feed/ | 15 items, recent. |
| Tenable **(new)** | https://www.tenable.com/blog/feed | 10 items, recent. |
| IT Security Guru **(new)** | https://www.itsecurityguru.org/feed/ | 10 items, recent. |
| Cloudflare Blog **(new)** | https://blog.cloudflare.com/rss/ | Keyword-filtered (engineering blog — security + performance/infra). 20 items. |
| AWS Security Blog **(new)** | https://aws.amazon.com/blogs/security/feed/ | 20 items, recent. |
| Cisco Security Blog **(new)** | https://blogs.cisco.com/security/feed | 9 items, recent. |

### Ransomware leak victims — 1 feed
| Source | URL | Notes |
|---|---|---|
| ransomware.live (thrunt.me mirror) | https://thrunt.me/corpus/ransomware/ | HTML table (5,602 rows). Only victims discovered in the last 3 days are candidates. The official API is **down** (see below). |

## Dead / blocked feeds and coverage gaps

| Source | Status | Gap / alternative |
|---|---|---|
| **ransomware.live API** (`api.ransomware.live/recentvictims`) | **HTTP 404** (verified 3×, incl. `/` and `/v2` variants) | The API server currently returns the main site's 404 page. Covered instead by the **thrunt.me mirror** (same postings, verified live) + the ransomware.live homepage as a manual fallback. Re-check the API periodically; if it returns, prefer it (lighter than the 1.4 MB mirror page). |
| **GBHackers** (`gbhackers.com/feed/`) | **HTTP 520** (Cloudflare) | Blocked at the edge for plain curl. No usable RSS from this environment. Its stories are usually also covered by BleepingComputer / The Hacker News / SecurityWeek within the hour. |
| **ASEC (AhnLab)** (`asec.ahnlab.com/feed`) | **HTTP 404** | The English-path feed does not exist. **Fixed by using the Korean path** — `asec.ahnlab.com/ko/feed` is now a live feed (see inventory). |
| **etnews 보안 section** | No clean security RSS identified | etnews RSS uses opaque numeric section codes (`rss.etnews.com/NN.xml`); no "보안" feed could be mapped. Covered by `04.xml` (AI/SW, keyword-filtered). |
| **KISA** | No RSS at all | Covered by the list-page HTML scrape (15 latest notices). If KISA changes its board markup, the `parse_kisa` regex needs updating. |
| **Naver news / Naver 보안섹션** | Not polled | Naver has no stable public security RSS; its security stories are syndicated from Yonhap/Boannews/Dailysecu, which are already covered. |
| **Vendor blogs (S2W, 이스트시큐리티, NSHC)** | Not polled | No stable RSS found. Their findings surface via Boannews/Dailysecu/BleepingComputer. |
| **Yonhap** | No dedicated security RSS | Covered by `society.xml` + `news.xml`, both keyword-filtered. Risk: a security story whose title contains none of the keywords would be missed. The keyword list is broad (Korean + English); review it if a story is ever missed. |

### Dead / blocked — found in the 2026-11-18 expansion (all curl-verified)

| Source | Status | Alternative |
|---|---|---|
| **NSHC** (`nshc.net/feed`) | **HTTP 403** (verified working once, then 403 on 4 consecutive tries) | Bot protection kicked in after the first fetch. Removed from the feed list. Findings surface via Boannews/Dailysecu. Re-check occasionally. |
| **S2W** (`s2w.inc/en/blog/rss`, `s2w.inc/blog/rss`) | **HTTP 403** | Bot protection. Covered by Boannews/Dailysecu/BleepingComputer. |
| **SK쉴더스** (`skshieldus.com/feed`) | **HTTP 403** (Access Denied) | Bot protection. Same coverage as above. |
| **SC Media** (`scmagazine.com/feed/`) | **HTTP 403** | Bot protection. CSO / Cybersecurity Dive / SecurityWeek cover the same enterprise-security news. |
| **Tripwire** (`tripwire.com/state-of-security/feed/`) | **HTTP 403** | Bot protection. |
| **Dark Reading sub-feeds** (`/threat-intelligence/rss.xml`, `/attacks-breaches/rss.xml`) | **HTTP 403** | Main Dark Reading feed works; sub-feeds are blocked. |
| **CyberNews** (`cybernews.com/feed/`) | **HTTP 403** | Blocked. BleepingComputer/THN cover the same stories. |
| **Naked Security (Sophos)** (`nakedsecurity.sophos.com/feed/`) | **Connection error** (unreachable, 3 tries) | Sophos's main news feed (`news.sophos.com/feed/`) is also unreachable from here. Covered by other outlets. |
| **Kaspersky Daily** (`daily.kaspersky.com/feed/`, `kaspersky.com/blog/feed/`) | **Connection error** (unreachable, 3 tries) | Kaspersky domains are unreachable from this environment. |
| **CyberSecNews** (`cybersecuritynews.com/feed/`) | **Connection error** (unreachable, 3 tries) | |
| **Adobe PSIRT** (`blogs.adobe.com/psirt/feed/`, `blogs.adobe.com/security/feed/`) | **Connection error** (unreachable) | Adobe security bulletins are covered by Tenable/NVD/CISA. |
| **Snyk** (`snyk.io/vuln/feed/`) | **200 but HTML, not RSS** | The URL serves a web page, not a feed. GitHub Advisories + NVD cover vuln data. |
| **MSRC blog** (`msrc.microsoft.com/blog/feed/`) | **200 but HTML, not RSS** | Microsoft Security Blog feed (in inventory) covers Microsoft security news. |
| **Reddit r/netsec, r/cybersecurity** (`old.reddit.com/r/.../.rss`) | **200 but HTML, not RSS** | Reddit now serves an HTML page instead of RSS to plain curl. No usable public RSS. |
| **SANS ISC** (`isc.sans.edu/rss.xml`, `/rss/`) | **HTTP 404** | The ISC RSS endpoint is gone (error page). |
| **Fortinet blog** (`fortinet.com/blog/feed`, `feeds.fortinet.com/fortinet/blog`) | **HTTP 404** | No working RSS found. Fortinet PSIRT news is covered by Tenable/NVD. |
| **SentinelOne** (`sentinelone.com/feed/`) | **HTTP 404** (empty feed) | No usable RSS. |
| **Rapid7** (`blog.rapid7.com/feed/`, `rapid7.com/blog/feed`) | **HTTP 404** | No usable RSS. |
| **Ars Technica security** (`feeds.arstechnica.com/arstechnica/security`) | **HTTP 404** | No security-section feed at that path. |
| **BleepingComputer topic feed** (`/feed/topic/security/`) | **HTTP 404** | Main BleepingComputer feed (in inventory) covers it. |
| **Trend Micro research** (`trendmicro.com/en_us/research/feed`) | **HTTP 404** | No usable RSS. |
| **F5 Labs** (`f5.com/labs/rss`) | **HTTP 404** | No usable RSS. |
| **ENISA news** (`enisa.europa.eu/news/feed`) | **HTTP 404** | No usable RSS. |
| **NCSC UK** (`ncsc.gov.uk/api/1/services/v1/news-feed/rss.xml`) | **HTTP 404** | No usable RSS at that path. |
| **ecrime.ch** (`ecrime.ch/feed/`) | **HTTP 404** | Ransomware tracker feed not found. |
| **abuse.ch Ransomware Tracker** (`ransomwaretracker.abuse.ch/rss/`, homepage) | **Connection error / HTTP 503** | The tracker itself is down (503). thrunt.me mirror remains the ransomware source. |
| **Boannews English** (`boannews.com/eng/rss/rss.xml`) | **200 but HTML, not RSS** | Korean Boannews feed (in inventory) covers the same stories. |
| **The CyberWire** (`thecyberexpress.com` — `thecyberwire.com/feeds/rss.xml`) | **Brotli-encoded responses** | The feed is valid RSS (40 items) but the server sends Brotli-compressed bodies that stdlib `urllib` cannot decode (no `brotli` module in stdlib). Cannot be polled without adding a dependency — skipped. |
| **Google Security Blog** (`security.googleblog.com/feeds/posts/default`) | **Stale feed** (valid RSS, but latest item 2026-04-23) | The feed has not updated in ~7 months; would never produce new items. Skipped. |
| **Google Cloud Blog / Mandiant** (`cloud.google.com/blog/rss.xml`, `.../threat-intelligence/rss.xml`) | **200 but HTML, not RSS** | Google Cloud blog serves HTML, not RSS. |
| **IBM Security Blog** (`ibm.com/blog/category/security/feed/`) | **200 but HTML, not RSS** | Serves HTML. |
| **SANS NewsBites** (`sans.org/newsletters/newsbites/rss/`) | **200 but HTML, not RSS** | JS-rendered page, no RSS. |
| **The Daily Swig** (`portswigger.net/daily-swig/rss`) | **200 but HTML, not RSS** | Serves HTML. |
| **SC Magazine UK** (`scmagazineuk.com/feed/`) | **Connection error** | Unreachable. |
| **Sophos News** (`news.sophos.com/feed/`) | **Connection error** | Unreachable (same as Naked Security). |
| **CIRCL CVE API** (`cve.circl.lu/api/last`) | **Works, but skipped** | Returns a mix of OSV-format and CVE-5.0-format records; parsing both formats adds complexity for low marginal value over NVD + KEV + GitHub Advisories. Revisit if NVD lags. |
| **ITWorld Korea / CNET Korea / DDaily** (guessed paths) | **404 / HTML** | Found the real feeds via homepage RSS links: `itworld.co.kr/feed/` and `cnet.co.kr/feed/` are now in the inventory. DDaily has no RSS link on its homepage. |

**Known filter quirk (pre-existing, not introduced by the expansion):**
the keyword `감염` (infection) also matches public-health stories — e.g.
Yonhap "질병청, 내년 의료관련감염 감시체계 참여기관 모집" was reported as
new on 2026-11-18. Malware-infection stories usually also contain
`악성코드`/`해킹`/`보안`, so dropping `감염` is an option, but it risks
missing a malware story whose title only says "감염". Left as-is; the
rebuild step (human/agent curation) filters these out.

**Coverage verdict (after the 2026-11-18 expansion):** all 6 briefing
categories are covered by at least 2 independent live sources, most by
5+. The residual gaps (GBHackers, S2W, SK쉴더스, SC Media, NSHC, Naked
Security, Kaspersky, Reddit, SANS ISC) are second-order: their stories
are near-always syndicated by a covered source within hours. The
ransomware category still has a single live source (thrunt.me mirror) —
the official API is down and abuse.ch is down; re-check both
periodically.

## Update trigger (LIVE — hook `security-brief-watch`)

The watcher only **detects** new items. The rebuild + deploy is a separate
step, deliberately kept apart so a poll costs ~nothing and a rebuild only
happens when there is actually something new.

```
every 15 min (06:00–24:00 KST) — hook security-brief-watch, poll_interval_secs 900
  └─ run: python3 watcher/check_feeds.py --out ~/hooks/state/security-brief-new-items.json
       ├─ exit 2 (no new items) → do nothing. Cost: one script run.
       └─ exit 0 (new items) → trigger a rebuild:
            1. A worker/agent reads ~/hooks/state/security-brief-new-items.json
               (fields: new_count, breaking_count, breaking_items,
               new_items[] with ongoing tags and breaking flag)
            2. Rebuilds the briefing page with the new items
               (artifact edit on silsigan-boannyuseu), prioritizing breaking
               items (feed top + key-issues section when ongoing-tagged)
            3. Exports the page, copies to index.html, commits, pushes
               to origin main with the security-brief-deploy key
            4. GitHub Pages serves the new version within minutes
```

Notes on the rebuild step:
- The rebuild is the **expensive** part (research + page build). It runs only
  when `new_count > 0`, which on a normal day is a few times, not 72×/day.
- Overnight (00:00–06:00 KST) the watcher does not run: Korean outlets are
  quiet and the briefing is a daytime product. International overnight news
  is picked up by the 06:00 run (3-day recency window covers it).
- The rebuild must respect the page's standing rules: Apple-like simple
  clean UX, no hallucination (only sourced items), all 9 sections present.
- The daily review cron `security-brief-daily-review` (06:00 KST,
  review-only) separately checks whether key-issue panels are stale or
  missing; it reports findings and never modifies the page.

## Recommended schedule

- **Poll:** every **15 minutes, 06:00–24:00 KST** (73 runs/day) — hook
  `security-brief-watch`, live since 2026-11-18, interval shortened
  30 → 15 min on 2026-11-19.
- **Rebuild:** only on `new_count > 0` (event-driven, not scheduled).
- **Key-issue review:** daily 06:00 KST, cron `security-brief-daily-review`
  (report-only, never modifies the page).

## Usage cost

| Step | Cost |
|---|---|
| One poll (73×/day) | ~0 agent usage. A plain Python script: ~70–130 s wall clock with 82 feeds (sequential network waits), <2 s CPU, ~8–10 MB download on a full fetch, far less when feeds return 304. No LLM, no browser, no credits. |
| One rebuild (only when new items exist) | The expensive step: a research + page-build cycle. A few per day at most, vs 72 if rebuilt on a fixed schedule. This is the whole point of the watcher: **poll cheap, rebuild only on change**. |
| Overnight | Nothing runs 00:00–06:00. |

## Push path (deploy key)

- The page is live at https://seungwooooooo.github.io/security-brief/
  (commit `1b63415`, verified 2026-11-18).
- **Use the dedicated key `~/.ssh/id_ed25519_security_brief`** (deploy key
  `security-brief-deploy`, write access). The old shared "Muse" key
  (`~/.ssh/id_ed25519`) **cannot** be reused here — GitHub forbids the same
  deploy key on two repos, and it is already registered on `vuln-jobs`.
  Verified: `git ls-remote` with the new key returns HEAD `1b63415`.
- Push command:
  `GIT_SSH_COMMAND="ssh -i ~/.ssh/id_ed25519_security_brief -o IdentitiesOnly=yes -o StrictHostKeyChecking=accept-new" git push origin main`

## Files

- `watcher/check_feeds.py` — the watcher (stdlib only, executable).
- `watcher/seen.json` — persistent seen-set (auto-created by `--seed`).
  Seeded 2026-11-18 08:57 KST (498 items). After the 2026-11-18 feed
  expansion (56 feeds): 619 items, ~324 KB. Do not hand-edit; re-run
  `--seed` to reset it.
- `watcher/README.md` — this file.

## What is NOT done yet (needs a decision / the operator)

1. **Hook + daily review cron are live** (hook since 2026-11-18,
   15-min polling + daily review cron since 2026-11-19). The rebuild
   step runs on `new_count > 0` via the hook.
2. The ransomware.live API should be re-checked occasionally; if it comes
   back, switch the ransomware feed to it (lighter than the mirror).
3. GitHub Advisories feed 403s when the shared egress IP's unauthenticated
   quota (60/hr) is used up. A GitHub token would fix it, but that needs a
   secret — decide whether to wire one up. (Note: with 15-min polling the
   quota pressure is higher; the feed is skipped that run when it 403s,
   and NVD + CISA KEV still cover CVE data.)
4. Re-check the dead/blocked list occasionally — NSHC, abuse.ch, and the
   ransomware.live API may come back; GBHackers/SC Media may unblock.
5. **Residual auto-detection limits:** a title that mentions no shared
   token cannot be grouped by title alone (e.g. The Register's "AI found
   more critical bugs in another widely used technology running 225M
   websites" — flagged breaking, but not auto-tagged). The worker still
   sees it as a breaking item and can connect it by reading the article.
   Korean titles rely on the `KO_ALIASES` map (currently only
   `엔진엑스` → `nginx`); add entries if another incident's Korean
   transliteration recurs.
