# SecureBrand Telegram collector: research and plan

Status: **plan for review (version 2). No code is written and nothing has been run against Telegram.**
All times in this document are **Iraq time (UTC+3)**.
Version 1: 1 November 2026. Version 2 (same day): updated with your answers, see §1.

---

## 0. Summary in plain language

**Your decisions:**

- **No Telegram account** of any kind.
- **No bot** (clients don't own the channels).
- **Thousands** of channels and public groups.
- **Keywords that change** over time.
- The data is for **brand monitoring, not AI or machine learning**.

That leaves one way to read Telegram directly: **logged out**, through Telegram's public web pages. It can be combined with an optional **paid data vendor**.

**What logged-out access can do well:**

- **Public channels:**
  - channel info and how it changes;
  - every post with text, links, photos, polls and forward source;
  - **rounded** view counts ("1.2K");
  - reactions;
  - history going back;
  - search inside each channel.
- **Comments under channel posts:** through Telegram's comments widget. How complete it is will be measured in Phase 0.
- **Keyword mentions:** in everything we collect. When you add a keyword, we also search what we already stored.

**What it cannot do, and what I recommend instead:**

- **Public groups have no web feed.** Logged out, a group can only be read **one message per request** through the post widget. That is fine for a few dozen quiet groups, but **not for thousands**. I recommend a "priority groups" mode for the ones that matter most. For wider group coverage, test a vendor (TGStat) in Phase 0.
- **No exact view counts, no forward counts, no exact deletion times.**
- **No search across all of Telegram.** Mentions are complete for every channel we watch. Discovering mentions *elsewhere* needs a vendor. We also suggest new channels to watch, based on forwards and links we see.

**Speed is the real limit.** Everything goes through one server's IP address, and we will not rotate IPs to get around Telegram's limits. How fresh thousands of channels can be depends on how many requests per second Telegram tolerates from one IP, which Phase 0 measures:

- at 1 request per second, about 3,000 channels split as **300 priority channels checked every 15 min**, 900 every hour and the rest every ~4 hours;
- the system automatically gives priority channels their freshness first.

**Rules:** Telegram's terms (since 3 Feb 2026) prohibit "data scraping" in general and forbid AI/ML use. You've confirmed the data is not used for AI/ML. The general scraping clause still applies to logged-out collection; it is recorded as risk R1 (§11). We will also **follow `t.me/robots.txt`** (test W0).

**Cost:** about **US$45–60 a month** for one server plus storage at ~3,000 channels. An optional vendor costs extra (TGStat from about 2,100 ₽ a month).

**Next step:** you approve this plan → I build a small read-only measuring tool in a few small PRs → a 7–10 day **Phase 0** on your server → final numbers and settings.

---

## 1. Decisions

### 1.1 Your answers (1 November 2026)

| # | Question | Your answer |
|---|---|---|
| 1 | Recommendations and use of data | You want my recommendations. The data is used for **brand monitoring, not AI or machine learning**. |
| 2 | Keywords and scale | Keywords are **dynamic**. Channels and public groups number **in the thousands**. |
| 3 | Accounts | **No Telegram account** of any kind. |
| 4 | Client-owned channels | **None.** So a bot can't be used. |
| 5 | Facebook project repo | **Can't be attached.** So this project is designed standalone; nothing from it is assumed. |

### 1.2 My recommended defaults

You can change any of these later; they are settings, not code.

| Topic | Default |
|---|---|
| Access | Logged-out only: profile page, web feed, post widget, comments widget. A vendor is optional. |
| Priority tiers for channels | **P1** (brands' competitors and the most important channels, about 10%); **P2** (important, about 30%); **P3** (everything else). You tag channels; the system adapts within each tier. |
| Freshness targets (new posts) | P1 ≤ 15 min, P2 ≤ 60 min, P3 ≤ 4 h at 1 request per second. Faster if Phase 0 shows more capacity (§9.5). |
| History backfill for a new channel | **90 days** of posts (can be raised per channel). Comments: **14 days**. |
| Re-checking a post's views and reactions | For **30 days** after posting (P1: up to 90 days). |
| Comments | All posts in **P1** channels, plus **any post that mentions a keyword**. Elsewhere only the comment count, if the feed shows it. |
| Public groups | **Priority-groups mode** for a limited set (size fixed after Phase 0, likely tens). Wider group coverage only through a vendor if Phase 0 shows one works. |
| Media | Metadata for everything. Photos kept only for **P1 channels and keyword-matched posts**. No video or file downloads (mostly not available logged out anyway). |
| Retention | Channel posts 24 months; comments and group messages 12 months; photos 90 days; metric snapshots thinned to daily after 90 days; raw pages **never stored**. |
| Pseudonymization of people | **On by default** for commenters and group members. |
| Discovering new channels | Suggest channels seen in forwards, @mentions and `t.me` links. A person approves each one; nothing is added automatically. |
| Infrastructure | Standalone: Go services, Postgres, Valkey, S3-compatible object storage, Prometheus/Grafana, Docker Compose on **one Hetzner server**, GitHub Actions CI. No Kafka at first; an "outbox" can feed Kafka or webhooks later. |
| Outputs | REST API plus outgoing webhooks for mentions and alerts; email alerts for operations. |
| Identity on the web | An honest User-Agent naming SecureBrand, **one fixed IP address**, and we obey `robots.txt`, `429` and `Retry-After`. **No proxies, no IP rotation.** |

---

## 2. How this research was done, and what could not be checked

- **Nothing was run against Telegram.** No page on t.me was fetched.
- **This session's network blocks** telegram.org, core.telegram.org, TGStat and Hetzner. So I used:
  - **Official Telegram sources on GitHub:**
    - Telegram Desktop's API schema (layer 232);
    - TDLib's documented API and source (Telegram's own library);
    - the Bot API 10.3 specification (24 Aug 2026);
    - a copy of Telegram's official **widget script** (`telegram-widget.js`), which shows the exact URLs the post and comments widgets load.
  - **Archived copies of Telegram's legal texts** (Open Terms Archive).
  - **Web search summaries** of official pages I couldn't open.
  - **Source code of open-source tools** that read the web feed (snscrape, RSSHub, minet, TeleGlance, BroadcastChannel and others).
- **Tags used below:**
  - **[V]** verified in an official Telegram source;
  - **[S]** from a search summary of an official page; re-check;
  - **[3P]** third-party;
  - **[P0]** measured in Phase 0.
- To re-check the [S] items word for word, allow `telegram.org` and `core.telegram.org` in the environment's network settings: open the cloud environment menu in the session title bar, choose Edit, then Network access. Steps: <https://code.claude.com/docs/en/cloud-environments#network-access>.

---

## 3. Telegram's rules

| Text | What it says | Tag |
|---|---|---|
| **Terms of Service**, added 3 Feb 2026 | "Telegram additionally prohibits data scraping as part of its Content Licensing and AI Scraping Terms, which apply to all users, businesses, and third-party services accessing the platform." | [V] |
| **Content Licensing and AI Scraping Terms** | "Access to user-generated content for any purpose other than ordinary, legitimate, and intended use of the Telegram platform as its user is prohibited." | [S] |
| same | Prohibits "scraping, indexing, harvesting, aggregation or use of data … to train, fine-tune, validate or otherwise engage in the development, enhancement, benchmarking or deployment of artificial intelligence, machine learning models and similar technologies". | [V] (quoted in the Open Terms Archive memo) |
| **API Terms of Service** 1.5 | No use of Telegram data for AI/ML "development, enhancement or deployment". Only relevant to accounts and apps; we use none. | [V] |
| **Bot Developer Terms** 4.3 | "Always prohibited uses include any form of data collection aimed at creating large datasets, machine learning models and AI products, such as scraping public group or channel contents." We use no bot. | [S] |
| **Privacy Policy** | "everything you post in public will be accessible to everyone." | [V] |

**What this means now:**

- **AI/ML clause:** you've confirmed the data is used for brand monitoring, not AI or ML. The plan enforces it. Telegram data is only processed by rules (keyword matching, counts, time series), and the plan records that it must **not** be sent to any ML system.
- **General scraping clause:** it still applies to logged-out collection. Whether brand monitoring is "ordinary, legitimate, and intended use" is unclear. This is **risk R1**. It is your call; a short legal check is still worth having.
- **Practical consequence:** Telegram can slow down or block our server's IP. There is no account to ban.
- **We will follow `t.me/robots.txt`.** If it disallows the pages we need, we stop and talk before going further (test W0).

---

## 4. The ways of access (research record)

### 4.1 Logged out: profile page, web feed, widgets ← **chosen**

| Page | What it gives | Tag |
|---|---|---|
| `t.me/<name>` | Profile card: title, description, photo, subscriber count ("12 345 subscribers" style). Also tells us whether a username exists and whether it is a channel, a group or a person. | [3P] |
| `t.me/s/<name>` | **Web feed** (channels only). Header: title, username, description, photo, counters (subscribers, photos, videos, files, links). About 20 posts per page [P0]. A "more" link (`data-before`) goes back in history and `data-after` goes forward; `?q=<words>` searches inside the channel. | [3P] |
| `t.me/<chat>/<id>?embed=1` | **Post widget**: one message. Works for **public channels and public groups** ("You can embed messages from public groups and channels anywhere"). The widget script loads exactly this URL. Options include `comment` (one comment), `userpic`, `dark`. | [V] URL from the widget script; [S] groups |
| `t.me/<channel>/<id>?embed=1&discussion=1` | **Comments widget**: the comments under one channel post (the channel needs a linked discussion group). Option `comments_limit`. How many comments it really returns, and how "load more" works logged out: [P0]. | [V] URL and options from the widget script |

**What a post in the feed shows** (from parser code [3P]):

- **Basics:** id, date, text with formatting, author signature, "forwarded from" name and link, reply preview, "via bot".
- **Media:**
  - photos (image URLs);
  - videos (thumbnail, duration);
  - voice notes;
  - documents (title and size only);
  - stickers;
  - locations.
- **Polls:** question, options, percentages, voter count.
- **Link previews.**
- **Views, rounded** ("1.2K").
- **Reactions:** emoji and counts.
- An **"edited"** marker.
- Maybe a **comments count** (several parsers look for one "when visible"; [P0]).

**What it does not give:**

- exact views;
- forward counts;
- the numeric id of a forward's source (only its name or link);
- edit times;
- deletion events (only absence);
- **any group feed**: groups only work message by message through the post widget;
- channels whose preview Telegram disabled (RSSHub mentions porn, copyright and politics) [3P];
- large media.

| | |
|---|---|
| **Needs** | Nothing; one server with a fixed IP. |
| **Limits** | Not documented, per IP. One academic tool pauses 0.5 s between requests by default [3P]; that is not a safe number for us. [P0] |
| **Risk** | The IP can be slowed (429) or blocked; the HTML can change. There is no account to lose. |
| **ToS** | Covered by the general scraping clause (§3, R1). |

### 4.2 A bot (Bot API 10.3): **not used** (no client-owned channels)

- A bot only sees chats it is added to, and channels only as an admin.
- It receives new posts, edits, reaction totals (admin, delayed a few minutes) and comments in real time.
- It gets **no view counts**, **no history from before it joined** and **no deletions** [V].
- Useful only if a client ever owns a channel or group and adds it.

### 4.3 A user account (MTProto): **not used** (your decision)

- The only way to get exact counts, full comment threads, groups read efficiently (100 messages per request), and Telegram's own search across all public channels. That search needs Premium, a daily free quota, then Stars [V].
- It needs a real phone number, and the account can be banned or frozen.
- Recorded here only so the trade-off is documented. **Reconsider only if groups or cross-Telegram search become essential and no vendor works.**

### 4.4 Third-party data services: **optional, tested in Phase 0**

**TGStat** has a large index of public Telegram channels and chats. They use their own collection; we would only call their API.

| API | What it does | Monthly prices [S] (rubles, 5% VAT incl.) |
|---|---|---|
| **Search API** | Post search and keyword mentions; billed by requests and **number of unique keywords** | START 2,100 ₽ · S 4,200 ₽ · M 12,600 ₽ · L 47,250 ₽ |
| **Stat API** | Channel statistics and subscriber history | free tier · S 2,100 ₽ · M 8,190 ₽ · L 20,370 ₽ · XL 60,900 ₽ |
| **Callback API** | Real-time keyword alerts | START 8,400 ₽ · S 29,820 ₽ · … |

Unknowns:

- Whether **groups' messages** are searchable (their docs talk about "posts" of channels);
- coverage of Iraqi and Kurdish channels;
- terms on storing data;
- paying rubles from Iraq;
- sign-up may need a person's Telegram login on their website (your decision; it is not our system using an account).

Others (Telemetr.io; listening suites such as Brand24, YouScan, Talkwalker, Meltwater, Brandwatch) list Telegram but were not verifiable from here. Scraping resellers (Apify, ScrapeCreators) read the same web pages we would, so they add little.

### 4.5 Side-by-side

✅ yes · ◐ partly · ❌ no

| Data | Logged out (chosen) | Bot | User account | Vendor |
|---|---|---|---|---|
| Channel info and changes | ✅ | ✅ | ✅ | ✅ |
| Subscriber count | ✅ (rounding [P0]) | ✅ | ✅ exact | ✅ with history |
| Posts: text, time, links, polls, forward source | ✅ (source as name or link) | ✅ new only | ✅ | ✅ |
| Photos | ✅ | ✅ | ✅ | ◐ |
| Videos and files | ◐ thumbnails, metadata | ✅ ≤ 20 MB | ✅ | ◐ |
| Views | ◐ rounded | ❌ | ✅ | ✅ |
| Reactions | ✅ | ✅ | ✅ | ◐ |
| Forward count | ❌ | ❌ | ✅ | ◐ |
| Comment count | ◐ [P0] | ❌ | ✅ | ◐ |
| Edits | ◐ flag + text change | ✅ | ✅ | ❌ |
| Deletions | ◐ by absence | ❌ | ◐ | ❌ |
| History | ✅ depth [P0] | ❌ | ✅ | ✅ |
| Comments | ◐ widget [P0] | ✅ | ✅ | ◐ |
| Public groups | ◐ **one message per request** | ✅ if owner adds it | ✅ | ? [P0] |
| Search inside a channel | ✅ | ❌ | ✅ | ✅ |
| Search across all public channels | ❌ | ❌ | ✅ | ✅ |

---

## 5. Recommendation

1. **Collect public channels logged out**, at a measured, steady rate from one IP, with priority tiers and automatic slow-down when capacity is short (§9.5). This is the core of the system.
2. **Comments** through the comments widget, for P1 channels and keyword-matched posts. Phase 0 checks completeness against the post's comment count.
3. **Public groups in "priority-groups mode":** read message by message through the post widget, for a limited, approved set. Size is set by Phase 0 cost; likely tens of groups, not thousands. Be clear with clients that **thousands of groups cannot be covered logged out**.
4. **Mentions:**
   - match every keyword on everything we collect, live;
   - when a keyword is added, search our stored data (retroactive);
   - optionally, search inside P1 channels with `?q=` to reach older history.
5. **Discovery:** suggest new channels from forwards, @mentions and `t.me` links we see; a person approves each one.
6. **Test TGStat in Phase 0** (5 keywords, a trial plan). If it covers Iraqi and Kurdish channels and groups, and you can pay, add it as an extra source for mentions outside our watched list. Otherwise skip it.
7. **No AI/ML processing** of Telegram data, **no proxies or IP rotation**, **obey `robots.txt`**.

---

## 6. What is realistic for each thing you asked for (logged out only)

| You asked for | Realistic? | Notes |
|---|---|---|
| Channel title, username, description, photo, subscriber count | **Yes** | Profile page and feed header. Count precision [P0]. |
| Linked discussion group | **Partly** | We can see that a post has comments (widget works). The group's own name/id may not be visible [P0]. |
| How channel info changes over time | **Yes** | Snapshots; a new row only when something changes, plus count points. |
| Post text, time, links, polls, forward source | **Yes** | Forward source as a name or link, not an id. |
| Photos / videos / files | **Photos yes**; video and files metadata only | |
| Views | **Rounded only** | "1.2K" means 1,150–1,249. Good for trends, not for exact numbers. |
| Reactions | **Yes** | |
| Forward count | **No** | |
| Reply/comment count | **Maybe** [P0] | |
| Edits | **Yes, without exact time** | We see the "edited" flag and the changed text at our next check. |
| Deletions | **Partly** | "Gone between check A and check B". |
| Comments under posts | **Likely**, completeness [P0] | |
| Public groups (messages, replies; no member lists) | **Only a limited priority set** | One request per message. Member lists are never collected. |
| New posts detected quickly | **Yes, by priority** | P1 ≈ 15 min at 1 req/s; faster if Phase 0 allows. |
| History backfill per new channel | **Yes** | 90 days by default; deeper if needed [P0 depth]. |
| Mentions across public channels | **Complete for watched channels.** Outside them: only via a vendor, or by growing the watch list from discovery suggestions | |
| Dynamic keywords | **Yes** | Live matching plus retroactive search over stored data (§9.11). |

---

## 7. What I still need from you

Only things I can't decide for you:

1. **The lists:**
   - channels and public groups (usernames or public `t.me/<name>` links; invite links are rejected), each with a priority tier (P1/P2/P3) and a reason (competitor, brand-related, watch);
   - the first set of keywords, with languages and variants.
   - Rough counts are enough to size the server; I assumed **3,000 channels and 1,000 groups**.
2. **A Linux server** (I recommend below) and your OK to run Phase 0 on it. Phase 0 uses the same IP address the real system will use.
3. **Budget OK** for about US$45–60 a month, and whether to try TGStat (paid in rubles).
4. **Who reads alerts and erasure requests** (an email address or a webhook URL), configured on the server, never in the repo.

---

## 8. Requirements and monthly cost

### 8.1 Checklist

| Item | Notes |
|---|---|
| Channel, group and keyword lists | §7. |
| One Linux server (Ubuntu 24.04 LTS) with a fixed public IPv4 | **Hetzner CX43** (8 vCPU, 16 GB RAM, 160 GB disk) in an EU location, plus a **250 GB volume** for Postgres. If CX is unavailable, a dedicated CCX plan. |
| S3-compatible object storage | Hetzner Object Storage, for photos and backups. |
| Docker and Docker Compose on the server | Commands in §10. |
| Telegram account, bot token, phone number, api_id | **None.** |
| Proxies | **None.** IP rotation would be getting around Telegram's limits. |
| Optional vendor | TGStat trial (Phase 0), then a plan if useful. |
| Secrets on the server | Database password, S3 keys, vendor API key (if any), alert webhook. In `/etc/securebrand-tg/` (`0600`); never in git, logs or chat. |

### 8.2 Monthly cost

Prices come from search summaries and third-party reviews [S/3P]; Hetzner pages are blocked here. Hetzner raised prices on 1 April and 15 June 2026, and new-server availability has been limited since late June 2026 [3P]. Check live prices.

| Item | ≈ per month |
|---|---|
| Hetzner CX43 | €16.5 |
| Server backups (+20%) | €3.3 |
| 250 GB volume | €12–15 |
| Object storage (1 TB + 1 TB egress included) | €6.49 |
| **Total** | **≈ €40–45 ≈ US$45–55** |
| *If a dedicated CCX server is needed instead* | add ≈ €70+ |
| *Optional TGStat* | Search API from 2,100 ₽ (START) to 12,600 ₽ (M); Stat API from 2,100 ₽ |

### 8.3 Storage estimate

Assumptions: 3,000 channels × 8 posts a day ≈ **24,000 posts a day**; comments for P1 channels and matched posts; 30 priority groups.

| Data | First year |
|---|---|
| Posts | ≈ 9 M rows ≈ 18 GB |
| Metric snapshots | ≈ 15 GB |
| Comments and group messages | ≈ 10–20 GB |
| Search indexes | ≈ 15–25 GB |
| 90-day backfill | ≈ 4 GB |
| **Postgres total** | **≈ 60–80 GB** → the 250 GB volume is enough for 2+ years |
| Photos (P1 and matched posts only, 90-day retention) | ≈ 30–60 GB steady, inside the 1 TB object storage |

---

## 9. Architecture

### 9.1 Principles

- **Read-only and logged out.** The only outgoing requests are GETs to `t.me` pages and Telegram's image CDN.
  - No login, no forms.
  - We never vote in polls or react.
  - We never follow invite links (`t.me/+…`, `joinchat`); the registry rejects them.
- **One fixed IP, honest User-Agent, `robots.txt` obeyed, `429` and `Retry-After` obeyed, no proxies.**
- **Everything goes through one capacity manager** (§9.5). No code can call Telegram without it.
- **Minimal data:** only fields we use; raw HTML is parsed in memory and thrown away.
- **Privacy at the door:** the suppression list and pseudonymization apply **before** anything is stored or sent on.
- **Rules only, no AI/ML** on Telegram data.
- Telegram dates are stored in UTC; reports and alerts show Iraq time.

### 9.2 Components

```
 registry (channels, groups, tiers, keywords, approvals, suppression list)
      │
      ▼
 scheduler ──▶ capacity manager (one budget for the IP; priorities; 429/Retry-After; circuit breaker)
      │                    │
      ▼                    ▼
 fetcher workers (feed pages, profiles, post widget, comments widget, in-channel search, images)
      │   parse in memory → normalized records
      ▼
 ingest (suppression check, pseudonymize, dedupe, versions, metrics) ──▶ Postgres
      │                                                                   ▲
      ├──▶ mention matcher (live) ──▶ mentions ──▶ outbox ──▶ API / webhooks (Kafka later)
      │                                   ▲
      │        retro search job (when keywords change) ──┘
      ├──▶ discovery (forwards, @mentions, t.me links) ──▶ review queue
      └──▶ image fetcher (P1 + matched only) ──▶ object storage
 privacy service: retention, erasure, suppression, audit      monitoring: Prometheus, Grafana, alerts
```

### 9.3 Services (Go, one repository, deployed with Docker Compose)

| Service | Job |
|---|---|
| `registry` | API and CLI for channels, groups, tiers, keywords, approvals, suppression list. Validates usernames, rejects invite links and private chats. |
| `scheduler` | Decides what is due: new-post checks, metric refreshes, comments, group polls, profile snapshots, backfill steps, deletion checks, in-channel searches. |
| `fetcher` | Workers that make the HTTP requests through the capacity manager and parse the pages. Selectors live in **one versioned file**, so a markup change is a one-file fix. |
| `ingest` | Normalizes, checks suppression, pseudonymizes, dedupes on (chat, message id), stores versions and metric snapshots, writes the outbox. |
| `matcher` | Live keyword matching on every new or edited item; the retroactive search job; discovery suggestions. |
| `media` | Downloads photos for P1 and matched posts, within size caps. |
| `privacy` | Retention deletes, erasure requests, suppression list, audit log. |
| `api` | REST API for SecureBrand (mentions, posts, channels, metrics) and outgoing webhooks from the outbox. |

**Shared:** Postgres 16 (with `pg_trgm`), Valkey (capacity state, locks, caches), object storage, Prometheus, Grafana, Alertmanager.

**Jobs** use a Postgres queue (`FOR UPDATE SKIP LOCKED`), which is simple and enough at this scale. The outbox can publish to Kafka later if SecureBrand's platform wants it.

### 9.4 Data model (main tables)

| Table | Contents |
|---|---|
| `sources` | Username, kind (channel, group), tier, reason, client, status (active, no-preview, not-found, suppressed), collection settings, approvals. |
| `source_snapshots` | Title, description, photo hash, subscriber count; a new row only on change, plus regular count points. |
| `posts` | Chat, message id, date, text and entities, media summary, forward source (name, link, original post link), reply-to, album, signature (**personal data**, pseudonymizable), edited flag, content hash, first and last seen, deletion window. |
| `post_versions` | Old text and media after edits. |
| `post_metrics` | Append-only: views (rounded value and text as shown), reactions (JSON), comment count, captured time. |
| `comments` and `group_messages` | Chat, message id, thread (post), reply-to, `author_key`, date, text, edited flag, deletion window. |
| `authors` | `author_key`; names only if pseudonymization is off, encrypted. |
| `keywords` | Client, term, variants, language, match type (word, phrase, hashtag, @username, domain), exclusions, active from/to, version. |
| `mentions` | Keyword, item, matched text position, how found (live, retro, in-channel search, vendor), keyword version. |
| `discovery_candidates` | Username, how often seen, from where, keyword hits, review status. |
| `media` | Item, type, size, S3 key, sha256, expiry. |
| `outbox`, `erasure_requests`, `suppression_list`, `audit_log` | Delivery to SecureBrand; privacy. |

`posts`, `comments`, `group_messages` and `post_metrics` are **partitioned by month**, so retention deletes are cheap (drop a partition).

### 9.5 Capacity: how thousands of channels share one IP

- Phase 0 measures the **safe rate R** for our IP: the highest rate with no `429` over several days, times about 0.7.
- The capacity manager hands out requests in priority order:

| Order | Task | Share |
|---|---|---|
| 1 | P1 new-post checks | |
| 2 | Comments for P1 and matched posts | |
| 3 | P2 new-post checks | |
| 4 | Priority groups | |
| 5 | Metric refreshes not covered by feed polls | |
| 6 | P3 new-post checks | |
| 7 | Profile snapshots | |
| 8 | Backfill and retroactive in-channel searches | at most 15% |

- **One feed request does two jobs.** A feed page returns the newest ~20 posts *with* their views and reactions. Checking for new posts therefore also refreshes metrics of recent posts at no extra cost.
- **Adaptive per channel:**
  - Channels that post rarely, or only at certain hours, are checked less often, within their tier's maximum delay.
  - Night hours **01:00–07:00 Iraq time** are slower unless a channel posts at night.
  - A channel that posts more than ~20 times between checks gets extra pages so nothing is missed.
- **When demand is higher than R,** lower tiers stretch automatically and the dashboard shows the real freshness achieved per tier. Nothing breaks; P3 just gets slower.

**What freshness 3,000 channels can get** (300 P1, 900 P2, 1,800 P3; 70% of capacity for new-post checks, before adaptive savings):

| Safe rate from one IP | P1 checked every | P2 every | P3 every |
|---|---|---|---|
| 0.5 request/s | 30 min | 2 h | ~8.5 h |
| **1 request/s** | **15 min** | **1 h** | **~4.3 h** |
| 2 requests/s | 5 min | 1 h | ~3.3 h (or P1 10 min, P2 30 min, P3 75 min) |

At 5,000+ channels everything is proportionally slower. A vendor or a narrower list would then be the way to keep freshness.

### 9.6 Rate limiting, retries and circuit breaker

- **429:** wait exactly `Retry-After` (or 60 s if missing) plus random extra. Lower R by half, then raise it slowly (~5% per quiet hour) back to the measured safe rate, never above it.
- **5xx and timeouts:** exponential backoff with jitter (1 s → … → 15 min), with a maximum number of attempts per job type, then a **dead-letter queue** with the reason.
- **Circuit breaker:** after repeated failures, pause all fetching for a while and alert. A challenge or CAPTCHA page means **stop everything and alert**; we never try to get past it.
- **Idempotent jobs:** writes are upserts on (chat, message id), so a retried job is harmless.
- **Parser drift:** if more than 5% of pages fail to parse, or a known field disappears, alert "markup changed" and keep raw counters, not raw HTML.

### 9.7 Schedules

**New-post checks:** by tier and adaptivity (§9.5).

**Metric refresh by post age** (views, reactions, comment count):

- at 15 min, 1 h, 3 h, 6 h, 12 h, 24 h, 2 d, 3 d, 7 d, 14 d and 30 d (P1: weekly up to 90 d);
- mostly served by feed pages (one page covers ~20 neighbouring posts).

**Profile snapshots:**

- P1: 4× a day at about **00:00, 06:00, 12:00 and 18:00 Iraq time** (± a few minutes of jitter);
- others: daily at about **04:00 Iraq time**.

**Backfill of a new channel:**

1. Read the profile.
2. Page back through the feed to 90 days, at most 15% of capacity.
3. Pause on any limit; resume from a saved point.
4. Then comments for the last 14 days of P1 and matched posts.

**Deletion checks:** compare the feed with what we stored at 1 h, 24 h, 7 d and 30 d. A post missing between its neighbours is confirmed with its single-post page. We store the window ("between 10:00 and 11:00 Iraq time").

### 9.8 Comments

- **Source:** the comments widget for a post.
- **When to fetch:**
  - when the post's comment count in the feed changes (if the feed shows it [P0]);
  - otherwise at 1 h, 6 h, 24 h, 72 h and 7 d after the post.
- **Completeness check:** comments we have ÷ the post's comment count. Below 98% after a re-fetch → alert and mark the thread "partial". Phase 0 tells us whether the widget can return full threads ("load more").
- **Edits and deletions** of comments are found the same way as for posts, at the scheduled re-fetches.
- **Authors** are pseudonymized by default (§9.12). The widget shows names and photos; we keep neither unless pseudonymization is switched off.

### 9.9 Priority-groups mode

Public groups have **no feed**. Each message is a separate page (`t.me/<group>/<id>?embed=1`).

**How it works:**

1. Find the current newest message id once (a short search over ids, about 20–40 requests).
2. Then each check requests the next ids until a few in a row don't exist (deleted messages leave gaps).
3. **Cost per check** ≈ new messages + 3–5. A group with 300 messages a day costs at least 300 requests a day.
4. Replies are visible as a reply preview, so threads can be rebuilt.
5. Member lists are never collected.

**Budget:** groups get a fixed share of capacity. Each group's cost is shown on the dashboard, so you can choose which ones are worth it.

**Feasibility, cost and edge cases** are tested in Phase 0 (test G1).

### 9.10 Media

- **Metadata always.**
- **Photos** downloaded (from Telegram's image CDN) only for P1 channels and keyword-matched posts. Each is stored once by sha256 and deleted after 90 days or on erasure.
- **Videos and files:** thumbnail and metadata only.

### 9.11 Dynamic keywords and mentions

**Managing keywords:**

- Keywords are managed through the API or CLI with versioning and active dates.
- Each can have variants, exclusions and a match type: word, phrase, hashtag, @username, web domain, or **forward from a brand channel**.
- A change takes effect within seconds: matchers reload when notified by Postgres `LISTEN/NOTIFY`.

**Text normalization** (Arabic, Kurdish, English):

- remove diacritics and tatweel;
- unify alef forms (أ إ آ → ا) and ى/ي; optionally ة/ه;
- handle Kurdish letters (ک ی ە ۆ ێ) carefully so words aren't broken;
- Latin case folding;
- a variants list per keyword for transliterations and misspellings.

**Live matching:**

- every new or edited post, comment, group message, caption, poll and link preview;
- all keywords at once with a multi-pattern matcher (Aho–Corasick), then word-boundary checks.

**Retroactive search** when a keyword is added or changed:

- searches stored normalized text (Postgres trigram index, partitioned by month) within the retention window;
- matches are saved with `found_via = retro`;
- for P1 channels it can also run the channel's own `?q=` search to reach history older than our backfill, at low priority.

**Outputs:**

- mentions go to the outbox → webhook or API;
- alerts arrive within one check cycle of the channel's tier.

**Discovery:**

- forwards from unknown channels, @usernames and `t.me` links in posts become **suggestions**, ranked by frequency and keyword hits;
- a person approves; nothing is added automatically.

### 9.12 Personal data: retention, pseudonymization, erasure, suppression

| Data | Default retention |
|---|---|
| Channel info | while watched + 12 months |
| Channel posts | 24 months |
| Metric snapshots | thinned to daily after 90 days |
| **Comments and group messages** | **12 months** |
| Photos | 90 days |
| Raw pages | never stored |

**Pseudonymization (on by default):**

- Logged out, commenters and group members appear with a display name, sometimes a link to their username, and a photo.
- We store `author_key = HMAC-SHA256(per-client secret key, normalized display name + username if shown)`, plus `HMAC(username)` when a username is visible, so erasure by username works.
- Names, usernames and photos are **not stored**.
- Signed post authors are treated the same way.
- Note: without an account we don't see Telegram's numeric user ids. Two different people with the same display name could get the same key when no username is shown. Pseudonymized analytics accept this.

**Erasure** (CLI and API; audited):

- **By person** (username or display name):
  1. Add them to the **suppression list first**.
  2. Delete their comments, group messages, signatures, mentions and photos.
  3. Purge caches.
  4. Send an erasure event through the outbox so downstream SecureBrand systems delete too.
  5. Record completion.
- **By channel or group:** the same for everything from that source; it is then marked suppressed and never collected again.
- **Backups** expire within the retention window. After any restore, the **erasure log is replayed** before the system serves data.

**Suppression list:**

- Contains hashed usernames and display names, channels and groups, with scope (one client or all) and a reason.
- Checked by the registry, scheduler, ingest (content from suppressed people is dropped before writing), media and outputs.
- **Erased data never comes back.**

**Other safeguards:**

- Logs contain no message text, names or usernames; only ids and hashes.
- Member lists are never collected.

### 9.13 Credentials and repository safety

- **Secrets** (database password, S3 keys, optional vendor key, alert webhook) live in `/etc/securebrand-tg/` (`0600`, service user only) or Docker secrets. They are never in git, logs, command-line arguments or chat.
- `.gitignore` already excludes env files, keys, session files and Phase 0 output.
- CI runs a secret scanner (gitleaks).
- **The repo is public:**
  - test fixtures are **synthetic**, made by a generator from a fixed seed;
  - CI fails if `testdata/` contains anything the generator didn't produce;
  - real pages and real data are never committed.

### 9.14 Testing: the "fake Telegram"

- **Fake t.me server** in tests serving synthetic profile pages, feeds, post widgets, comment widgets and group message pages, with the same structure as the real ones.
- It uses fictional channels and people, with generated Arabic, Kurdish and English text.
- Switches simulate `429` with `Retry-After`, 5xx, slow responses, "no preview" redirects, deleted messages, edits, a channel posting faster than we poll, and markup changes.
- **Unit tests:** parser, normalizer, Arabic and Kurdish normalization, matcher, capacity manager (fake clock), privacy rules.
- **Integration tests:** Postgres, Valkey and MinIO in containers (testcontainers-go) in CI; full flows against the fake server, including **erasure plus suppression end-to-end** ("erased data never comes back").

### 9.15 Monitoring and alerts

**Metrics (Prometheus):**

- requests by page type and result;
- 429s and `Retry-After` seconds;
- current rate R;
- queue lag;
- **real freshness per tier** (post time → first seen, p50 and p95);
- refresh schedule adherence;
- comment completeness;
- group costs;
- parser failure rate;
- dead letters;
- retention and erasure jobs;
- disk use.

**Alerts** (email or webhook, times in Iraq time):

- repeated 429s or a challenge page;
- parser failures > 5%;
- P1 freshness target missed for 30 min;
- completeness < 98%;
- erasure overdue;
- disk > 80%;
- backups failing.

**Dashboards:** Grafana.

---

## 10. Phase 0: measure before building

**Goal:** real numbers for logged-out collection at your scale, from **the server and IP the system will use**, over **7–10 days**, with strict caps.

**Tool:** a small read-only Go command, `tgprobe`, built in the first PRs after approval.

- It has a hard cap on total requests and on rate.
- It **stops at the first `429`, block or challenge**.
- It writes **only aggregated numbers** (counts, timings, which fields exist, rounding, error codes). No text, no names.
- Output goes to `/var/lib/securebrand-tg/phase0/` on your server, never to git.

**Sample:**

- 50 channels (Arabic, Kurdish, English; busy and quiet), with full collection;
- the **full channel list** for a one-request-each availability check, at 1 request every 5 s (3,000 channels ≈ 4 hours);
- 5 public groups (quiet and busy);
- 5 keywords.

**Tests (each runs only after you approve it):**

| # | Test | Measures |
|---|---|---|
| W0 | `t.me/robots.txt` (you can run it now, below) | Whether Telegram's crawler rules allow the pages we need |
| W1 | Profile + feed header for the 50 channels | Fields present; subscriber count exact or rounded |
| W2 | Feed pages of 5 channels back through history at 1 request / 5 s | Posts per page; how far back; views rounding; reactions; comment count; edited flag; forward source; media; whether compression and "not modified" answers work |
| W3 | **Rate test:** 1 request every 10 s → 5 s → 2 s → 1 s → 0.5 s, 2 hours per step, **stop at the first 429**; then 72 hours at about 70% of the highest clean step | **The safe rate R**: the number that decides freshness (§9.5) |
| W4 | Comments widget on 50 posts with comments | Comments returned vs the post's count; whether "load more" works logged out |
| W5 | In-channel search `?q=` with the 5 keywords (Arabic and Kurdish) | Works or not; paging; vs our own matching |
| W6 | Availability over the full list | Share of channels with no preview, wrong usernames, groups listed as channels |
| W7 | Single-post pages for posts that disappeared from feeds | Deletion detection |
| W8 | Image URLs fetched again after 24 h | Whether Telegram's image links expire |
| G1 | The 5 groups through message pages for 3 days | Does it work logged out; requests per group per day; gaps; reply previews |
| V1 | Optional TGStat trial with the 5 keywords | Coverage of our channels and groups, Iraqi and Kurdish content, extra mentions found, terms, whether payment works |

**How the results decide the settings:**

- **Rate:** R = about 70% of the highest clean rate in W3. Tier intervals come from the §9.5 table.
- **Comments:** if W4 reaches ≥ 98% → comments as planned. Otherwise the scope is limited to P1 and reported as "partial".
- **Groups:** the G1 cost per group sets how many priority groups fit in the group share.
- **Vendor:** V1 decides whether TGStat is worth paying for.
- **robots.txt:** if W0 disallows these pages, **we stop and talk** before anything else.

**Report:** `docs/phase0-report.md`, aggregated numbers only, plus an updated plan.

**Timeline:**

| Days | Work |
|---|---|
| 1–3 | Small PRs that build `tgprobe` and the fake server |
| 4 | You set up the server (commands below) |
| 5–14 | Tests, including the 72-hour run. Daily summary at 09:00 Iraq time. |
| 15 | Report |

### Commands you can run now

Test W0, Telegram's crawler rules. It contains no personal data; send me what it prints:

```bash
curl -sS https://t.me/robots.txt
```

Create a dedicated system user:

```bash
sudo useradd --system --create-home --home-dir /var/lib/securebrand-tg --shell /usr/sbin/nologin securebrand-tg
```

Create the secrets folder, readable only by that user:

```bash
sudo install -d -m 0700 -o securebrand-tg -g securebrand-tg /etc/securebrand-tg
```

Create the Phase 0 output folder:

```bash
sudo install -d -m 0700 -o securebrand-tg -g securebrand-tg /var/lib/securebrand-tg/phase0
```

Show the server's public IPv4 address. Phase 0 and the real system must use this same one:

```bash
curl -4 -sS https://ifconfig.me
```

### Commands for later (once the `tgprobe` PRs are merged; flags may change)

Put the sample list on the server (you edit it; it is not committed to git):

```bash
sudo -u securebrand-tg nano /etc/securebrand-tg/phase0-sample.yaml
```

Run test W1:

```bash
sudo -u securebrand-tg /opt/securebrand-tg/bin/tgprobe web profile --sample /etc/securebrand-tg/phase0-sample.yaml --out /var/lib/securebrand-tg/phase0
```

---

## 11. Main risks

| # | Risk | Effect | What we do |
|---|---|---|---|
| R1 | **Telegram's general scraping clause** (since Feb 2026) | Telegram could object to the collection; clients could be exposed | Brand monitoring only, no AI/ML; minimal data; obey robots.txt and limits; a short legal check is still advised; stop if Telegram objects |
| R2 | **IP slowed or blocked** | Freshness drops or collection stops | Measured safe rate, automatic back-off, alerts; no IP rotation; a vendor as fallback |
| R3 | **Not enough capacity** for thousands of channels from one IP | P3 channels checked only every few hours | Tiers, adaptive polling, dashboard of real freshness; narrow the list or add a vendor |
| R4 | **Public groups can't be covered at scale** logged out | Most groups missing | Priority-groups mode; vendor test; tell clients clearly |
| R5 | **HTML changes** | Parser breaks | Selectors in one file; drift alarm; fake-server tests |
| R6 | **No preview** for some channels (some political ones may be affected) | Those channels are unreadable | Detected in W6 and flagged; vendor may cover them |
| R7 | **Rounded numbers** | Views are approximate | Store both the shown text and the parsed value; show ranges in reports |
| R8 | **Comments incomplete** if the widget limits them | Partial threads | Completeness metric; mark "partial" |
| R9 | **Personal data and law** (GDPR if EU/UK people or clients are involved; local law) | Complaints, fines | Retention, pseudonymization, erasure plus suppression; a short DPIA with your adviser |
| R10 | **Vendor issues** (rubles, sanctions, coverage, their terms) | No vendor option | Phase 0 trial before paying |

## 12. Open questions (answered by Phase 0 or a legal check)

1. What does `t.me/robots.txt` allow? (W0, you can run it now.)
2. The safe request rate from our IP. (W3)
3. Does the feed show comment counts? Does the comments widget return full threads? (W2, W4)
4. Does the post widget work for public **group** messages logged out, and at what cost per group? (G1)
5. How far back can the feed go? Do image links expire? (W2, W8)
6. Exact wording of the Content Licensing terms (page blocked here; I have a search summary).
7. Does TGStat cover groups and Iraqi/Kurdish channels, and can it be paid for from Iraq? (V1)

## 13. Implementation roadmap (after you approve)

Each item is a small **draft PR** with unit and integration tests and green CI. I review each one and merge when CI is green, as you asked.

**Before Phase 0:**

| PR | Content |
|---|---|
| 1 | Repo skeleton: Go module, Makefile, CI (lint, tests, gitleaks, fixture check), contribution notes. |
| 2 | Capacity manager: one budget for the IP, priorities, `429`/`Retry-After`, AIMD, circuit breaker; fake-clock tests. |
| 3 | Page parsers (profile, feed, post widget, comments widget, group message) with selectors in one file; synthetic fixture generator; fake t.me server. |
| 4 | `tgprobe` Phase 0 command with hard caps and aggregated reports; integration tests against the fake server. |

→ **Phase 0 run and report**, then a plan update.

**Phase 1** (re-planned after Phase 0):

- database schema and migrations;
- registry with suppression list;
- scheduler;
- fetcher;
- ingest with versions and metrics;
- comments;
- priority groups;
- keywords, live matcher and retroactive search;
- discovery queue;
- privacy service;
- media;
- API, outbox and webhooks;
- monitoring;
- deployment (Docker Compose), with your server commands in each PR.

## 14. Sources

**Official Telegram sources (read via GitHub):**

- Telegram Desktop API schema, layer 232: <https://github.com/telegramdesktop/tdesktop/blob/dev/Telegram/SourceFiles/mtproto/scheme/api.tl>
- TDLib API with documentation: <https://github.com/tdlib/td/blob/master/td/generate/scheme/td_api.tl>
- TDLib post-search code: <https://github.com/tdlib/td/blob/master/td/telegram/MessageQueryManager.cpp>
- Bot API 10.3 specification (generated from the official docs): <https://github.com/PaulSonOfLars/telegram-bot-api-spec>
- Copy of Telegram's widget script (post and comments widget URLs): <https://github.com/tonkeeper/tonkeeper-web/blob/HEAD/apps/web/src/telegram-widget.js>

**Archived official legal texts (Open Terms Archive):**

- Terms of Service: <https://github.com/OpenTermsArchive/contrib-versions/blob/main/Telegram/Terms%20of%20Service.md>
- API Terms of Service: <https://github.com/OpenTermsArchive/contrib-versions/blob/main/Telegram/Developer%20Terms.md>
- Privacy Policy: <https://github.com/OpenTermsArchive/contrib-versions/blob/main/Telegram/Privacy%20Policy.md>
- Memo on the 3 Feb 2026 change: <https://github.com/OpenTermsArchive/opentermsarchive.org/blob/main/content/memos/telegram-prohibits-collecting-data-for-ai-use/index.md>

**Official pages seen through search (please re-check live):**

- <https://telegram.org/tos/content-licensing>
- <https://telegram.org/tos/bot-developers>
- <https://core.telegram.org/widgets/post>
- <https://core.telegram.org/widgets/discussion>
- <https://core.telegram.org/bots/features>
- <https://core.telegram.org/method/channels.searchPosts>
- <https://core.telegram.org/api/premium>
- <https://telegram.org/blog/post-search-story-albums-and-more>

**Web-feed parsers (code read for what the pages contain):**

- snscrape: <https://github.com/JustAnotherArchivist/snscrape/blob/master/snscrape/modules/telegram.py>
- RSSHub: <https://github.com/DIYgod/RSSHub/tree/master/lib/routes/telegram>
- minet: <https://github.com/medialab/minet/blob/master/minet/telegram/scraper.py>
- TeleGlance: <https://github.com/NWGKGIT/TeleGlance>
- BroadcastChannel: <https://github.com/miantiao-me/BroadcastChannel>

**Libraries (for the record):**

- gotd/td: <https://github.com/gotd/td>
- Telethon: <https://github.com/LonamiWebs/Telethon>
- Pyrogram: <https://github.com/pyrogram/pyrogram>
- TDLib: <https://github.com/tdlib/td>

**Prices and vendors (search summaries):**

- TGStat prices: <https://tgstat.com/p/prices>
- TGStat API intro: <https://api.tgstat.ru/docs/ru/start/intro.html>
- Hetzner 2026 prices: <https://costgoat.com/pricing/hetzner>, <https://www.bitdoze.com/hetzner-cloud-cost-optimized-plans/>
- Bellingcat toolkit on TGStat: <https://github.com/bellingcat/toolkit/blob/main/gitbook/tools/tgstat/README.md>
