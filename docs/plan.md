# SecureBrand Telegram collector: research and plan

Status: **plan for review. No code is written and nothing has been run against Telegram.**
All times in this document are **Iraq time (UTC+3)**.
Prepared: 1 November 2026.

---

## 0. Summary in plain language

**Can we collect public Telegram data?** Yes, technically, in three ways:

1. **Logged out (no account).** The public web preview `t.me/s/<channel>` shows a channel's info and its posts. Views are rounded ("1.2K"). Reactions and media are shown, and you can page back through history and search inside one channel. It does **not** work for groups. It does not give exact counts, forward counts or deletions, and some channels have no preview.
2. **A bot.** A bot only sees chats it has been **added to**, and in a channel only as an **admin**. It sees new posts, edits, reaction totals and comments in real time. It sees **no views and no history from before it joined**, and it is never told about deletions. So a bot only helps for **channels our clients own** and add it to.
3. **A logged-in user account (MTProto).** This is the only way that gives everything: exact views, forwards, reactions, complete comment threads, public groups, full history, and **keyword search across all public channels**. Search needs Telegram Premium, has a small daily free quota, and costs Telegram Stars after that. It needs a real phone number and carries a real **risk of the account being banned or frozen**.

**The big catch: Telegram's rules changed.** Since **3 February 2026** the main Terms of Service say Telegram *"prohibits data scraping"* for *"all users, businesses, and third-party services"*. The API terms also forbid using Telegram data to *"train, fine-tune or otherwise engage in the development, enhancement or deployment of artificial intelligence, machine learning models"*. In plain words:

- No method is clearly **allowed** for systematically collecting channels we don't own, including logged out.
- Feeding Telegram data into **ML analysis** (sentiment models, LLM summaries, OCR or logo recognition) is explicitly forbidden. The Facebook project's analysis handoff cannot be copied as it is.
- The cleanest case is **client-owned channels**, read by a bot the client adds as admin, collecting only what is needed.

**My recommendation:**

- Get a **legal opinion first** (I am not a lawyer).
- Then run a 7-day **Phase 0** on about 10 channels to measure the real limits.
- The likely final design is a mix:
  - **logged out** for public channels we don't own;
  - **a bot** for clients' own channels;
  - **one user account** (you choose which) only for what nothing else can do: comments, public groups, cross-channel keyword search and exact counts.
- No ML on Telegram data until a lawyer clears it.

---

## 1. How this research was done, and what could not be checked

- **Nothing was run against Telegram.** No page on t.me was fetched, no account or bot was used.
- **This session's network blocks** telegram.org, core.telegram.org, TGStat and Hetzner (proxy answers `403` to CONNECT). So I could not open Telegram's documentation pages directly. Instead I used:
  - **Official Telegram sources hosted on GitHub:**
    - Telegram Desktop's API schema (`api.tl`, **layer 232**);
    - TDLib's documented API (`td_api.tl`) and its source code (TDLib is Telegram's own client library);
    - the Bot API specification generated from the official docs (**Bot API 10.3, 24 August 2026**).
  - **Archived copies of Telegram's legal texts** from Open Terms Archive, which tracks them: Terms of Service, API Terms of Service, Privacy Policy.
  - **Web search summaries** for official pages that are not mirrored: Bot Platform Developer Terms, Content Licensing terms, the widget pages, the post-search announcement, and vendor prices.
  - **Source code of open-source tools** that read the web preview (snscrape, RSSHub, minet from Sciences Po médialab, TeleGlance, BroadcastChannel). This shows what the preview contains.
  - **Library repositories:** gotd/td, gotd/contrib, Telethon, Pyrogram, TDLib.
- Each fact below carries a tag:
  - **[V]** verified in an official Telegram source (schema, TDLib, Bot API spec, or an archived official text);
  - **[S]** seen through a search summary of an official page I could not open; please re-check;
  - **[3P]** third-party source;
  - **[P0]** unknown until Phase 0 measures it.
- **To let me re-check the [S] items word for word:** allow `telegram.org`, `core.telegram.org`, `tgstat.com` and `www.hetzner.com` in this environment's network settings. Open the cloud environment menu in the session title bar, choose Edit, then Network access, and add them under Allowed domains. Steps: <https://code.claude.com/docs/en/cloud-environments#network-access>.

---

## 2. Telegram's rules: read this first

### 2.1 What the texts say

| Text | What it says | Tag |
|---|---|---|
| **Terms of Service** (telegram.org/tos), added 3 Feb 2026 | "Telegram additionally prohibits data scraping as part of its Content Licensing and AI Scraping Terms, which apply to all users, businesses, and third-party services accessing the platform." | [V] archived copy; date from Open Terms Archive |
| **Content Licensing and AI Scraping Terms** (telegram.org/tos/content-licensing) | "Access to user-generated content for any purpose other than ordinary, legitimate, and intended use of the Telegram platform as its user is prohibited." | [S] |
| same | "firmly prohibits the scraping, indexing, harvesting, aggregation or use of data obtained from its platform to train, fine-tune, validate or otherwise engage in the development, enhancement, benchmarking or deployment of artificial intelligence, machine learning models and similar technologies." | [V] quoted in the Open Terms Archive memo |
| same | Narrow exception: access to "data required to launch and operate a legitimate third-party Telegram Client, Telegram Bot, or Telegram Mini App", licensed only as strictly required to operate that service. | [S] |
| **API Terms of Service** 1.5 (core.telegram.org/api/terms) | "you are prohibited from using, accessing or aggregating data obtained from the Telegram platform to train, fine-tune or otherwise engage in the development, enhancement or deployment of artificial intelligence, machine learning models and similar technologies." | [V] archived copy |
| API ToS 1.4 | Forbidden: "making actions on behalf of the user without the user's knowledge and consent", "tampering with the 'read' statuses of messages", etc. | [V] |
| API ToS 2.1 / 2.3 | You must obtain your own api_id. The app title must not include "Telegram" (unless preceded by "Unofficial"). | [V] |
| API ToS 3.3 | "If your app allows accessing content from Telegram channels, you must include support for official sponsored messages". Whether this applies to a back-end collector is an **open question**. | [V] |
| API ToS 4.1–4.2 | On breach Telegram notifies the account; if not fixed in 10 days, API access is discontinued. | [V] |
| **Bot Platform Developer Terms** 4.3 (telegram.org/tos/bot-developers) | Collect only data essential for the service. "Always prohibited uses include any form of data collection aimed at creating large datasets, machine learning models and AI products, such as scraping public group or channel contents." | [S] |
| **Creating your application** (core.telegram.org/api/obtaining_api_id) | "all accounts that sign up or log in using unofficial Telegram clients are automatically put under observation". Using the API for flooding, spamming or faking subscriber and view counters → permanent ban. Wrongly banned accounts can write to recover@telegram.org. One api_id per phone number. | [S], and quoted in gotd's SUPPORT.md [3P] |
| **Privacy Policy** | "everything you post in public will be accessible to everyone." (Accessible is not the same as licensed for collection.) | [V] |

### 2.2 What this means for us

1. **There is no method that Telegram's terms clearly allow** for systematically collecting public channels and groups we don't own. The scraping clause names "all users, businesses, and third-party services", which includes logged-out access to `t.me`. Whether brand monitoring counts as "ordinary, legitimate, and intended use" is a legal question.
2. **AI/ML use is explicitly forbidden.** This covers *deployment*, not just training. Running a sentiment model, an LLM, OCR or logo detection on Telegram data is very likely covered. **The analysis handoff used in the Facebook project must be reviewed before any Telegram data goes there.** Until it is cleared, the plan only uses rule-based processing (keyword matching, counts, time series).
3. **The best-defended case** is a **client's own channel or group**. The client (the owner) adds our bot as admin, and the bot is a "legitimate Telegram Bot" serving that owner. Even then, Bot Developer Terms 4.3 says to collect only what is essential and never build "large datasets". Comments are still other people's content.
4. **Third-party data vendors** (section 3.4) move the collection away from us. That lowers *our* account risk, but the legal question stays.
5. **Consequences of breaking the rules:** account ban or freeze (user-account path), IP blocking (logged-out path), loss of API access, and possible legal or reputational trouble for SecureBrand and its clients.

**Decision needed from you:** get a written legal opinion (and possibly ask Telegram directly for permission) before collecting from channels we don't own. Phase 0 can start with **client-owned channels only** while that is pending (see §9).

---

## 3. The ways of access, in detail

### 3.1 Logged out: web preview, embeds, profile pages

**What exists:**

- `t.me/<name>`: a profile card with title, description, photo, and subscriber count ("12 345 subscribers" style) [3P].
- `t.me/s/<name>`: the web preview feed [3P].
  - A channel header shows title, username, description, photo, and counters (subscribers, photos, videos, files, links).
  - A page of recent posts follows (about 20 posts; [P0]).
  - Older posts load via the "more" link (`data-before`), newer ones via `data-after`.
  - `?q=<words>` searches inside that channel.
- `t.me/<name>/<id>?embed=1`: one post as an embeddable widget [3P].
- **Discussion widget**: embeds the comments of a public channel post. It works only for channels with a linked discussion group, and has a comment-count limit option [S]. How many comments it really returns: [P0].

**What a post shows** (from parser source code [3P]):

- **Basics:** id, date, text with formatting, author signature (if the channel signs posts), "forwarded from" name and link, reply preview, "via bot".
- **Media:** photos (CDN image URLs); videos (player or thumbnail, duration); round videos; voice notes; documents (title and size only); stickers; locations.
- **Polls:** question, options, percentages, voter count.
- **Link previews.**
- **Views, rounded** ("1.2K"; minet's parser "cleans human-readable numbers").
- **Reactions** (emoji and counts, including paid reactions).
- An **"edited"** marker (no edit time).
- Possibly a **comments counter** (some parsers look for it; [P0]).

**What it does not give:**

- exact view counts;
- forward counts;
- the numeric id of a forward's source (only a name or link);
- edit time or edit history;
- deletions (you can only notice a post is gone);
- full comment threads (unless the widget proves complete, [P0]);
- **public groups at all**: snscrape treats a redirect away from `/s/` as "no public post list" [3P];
- channels whose preview is unavailable. RSSHub warns "some channels are not available in t.me/s", naming porn, copyright and politics [3P]. This could matter for Iraqi political channels.
- Large media (videos and files) mostly cannot be fetched [P0].

| | |
|---|---|
| **Needs** | Nothing. One server with a stable IP and an honest User-Agent. |
| **Limits** | Not documented. Limited per IP. Tools pause about 0.5 s between requests (minet default) [3P], which is not a safe number for us. History depth reachable by paging: [P0]. |
| **Ban risk** | No account to ban. The server IP can be slowed (HTTP 429) or blocked. The HTML can change and break the parser. |
| **ToS position** | Covered by the scraping prohibition (§2). Not an "allowed" route. |

**Small tests (Phase 0, only with your approval):** W1–W7 in §9.

### 3.2 A bot (Bot API, current version 10.3, 24 Aug 2026)

**What it needs:**

- A bot token from @BotFather (free).
- **Channels:** bots can be added only as administrators [3P/S]. No special right is needed just to receive posts, but we should ask for **no rights at all beyond the minimum** (no posting, no deleting).
- **Groups / discussion groups:** the bot must be **admin**, or have **privacy mode turned off** in BotFather and then be re-added [S]. Otherwise it only sees commands and replies to itself.
- Bots don't see other bots' messages, unless a newer bot-to-bot mode is enabled [S].

**What it receives** [V], only *after* it joins:

| Update / method | Gives |
|---|---|
| `channel_post`, `edited_channel_post` | New posts and edits: text, entities, media, `forward_origin`, `author_signature`, `media_group_id`, `edit_date`, polls. |
| `message`, `edited_message` (in the discussion group or a group) | Comments and group messages. The channel post's automatic copy in the discussion group is flagged `is_automatic_forward`. |
| `message_reaction_count` | Reaction totals on channel posts. **Bot must be admin** and must request this update. Updates are "grouped and can be sent with delay up to a few minutes". |
| `message_reaction` | Who reacted (in groups). Admin needed. We would **not** request it. |
| `getChat` → `ChatFullInfo` | Title, username, description, photo, `linked_chat_id`, `has_protected_content`, `has_visible_history`, and more. |
| `getChatMemberCount` | Subscriber or member count. |

**What it does not get** [V]:

- **No view counts.** The `Message` type has no views field.
- No forward counts and no reply counts.
- **No deletions.** Only business-account deletions exist, which don't apply.
- **No history from before it joined.** There is no history method; a bot only receives updates.
- Nothing from chats it isn't in.
- No search.
- Bots also can't use user-only MTProto methods such as `channels.getChannelRecommendations` [S].

| | |
|---|---|
| **Limits** | Undelivered updates are kept about 24 hours [S]. Files up to 20 MB through the cloud Bot API (more with a self-hosted Bot API server) [S]. Reading has no practical rate problem. |
| **Ban risk** | Very low if the bot only listens. |
| **ToS position** | The best of the three, because the channel owner invites it. Bot Developer Terms 4.3 still apply (minimal data, no datasets, no AI). We need a privacy policy for the bot (Telegram offers a standard one). |
| **Fit** | **Only for chats our clients own** (or whose owners agree). |

### 3.3 A logged-in user account (MTProto)

**What it needs:**

- A real phone number and a Telegram account using it.
- `api_id` and `api_hash` from my.telegram.org, registered on that same number (one api_id per number [S]).
- A client library.
- A **session file**, which is a full login to the account.
- Two-step verification (2FA) on the account.
- Optional **Telegram Premium**: needed for global post search [S/V]. It also raises the join limit from 500 to 1000 channels and groups [3P from Telegram Desktop source] and removes download speed limits (Premium ToS 7.5 [V]).

**What it can read from a public channel without joining** [V for the methods and fields; [P0] for real-world behaviour]:

| Method | Gives |
|---|---|
| `contacts.resolveUsername` | Turns `@name` into an id plus `access_hash`. **Heavily rate-limited** in practice [3P]. RSSHub's code says "contacts.ResolveUsername is heavily rate limited"; one report describes 15–18 hour bans after a burst of about 160 calls. **Resolve once and cache forever.** |
| `channels.getFullChannel` | `about`, `participants_count`, `linked_chat_id` (discussion group), pinned message, available reactions, `can_view_stats` (admins), and more. |
| `messages.getHistory` | Up to **100 messages per call**, back to the first post. Each message has: `views`, `forwards`, `replies` (comment count plus the discussion group id), `reactions` (counts per emoji), `fwd_from` (source peer, original post id, original author, date), `edit_date`, `post_author`, `grouped_id` (albums), `media` (all types, including polls), `entities`, `factcheck`. |
| `messages.getMessagesViews(ids, increment=false)` | Refreshes views, forwards and replies for many posts in one call. **We always pass `increment=false`.** Inflating view counters is exactly what Telegram bans. |
| `messages.getMessagesReactions(ids)` | Refreshes reactions. |
| `channels.getMessages(ids)` | Re-checks given posts. A deleted post comes back empty, which is how we **detect deletions and edits** without joining. |
| `messages.getDiscussionMessage` and `messages.getReplies(channel, post_id)` | The **comment thread** of a post, which lives in the linked discussion group. TDLib: "Message thread of a channel message is in the channel's linked supergroup." Up to 100 per call. |
| `messages.search(peer, q)` | Keyword search inside one channel. |
| `channels.searchPosts(query or hashtag)` | **Keyword search across all public channels' posts.** See below. |
| `channels.checkSearchPostsFlood(query)` | Shows the search quota without spending it: `total_daily`, `remains`, `wait_till`, `stars_amount`, `query_is_free`. |
| `channels.getChannelRecommendations` | Similar channels (users only) [S]. |
| `contacts.search(q)` | Find public channels and groups by name. |

**Global post search, as it works now:**

- **Full-text search:** each account has a daily free quota. After that the server answers `FLOOD_WAIT_<seconds>_OR_STARS_<n>`: wait, or pay *n* Stars per search (seen in TDLib's code) [V].
- Paging through the **same query** is free ("repeated requests with the same query will be free") [V].
- Up to 100 results per page [V].
- **Hashtag and cashtag search** uses the same method without that payment path [V]; whether it has its own limits is [P0].
- The 31 July 2025 announcement says post search is "initially available to Telegram Premium users", and some API layers list `PREMIUM_ACCOUNT_REQUIRED` [S].
- It searches **posts in public channels**, not groups [V].
- The free quota and the Stars price are set by the server: [P0], read with the free `checkSearchPostsFlood`.

**Public groups:**

- Official apps let you preview most public groups without joining. Whether our account can read their history and replies that way is [P0].
- Joining would make the account **visible in the member list**, counts toward the 500/1000 limit, and may trigger the group's anti-spam or CAPTCHA bot. If a group needs that, **we stop there**; we never try to pass such checks automatically.
- Member lists (`channels.getParticipants`) are **blocked in our code** unless you approve them.

**What needs joining:**

- *Push* updates (new posts, deletions pushed to us). Without joining we poll, which works but costs requests.
- TDLib also notes that updates in channels are only received for "opened" chats.

**What needs admin rights** [V]:

- `stats.getBroadcastStats`: follower growth, views by source, languages, interactions.
- This is only possible on **client-owned channels**, if the client makes our account an admin with the statistics right.

**Errors to handle** [V/S]:

| Error | Meaning |
|---|---|
| `FLOOD_WAIT_X` | Wait X seconds; per method and account; can be hours. |
| `FLOOD_WAIT_X_OR_STARS_Y` | Post search quota used up. |
| `FLOOD_PREMIUM_WAIT_X` | Download speed limit for non-Premium accounts. |
| `FROZEN_METHOD_INVALID` | The account is **frozen**: read-only, with appeal only via @SpamBot [3P]. |
| `CHANNEL_PRIVATE`, `USERNAME_INVALID`, `MSG_ID_INVALID`, `CHANNELS_TOO_MUCH` | Lost access, bad username, bad message id, join limit reached. |

Telegram **publishes no numeric rate limits** [3P].

| | |
|---|---|
| **Ban risk** | **Medium.** Unofficial-client logins are "under observation". New accounts and VoIP numbers are riskier: gotd's guide says "Do not use VoIP numbers" and "use it passively (i.e. receive more than sending)" [3P]. Bursts, many username lookups, joins and sending raise the risk. |
| **ToS position** | API ToS (own api_id, no AI/ML, no view inflation, sponsored-message clause open) plus the scraping clause. |
| **Libraries** | **gotd/td (Go):** "stable", v0.163.0 published 22 Oct 2026, pluggable session storage, FLOOD_WAIT and rate-limit middleware in gotd/contrib [3P]. It fits a Go stack. **TDLib:** official and robust, C++ with JSON; the Go wrapper is "WIP". **Telethon:** GitHub repo archived 21 Feb 2026, moved to Codeberg. **Pyrogram:** archived 23 Dec 2024, "no longer maintained". → **Recommendation: gotd/td.** |

### 3.4 Official and third-party data services

- **Official:** Telegram offers **no data API** for channels you don't own. Channel statistics exist only for that channel's admins (above).
- **TGStat** (tgstat.com): a large index of public channels and chats with three APIs. Official price page via search, rubles, 5% VAT included [S]:
  - **Search API** (post search and mentions): START 2,100 ₽, S 4,200 ₽, M 12,600 ₽, L 47,250 ₽ per month.
  - **Stat API** (channel stats, mentions by period and by channel): free tier; S 2,100 ₽ (30 channels, 6,000 requests a month); M 8,190 ₽; L 20,370 ₽; XL 60,900 ₽.
  - **Callback API** (real-time keyword and channel alerts): START 8,400 ₽, S 29,820 ₽, M 72,240 ₽, L 149,100 ₽.
  - Unknowns: coverage of Iraqi and Kurdish channels; their terms on storing data; paying in rubles from Iraq. Scraping TGStat itself is forbidden by their user agreement [3P, Bellingcat toolkit].
- **Telemetr.io, social-listening suites** (Brand24, YouScan, Talkwalker, Meltwater, Brandwatch) list Telegram as a source. Prices and coverage are not verified (sites blocked here).
- **Scraping resellers** (Apify actors, ScrapeCreators' Telegram endpoints) mostly read the same web preview, or use *your* session. It is the same ToS problem, outsourced. I did not call any of them.
- **Assessment:** a licensed index (TGStat or similar) is the most realistic way to find mentions *beyond* the channels we watch. It needs a trial, legal review and a payment route.

### 3.5 Side-by-side

✅ = yes · ◐ = partly · ❌ = no · [P0] = to be measured

| Data | Logged out | Bot (owned chats only) | User account | Third-party |
|---|---|---|---|---|
| Channel title, username, description, photo | ✅ | ✅ | ✅ | ✅ |
| Subscriber count | ✅ (exact or rounded: [P0]) | ✅ | ✅ exact | ✅ with history |
| Linked discussion group | ◐ [P0] | ✅ | ✅ | ? |
| Post text, time, links | ✅ | ✅ (new posts only) | ✅ | ✅ |
| Media files | ◐ photos, small videos | ✅ ≤ 20 MB | ✅ (speed-limited without Premium) | ◐ |
| Polls | ◐ percentages [P0] | ✅ | ✅ (results maybe only after voting; we never vote) [P0] | ? |
| Forward source | ◐ name or link | ✅ | ✅ id and post | ◐ |
| Views | ◐ rounded | ❌ | ✅ exact | ✅ |
| Reactions | ✅ [P0 precision] | ✅ totals, delayed | ✅ | ◐ |
| Forward count / comment count | ❌ / ◐ [P0] | ❌ / ❌ | ✅ / ✅ | ◐ |
| Edits | ◐ flag only | ✅ | ✅ | ❌ |
| Deletions | ◐ by absence | ❌ | ◐ by re-check (✅ pushed if joined) | ❌ |
| History before we started | ✅ depth [P0] | ❌ | ✅ full | ✅ their archive |
| Comments under posts | ◐ widget [P0] | ✅ (bot in discussion group) | ✅ | ◐ |
| Public groups (messages, replies) | ❌ | ✅ only if admin or privacy off | ✅ without joining [P0], or join with approval | ◐ |
| Member lists | — not collected — | — not collected — | blocked in code | — |
| New-post delay | minutes (polling) | seconds | minutes (polling), seconds if joined | vendor-dependent |
| Search inside one channel | ✅ | ❌ | ✅ | ✅ |
| Search all public channels | ❌ | ❌ | ✅ (Premium, quota, Stars) | ✅ (paid) |
| Admin statistics | ❌ | ❌ | ✅ if made admin | ❌ |

---

## 4. Recommendation

**Answer to "logged out, bot, or user account, and for which data?"**

| Way | Use it for |
|---|---|
| **Logged out (default)** | Public channels we don't own: channel info and its changes over time, posts, rounded views, reactions, polls, forward sources (names), history backfill, search inside each watched channel. |
| **Bot** | Channels and groups **our clients own**, where they add it as admin: new posts within seconds, edits, reaction totals, every comment, group messages. Add nothing else to it. |
| **One user account** (you decide which; see §7.2) | Only what the others can't do: (1) public groups we don't own; (2) complete comment threads and exact comment counts; (3) exact views and forwards; (4) deletion checks; (5) keyword search across all public channels (needs Premium); (6) similar-channel discovery; (7) channels without a web preview; (8) admin statistics on client channels, if a client makes it admin. Read-only, slow and steady, **no joins** unless you approve a specific public group. |
| **Third-party (optional)** | Licensed mention discovery across channels we don't watch, if budget, legal review and payment allow. |

**Conditions:**

- Legal sign-off on §2 before collecting third-party channels.
- No ML on Telegram data until it is cleared.
- **Phase 0 decides the final split.** The design keeps the split as configuration per channel and per data type, so we can move a data type from "web" to "account" or the reverse without code changes.

**Why not "user account for everything"?**

- It puts all the load and all the ban risk on one account.
- If it is banned or frozen we lose everything at once.
- Logged-out collection spreads the load and keeps the system working when the account is paused.

**Why not "logged out only"?**

- No public groups.
- No reliable comments.
- No cross-channel search.
- Only rounded numbers.

---

## 5. What is realistic for each thing you asked for

| You asked for | Realistic? | How |
|---|---|---|
| Channel title, username, description, photo, subscriber count, linked group | **Yes** | Logged out (profile + feed header). Account for the exact count and `linked_chat_id`. Bot for owned channels. |
| How these change over time | **Yes** | Snapshot on a schedule (§8.5). Store a row only when something changes, plus regular subscriber-count points. |
| Post text, time, links | **Yes** | All three ways. |
| Media (photos, video, files, polls) | **Mostly** | Photos and small videos logged out. Big files only with the account or the bot. Poll results may need the user to vote, and we **never vote** [P0]. |
| Forwards and their original source | **Yes** | Logged out gives the source name and link. The account gives the source channel id and original post id. |
| Views | **Yes**, exact only with the account | Logged out: rounded. Bot: none. |
| Reactions | **Yes** | All three (bot: totals only, delayed). |
| Forward count, reply count | **Account only** (reply count possibly logged out [P0]) | |
| Edits | **Yes** | Flag logged out. Full versions with the account or the bot. |
| Deletions | **Partly** | Detected by re-checking (we know "deleted between check A and check B", not the exact time). Pushed in real time only if the account joins. |
| Comments under posts (discussion group) | **Yes with the account**; bot for owned channels; widget [P0] | §8.8 explains how we keep them complete. |
| Public groups: messages and replies, no member lists | **Account only** (or bot if the owner adds it) | Read without joining if possible [P0]; join only with your approval. Member lists are blocked in code. |
| New posts detected quickly | **Yes** | Bot: seconds. Others: polling every 2–60 minutes by priority (§8.5). |
| History backfill for each new channel | **Yes** | Logged out (depth [P0]) or account (full). Comments backfill limited to recent posts (you choose how far). |
| Mentions of our keywords across public channels | **Partly** | (a) **Complete** for every channel and group we watch (we match locally on everything we collect). (b) **Partial** across all other public channels via the account's post search (quota, Premium) and/or a paid index like TGStat. (c) **Never** in private chats or groups. Public groups are not covered by Telegram's global post search. |

---

## 6. Clarifying questions (all at once)

**Scope and size**

1. How many of each: (a) brand-owned channels, (b) competitor channels, (c) other public channels to watch, (d) public groups? What growth do you expect in 12 months?
2. The brand names and keywords: how many? Which languages and scripts: Arabic, Kurdish (Sorani and/or Kurmanji), English, transliterations ("Franco-Arabic"), common misspellings?
3. How fresh must data be?
   - New posts from brand channels: 1 minute? 5? 30?
   - New posts from competitor and other channels?
   - New keyword mentions (alert delay)?
4. How far back should backfill go for a new channel: all history, 12 months, 90 days? And for comments and group messages?
5. For how long after publishing should we keep re-checking a post's views, reactions and comments: 7, 30 or 90 days?

**Media**

6. Metadata only, thumbnails, full photos, or also videos and files? Any size cap? Which analysis service would receive media, and **does it use ML** (see §2.2)?

**Clients, accounts and money**

7. Monthly budget for this system (servers plus Telegram plus any vendor)?
8. Do clients own some of these channels or groups? Would they add our bot as admin (and to the discussion group)? Would any make our user account an admin so we can read official statistics?
9. A user account is needed for groups, complete comments and global search (§3.3, §7.2). **Which account?** It should be a company-owned SIM on a real carrier (no VoIP), used normally in the official app. Should it have **Premium** for global search?
10. Public groups: may the account **join** a public group when it can't be read otherwise? Yes or no per group, or never?
11. Comments: all comments on all watched posts, or only on brand-owned posts and matched mentions? Replies to comments too?

**Legal and privacy**

12. Who gives the legal opinion on Telegram's terms (§2)? Should I prepare a short brief for that lawyer? Would you like to ask Telegram for written permission?
13. Which data-protection laws apply: are any clients or monitored people in the EU or UK (GDPR)? Do you have a DPO or legal adviser?
14. Default retention periods (proposal in §8.12) and whether **author pseudonymization is on by default**. Who handles erasure requests?

**Infrastructure and outputs**

15. Should I reuse the Facebook project's patterns and infrastructure (same Postgres, Kafka/Redpanda, Valkey, S3, monitoring)? If yes, **please attach `Abdullah95k/securebrand-microservices`** so I can check what fits Telegram. Same cluster, or separate?
16. Where does the server run (Hetzner? your own?), what size, and what monitoring stack do you use?
17. Who consumes the data: a dashboard, alerts (by email, Slack or Telegram), an API for the SecureBrand platform?
18. Are you open to paying a licensed index (TGStat or similar) for wider mention discovery, and can you pay in rubles?
19. Is GitHub Actions fine for CI?

---

## 7. What you need to provide, and the monthly cost

### 7.1 Checklist

| Item | Needed for | Notes |
|---|---|---|
| Lists of channels, groups and keywords | Everything | With the reason for each (brand, competitor, watch) and the client it belongs to. **Usernames or public `t.me/<name>` links only; invite links (`t.me/+…`, `joinchat`) are rejected.** |
| Legal opinion on §2 | Before collecting third-party channels | |
| A Linux server | Everything | See §7.3. |
| Storage | Posts, metrics, media | Postgres volume plus S3-compatible bucket (§7.3). |
| **Bot token** (from @BotFather) | Client-owned channels | Only if a client agrees to add the bot. Stored on the server only (§8.13). |
| **One user account:** real phone number on a company-owned SIM; Telegram account in the official app; 2FA password; `api_id`/`api_hash` from my.telegram.org with that number | Groups, comments, global search, exact counts | You decide which account. Optional Premium. |
| Proxies | **None** | We do not rotate IPs to dodge limits. That would be circumventing Telegram's protections. One stable server IP is better for the account too. |
| Optional: TGStat (or similar) subscription | Wider mention discovery | After a trial and legal check. |

### 7.2 Why a user account is needed, and the rules for it

**Why:**

- Public groups we don't own can't be read any other way.
- Complete comments and exact counts need it.
- Cross-channel keyword search is account-only.

**Rules:**

- It must be a **real, company-controlled number**: never bought, borrowed or virtual (VoIP).
- Use the account normally in the official app for a while before connecting our software.
- Turn on 2FA and a recovery email.
- Don't use a staff member's personal account: a ban would take their private chats with it.
- Register **our own** api_id for that number. The app title must not contain "Telegram".
- Log in from **one** server only and keep it stable.
- If it gets frozen we **stop and tell you**; we never create a replacement account on our own.

### 7.3 Estimated monthly cost

Vendor pages are blocked here. These prices come from search summaries of Hetzner's and TGStat's price pages and third-party reviews [S/3P]. Please check them live.

Hetzner raised prices on 1 April and 15 June 2026, and new-server availability has been limited since late June 2026 [3P].

| Item | Small scale (≈200 channels, 20 groups) | Notes |
|---|---|---|
| Server: Hetzner CX43 (8 vCPU, 16 GB, 160 GB) | ≈ €16.5 | Enough for a standalone stack (Postgres, Redpanda, Valkey, services). Dedicated CCX23 (4 vCPU, 16 GB) ≈ €86 if CX is unavailable. |
| Server backups (+20%) | ≈ €3.3 | |
| Extra volume for Postgres (100 GB) | ≈ €5–6 | |
| Object storage (S3) | €6.49 | Includes 1 TB storage and 1 TB egress. |
| Telegram Premium for the account | ≈ US$5 | US App Store price $4.99/month [3P]; varies by country. Only if global search is wanted. |
| Telegram Stars for searches beyond the free quota | US$0–20 | 100 Stars ≈ US$2 [3P]. Real need measured in Phase 0. |
| SIM line | your carrier's price | Keep it active: losing the number can mean losing the account. |
| **Total, standalone** | **≈ €32 + US$5–25 → roughly US$40–65/month** | With CCX23 instead: ≈ US$115–145. |
| **Total, reusing the Facebook project's cluster** | **≈ €10–15 + US$5–25** | Mainly extra storage. |
| Optional TGStat | from 2,100 ₽/month (Stat S) to 12,600 ₽ (Search M); real-time Callback from 8,400 ₽ | Convert at the current rate. |

**Storage estimate**

Assumptions for the small scale: 200 channels × 8 posts a day; 20% of posts get on average 30 comments; 20 groups × 300 messages a day.

| Data | Per month | First year |
|---|---|---|
| Posts | ≈ 48k rows | |
| Metric snapshots | ≈ 0.6M rows | |
| Comments | ≈ 290k | |
| Group messages | ≈ 180k | |
| Postgres total | | ≈ **10–15 GB** including indexes and a 3,000-post backfill per channel. At 1,000 channels: ≈ 50–75 GB. |
| Photos (90-day retention) | | ≈ **20 GB** steady. |
| Videos (if downloaded fully) | can reach 100s of GB | Proposal: metadata and thumbnail by default. Full video only for brand-owned posts and matched mentions, with a size cap. |

---

## 8. Architecture

### 8.1 Principles

- **Read-only by construction.** The account and bot code has an **allowlist of permitted methods**. Sending, joining, voting, reacting, marking read, view increments, invite-link methods (`checkChatInvite`, `importChatInvite`) and member-list methods are blocked in code. `joinChannel` is only allowed for public chats you approved in the registry.
- **One adapter per access way**: `web`, `bot`, `mtproto`, later `vendor`. All produce the same normalized records. A **collection plan** per source says which adapter handles which data type.
- **Every request goes through the rate limiter.** No code path can call Telegram without passing it.
- **Minimal data:** collect only fields we use, no raw-response archives, no member lists.
- **Privacy at the boundary:** the suppression list and pseudonymization are applied **before** anything is written or published.
- **Telegram timestamps are UTC.** We store UTC; reports and alerts show Iraq time.

### 8.2 Components

```
                 ┌──────────────── registry (sources, keywords, clients, approvals, suppression) ──────┐
                 │                                                                                    │
            ┌────▼─────┐   due tasks    ┌───────────────┐
            │scheduler │──────────────▶ │  rate limiter │  (Valkey: budgets, pauses, circuit state)
            └────┬─────┘                └──────┬────────┘
                 │ jobs (Kafka topics or Postgres queue)
     ┌───────────┼───────────────┬───────────────────┐
┌────▼────┐ ┌────▼─────┐ ┌───────▼───────┐ ┌─────────▼─────────┐
│web      │ │mtproto   │ │bot receiver   │ │vendor (optional)  │
│collector│ │worker    │ │(webhook)      │ │TGStat etc.        │
│(t.me)   │ │(1 per    │ │               │ │                   │
│         │ │ account) │ │               │ │                   │
└────┬────┘ └────┬─────┘ └───────┬───────┘ └─────────┬─────────┘
     └───────────┴───── normalized records ───────────┘
                         │
                 ┌───────▼────────┐   suppression check, pseudonymization, dedupe, versions
                 │ normalizer /   │──────────────▶ Postgres (sources, posts, versions, metrics,
                 │ ingest         │                comments, group messages, mentions, audit)
                 └───┬───────┬────┘
                     │       └──────▶ media fetcher ──▶ S3 (with size caps, retention)
                     ▼
              mentions matcher ──▶ alerts / API / dashboard
                     ▲
     privacy service: retention jobs, erasure, suppression list, audit
```

### 8.3 Services (Go)

| Service | Job |
|---|---|
| `tg-registry` | API and CLI for sources, keywords, clients, approvals (for example "may join group X"), and the suppression list. Rejects invite links and private chats. |
| `tg-scheduler` | Computes what is due: profile checks, new-post polls, metric refreshes, comment syncs, deletion checks, backfill steps, searches. Writes jobs. |
| `tg-web` | Logged-out collector and HTML parser. Selectors live in one versioned file, as TeleGlance does. |
| `tg-mtproto` | **One process per account.** Owns the encrypted session; gotd/td with our limiter and method allowlist. Never runs twice at once for the same account. |
| `tg-bot` | Bot API webhook (or long polling) receiver for owned chats. |
| `tg-ingest` | Normalizes, checks the suppression list, pseudonymizes, dedupes on (chat id, message id), stores versions and metrics, emits events. |
| `tg-media` | Downloads allowed media within size and type caps to S3. Hands off to analysis only if that is legally cleared. |
| `tg-mentions` | Local keyword matching on everything ingested; runs scheduled global searches; puts discovered unknown channels in a **review queue** (never auto-adds). |
| `tg-privacy` | Retention deletes, erasure requests, suppression list, audit log, propagation of erasures to downstream systems. |

**Shared:** Postgres, Valkey, S3, and Redpanda (Kafka) if we reuse the Facebook stack. At small scale a Postgres job queue (`FOR UPDATE SKIP LOCKED`) would also do. Question 15 decides this.

### 8.4 Data model (main tables)

| Table | Contents |
|---|---|
| `sources` | Telegram peer id, kind (channel, group), current username, reason (brand, competitor, watch), client, priority tier, collection plan per data type, approvals, status. |
| `peer_cache` | `access_hash` per account and peer (not secret, but bound to that account), when resolved. Means **resolve once**. |
| `source_snapshots` | Title, username, description, photo hash, linked group, subscriber count, captured time. A new row **only when something changed**, plus periodic count points. |
| `posts` | Chat id, message id, date, text and entities, media summary, forward source (peer id, name, original post id, original author, date), reply-to, album id, signature (**personal data**: pseudonymizable), edit date, first and last seen, content hash, `deleted_detected_at`, `deleted_window` (between which checks). |
| `post_versions` | Previous text and media of edited posts. |
| `post_metrics` | Append-only: views, forwards, replies, reactions (JSON), captured time, source adapter, and an `exact` flag (web = rounded). |
| `comments` and `group_messages` | Chat id, message id, thread root (post), reply-to, `author_key`, date, text, edit date, deletion info. |
| `authors` | `author_key` (keyed hash). Display name and username only when pseudonymization is off, encrypted. |
| `media` | Post or comment reference, type, size, mime type, S3 key, sha256, status, expiry. |
| `keywords` and `mentions` | Keyword variants and normalization rules; matches with location, how found (local, channel search, global search, vendor). |
| `erasure_requests`, `suppression_list`, `audit_log` | Privacy (§8.12). |

### 8.5 Scheduling: how often things are re-checked

All numbers are **starting points for Phase 0 to adjust**.

- **New-post detection, by tier:**
  - Owned channels with the bot: push, seconds.
  - **Hot** (brand-owned without bot, main competitors): every **2–5 min**.
  - **Normal:** every **15–20 min**.
  - **Cold:** every **60 min**.
  - Each channel's interval adapts to how often it posts (fast posters checked more often, quiet ones less), within the tier's limits.
  - Night slow-down **01:00–07:00 Iraq time** (intervals × 2), unless the channel posts at night.
- **One logged-out request does two jobs.** A feed page returns the newest ~20 posts *with* their views and reactions. The "new post" poll therefore also refreshes metrics of recent posts for free.
- **Metric refresh by post age** (views, reactions, forwards, comment count): at **15 min, 1 h, 3 h, 6 h, 12 h, 24 h, 2 d, 3 d, 7 d, 14 d, 30 d**, then stop. Brand-owned posts continue weekly up to 90 days.
  - Posts due together are refreshed together: one feed page (~20 posts) or one `getMessagesViews` call (many ids).
- **Channel profile:**
  - Brand and competitor channels: 4× a day at about **00:00, 06:00, 12:00 and 18:00 Iraq time** (± random jitter of a few minutes).
  - Others: once a day at about **04:00 Iraq time**.
- **Comments:** see §8.8. **Deletions:** see §8.9.
- **Backfill of a new channel:**
  1. Resolve once and read the profile.
  2. Walk history from newest to oldest down to the depth you choose, using at most **30% of the budget**, at low priority.
  3. Pause on any limit and resume from a saved cursor.
  4. Then comments for the last *N* days of posts.
- **Global keyword search:** spread evenly over the day within the daily free quota (read via `checkSearchPostsFlood`). Paid searches only within a monthly Stars budget you set.

**Example request budget**

Small scale, logged out: 50 hot channels at 5 min, 100 normal at 20 min, 50 cold at 60 min, plus deeper refreshes, profiles and comment widgets.

| Task | Requests per day |
|---|---|
| Hot channels (50, every 5 min) | 14,400 |
| Normal channels (100, every 20 min) | 7,200 |
| Cold channels (50, every 60 min) | 1,200 |
| Deeper metric refreshes | ≈ 1,000 |
| Profiles | ≈ 200 |
| Comment widgets | ≈ 1,000 |
| **Total** | **≈ 25,000 ≈ one request every 3.5 s** |

The account would do only its special jobs: comments, groups, search, exact counts, deletion checks. Target **≤ 15,000 calls a day**, never more than about 1 per second.

### 8.6 Rate limiting that respects FloodWait

- **Budgets are kept per credential** (each account, each bot, the server's IP for t.me) **and per method or endpoint**, as token buckets in Valkey. Examples:
  - `resolveUsername`: a few per hour, hard daily cap.
  - `getHistory` and `getReplies`: shared steady rate.
  - `searchPosts`: daily quota.
- **FLOOD_WAIT_X:**
  1. Pause that method on that account for **X seconds plus 10–20% random extra**, and never retry earlier.
  2. Pause the **whole account** if X > 5 min.
  3. Reduce that method's budget by half (AIMD: then raise it slowly by about 5% per quiet hour).
  4. Record X in metrics.
  5. If X > 1 h, open the **circuit** for that account, alert you, and move eligible work to the logged-out adapter.

  gotd/contrib has a FLOOD_WAIT middleware ("Default is to wait at most a minute", up to 5 retries) [3P]. We use our own wrapper so long waits are **parked**, not slept inside a request, and are visible across restarts.
- **`FLOOD_WAIT_X_OR_STARS_Y`** (search): treat as "quota used up"; pay only if the job explicitly allows it and the Stars budget has room.
- **`FLOOD_PREMIUM_WAIT_X`:** slow media downloads only.
- **`FROZEN_METHOD_INVALID`:** stop **all** account work immediately, alert, wait for you.
- **Logged out:**
  - Honour HTTP 429 and `Retry-After` exactly.
  - Back off exponentially on 5xx.
  - A circuit breaker per host.
  - If we ever get a challenge page or CAPTCHA, **we stop and alert**; we never try to get past it.
- **Phase 0 sets the steady-state budgets at about half** of the highest rate that showed no limits.

### 8.7 Retries and failure handling

- Jobs are idempotent: writes are upserts on (chat id, message id). Delivery is at-least-once.
- Retries use exponential backoff with jitter (1 s → 2 s → 4 s … up to 15 min), with a maximum number of attempts per job type. After that the job goes to a **dead-letter queue** with a reason, and a dashboard shows the count.
- Telegram-side limits are **not** retried by the generic retry loop. They go through §8.6.
- **Parser failures** (unknown HTML) are counted per selector. Above 5% failures, alert "markup changed" and fall back to the account for affected data if allowed.
- The account process keeps its updates state and backfill cursors in Postgres, so a restart continues where it stopped.

### 8.8 Keeping comments complete

- **Target number:** the post's comment counter (`replies.replies` from the account, or the counter from the web preview if Phase 0 finds one). This counts all replies in the thread, including replies to replies.
- **Incremental fetch:** when a refresh shows the counter went up, fetch only newer comments: `getReplies(…, min_id = last seen comment id)`, paging until done.
- **Full reconciliation** of the whole thread at **1 h, 24 h and 7 d** after the post (and 30 d for brand-owned posts):
  - ids that disappeared → deleted;
  - changed `edit_date` → new version.
- **Completeness metric** = comments we have (not deleted) ÷ counter. If it is below **98%** after reconciliation, re-sync that thread and alert if it stays low.
- **Owned channels:** the bot in the discussion group receives every comment live. The account (if available) still runs the reconciliation, because bots never see deletions.
- **Logged-out widget:** used only if Phase 0 shows it returns complete threads. Otherwise only as a "something new" signal.

### 8.9 Edits and deletions

- **Edits:** detected by `edit_date` or a changed content hash (account or bot), or by the "edited" marker and a changed hash (web). We store the old version in `post_versions`.
- **Deletions without joining:**
  - re-check known ids with `channels.getMessages` in batches at **1 h, 24 h, 7 d, 30 d**: an empty result means deleted;
  - logged out: the id is missing from the feed between neighbours, confirmed by the single-post page.
  - We record **the window** ("between 10:00 and 11:00 Iraq time"), not a false exact time.
- **What we keep after a deletion** (your choice, question 14). Proposal:
  - keep **channel posts** with a deleted flag (useful for competitor monitoring);
  - for **comments and group messages** by people, remove the text within 30 days and keep only counts.

### 8.10 Media

- Defaults: metadata always; photos and thumbnails; videos and files only for brand-owned posts and matched mentions, capped at 20 MB (configurable).
- Store a sha256 to avoid duplicates. S3 objects are deleted on retention expiry or erasure.
- **No hand-off to the analysis service** for Telegram data until §2.2 is resolved.

### 8.11 Mentions

1. **Local matching (complete for watched chats):**
   - Runs on all posts, comments, group messages, captions, link previews, poll text and file names we collect, plus @mentions of brand usernames, links to brand domains, and **forwards from brand channels** (shares).
   - Text normalization for Arabic and Kurdish: remove diacritics and tatweel; unify alef forms (أ إ آ → ا) and ى/ي; optionally ة/ه.
   - Script-aware handling of Kurdish letters (ک ی ە ۆ ێ) so we don't break words.
   - Latin case folding; per-keyword variant lists and exclusions you maintain.
2. **Per-channel search** to backfill mentions in a newly added channel without downloading its whole history: `?q=` logged out, or `messages.search` with the account.
3. **Global discovery:**
   - the account's `channels.searchPosts` for each keyword, within quota (Premium);
   - hashtag search for brand hashtags;
   - "similar channels" for competitors.
   - Unknown channels found this way go to a **review queue**; you decide whether to watch them.
4. **Optional vendor** (TGStat Search or Callback) for wider coverage.

### 8.12 Personal data: retention, pseudonymization, erasure, suppression

**Data classes and proposed default retention** (you decide):

| Class | Examples | Default |
|---|---|---|
| Channel metadata | title, subscribers | while watched + 12 months |
| Channel posts | text, metrics | 24 months; metric snapshots thinned to daily after 90 days |
| People's content | comments, group messages | **12 months** |
| Author identities | user id, name, username | tied to their content; pseudonymized by default |
| Media | photos, video | **90 days** |
| Raw responses | — | **not stored.** Debug captures only on the server, encrypted, deleted after 72 h, never in git |

**Pseudonymization (per client switch):**

- `author_key = HMAC-SHA256(secret_key, "tg:user:" + user_id)`, with a separate key per client so the same person can't be linked across clients.
- With pseudonymization on, names and usernames are **not stored**. We also keep `HMAC(username)` at collection time, so an erasure request by username can still find the rows.
- Signed post authors (`post_author`) are handled the same way.
- Public channels themselves are organizations, not people, and are not pseudonymized.

**Erasure** (CLI and API; every request audited):

- **By person** (user id or @username):
  1. Add the person to the **suppression list first**, so nothing new arrives during deletion.
  2. Delete their comments, messages, signatures, mentions and media rows, plus the S3 objects.
  3. Purge caches.
  4. Publish an erasure event so downstream systems (analysis service, search indexes, Kafka compacted topics via tombstones) delete too.
  5. Record completion.
- **By channel or group:** the same, for everything from that source, and the scheduler stops watching it.
- **Backups** expire within the retention window. After any restore, the **erasure log is replayed** before the system serves data.

**Suppression list:**

- Entries are hashed person ids and usernames, channel ids and usernames, with scope (one client or all) and a reason code.
- Checked by the registry (can't add a suppressed source), the scheduler, ingest (drops suppressed authors' content before writing), the media fetcher and the mentions output.
- Postgres is the source of truth, with a fast copy in Valkey.
- **Erased data never comes back**, even if it is collected again.

**Other safeguards:**

- Logs never contain message text, names, usernames or phone numbers. Only ids and hashes.
- Member lists are never collected (blocked method).

### 8.13 Credentials and the session file

- **Locations:**
  - All secrets live in `/etc/securebrand-tg/` on the server, readable only by the service user (`0600`), or in environment variables set by systemd or Docker secrets.
  - **Never** in the repo, in logs, in command-line arguments (they show in `ps` and shell history) or in chat.
- **Session file:**
  - Encrypted at rest (AES-GCM) with a key kept in a separate file.
  - Lives on one server only.
  - We list the account's active sessions (`account.getAuthorizations`) daily and alert on any unknown login.
- **Login is interactive:** you run the login command on your server and type the phone code and 2FA password yourself. I never see them.
- **Redaction:** the Bot API puts the token in the URL path, so our HTTP client strips it from every log line and error message.
- **Repo safety:** `.gitignore` excludes `*.session`, `.env*`, `secrets/`, Phase 0 output. CI runs a secret scanner (gitleaks plus patterns for bot tokens and 32-hex api_hash values).

### 8.14 Testing and the "fake Telegram"

- **Fake t.me:** an HTTP test server serving **synthetic** pages built from templates with the same structure as the real preview (fictional channels, generated text in Arabic, Kurdish and English, fake numbers). Switches simulate 429 with `Retry-After`, 5xx, redirect (no preview), and markup drift. **No captured real pages are ever committed.**
- **Fake MTProto:** our adapter interface has an in-memory fake that can script `FLOOD_WAIT`, `FROZEN_METHOD_INVALID`, empty messages (deletions), edits, and comment threads. The gotd layer is tested with gotd's mock invoker (`tgmock`).
- **Fake Bot API:** test server for webhook, `getUpdates` and `getChat`.
- **Integration tests:** Postgres, Valkey, Redpanda and MinIO in containers (testcontainers-go) in CI.
- **Test coverage:**
  - unit tests: limiter (fake clock), parser, normalizer, Arabic and Kurdish normalization, privacy rules;
  - integration tests: full flows against the fakes, including erasure plus suppression end-to-end ("erased data never comes back").
- **Fixture rule:** fixtures are generated by a script from a fixed seed. A CI check fails if `testdata/` contains anything not produced by the generator.

### 8.15 Monitoring and alerts

**Metrics (Prometheus):**

- requests by adapter, method and result;
- FloodWait count and seconds by method;
- 429s;
- remaining budgets and search quota;
- account state (ok, paused, frozen);
- queue lag;
- **freshness** (post time → first seen, p50 and p95 per tier);
- refresh schedule adherence;
- **comment completeness**;
- parser failure rate;
- dead-letter count;
- retention and erasure job status;
- disk usage.

**Alerts** (email or Slack, times shown in Iraq time):

- account frozen or paused > 1 h;
- parser failures > 5%;
- freshness target missed;
- completeness < 98%;
- unknown login on the account;
- erasure overdue;
- disk > 80%.

Grafana dashboards. We reuse the Facebook project's stack if it fits.

### 8.16 What to reuse from the Facebook project (needs the repo attached)

To check, not assume:

- the adaptive rate limiter and circuit breakers (Telegram's limits are per *account and method* with explicit waits; Facebook's are different);
- the Phase 0 kit structure;
- the fake-platform testing approach;
- GDPR erasure (extend it with the suppression list);
- Kafka topic conventions;
- the shared Postgres, S3 and monitoring.

The **analysis-service handoff** needs the legal check in §2.2 before reuse.

---

## 9. Phase 0: measure before building

**Goal:** real numbers for the chosen mix, on a small sample, over **7 days**, with strict caps.

**Sample:**

- about **10 channels**: up to 3 brand-owned (ideally with the bot), 4 competitors, 3 other public channels;
- **2 public groups**;
- **3 keywords**.

While the legal opinion is pending, I suggest starting with **brand-owned channels only**.

**Tool:**

- A small read-only Go command, `tgprobe`, built in the first pull requests after you approve this plan.
- It has hard caps on requests per test and overall.
- It writes **only aggregated numbers**: counts, timings, error codes, which fields exist, whether numbers are rounded. It writes **no message text, no names**.
- Output goes to `/var/lib/securebrand-tg/phase0/` on your server, never to git.
- With the account it **never probes for the ceiling**: it runs at a fixed gentle rate and records what happens.

**Tests (each one runs only after you approve it):**

| # | Test | Measures |
|---|---|---|
| W0 | Read `https://t.me/robots.txt` (you can do this now, see commands below) | Telegram's own crawler policy |
| W1 | Profile and feed header for each sample channel | Fields present; subscriber count exact or rounded |
| W2 | Feed pages for 2 channels, back through history at 1 request / 5 s | Posts per page, how far back it goes, fields (views rounding, reactions, comment counter, edited, forward source, media sizes) |
| W3 | Polling tolerance: 1 request every 10 s → 5 s → 3 s → 2 s, 1 hour each, **stop at the first 429 or challenge** | Safe logged-out rate from one IP |
| W4 | Discussion widget on 20 posts with comments | Comments returned vs the account's count |
| W5 | In-channel search `?q=` for the 3 keywords | Works or not; results vs local matching |
| W6 | Feed URL for the 2 public groups | Expected: no preview |
| W7 | Single-post pages for posts that vanished from feeds | Confirms deletion detection |
| M0 | You set up the account, api_id and 2FA, and log in on the server | — |
| M1 | Resolve the sample usernames once, **spaced 5 minutes apart**; cache | Any limits on resolving |
| M2 | Full channel info; history backfill of 1 channel at 1 call / 3 s | Throughput, any FloodWait |
| M3 | Comment threads of 50 posts | Time, completeness vs counters |
| M4 | 7 days of refresh at the target budget (≈ 0.15–0.3 calls/s) | FloodWait frequency per method |
| M5 | The 2 public groups **without joining** | Readable or not. If not, we **don't** join; we report back. |
| M6 | `checkSearchPostsFlood` (free); then, only if you approve Premium, the 3 keywords × a few pages | Daily free quota, Stars price, usefulness vs local matches |
| M7 | Re-check stored post ids | Deletion detection rate |
| M8 | Exact views (account) vs rounded views (web) | How much rounding loses |
| B1 | Bot added by a client as admin to one channel and its discussion group, for 7 days | Posts, edits, reaction-count delay, comments vs the account's data |

**How the results decide the design:**

- **Logged-out channel collection:** use it if W3 shows a comfortable rate with no 429 and W1/W2 fields are enough. Otherwise more goes to the account.
- **Comment widget:** use it for comments if W4 reaches ≥ 98% completeness. Otherwise comments come from the account.
- **Account budget:**
  - If M4 shows more than one FloodWait over 10 minutes per day, halve the budget and re-run.
  - If the account is frozen, stop and rethink with you.
- **Search:** if M6's free quota is too small for our keywords, decide between a Stars budget and a vendor.
- **Settings:** the final budgets and intervals are set at about half the comfortable measured rates.

**Phase 0 report:** `docs/phase0-report.md`, aggregated numbers only. Plus an updated version of this plan.

**Timeline:**

| Days | Work |
|---|---|
| 1–3 | Small PRs that build `tgprobe` and the fakes |
| 4 | You set up the server (commands below) |
| 5–11 | 7-day run. A daily summary at 09:00 Iraq time. |
| 12 | Report |

### Commands you can run now (they don't touch Telegram except W0)

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

Test W0: read Telegram's crawler policy. It contains no personal data; send me what it prints:

```bash
curl -sS https://t.me/robots.txt
```

### Commands for later (once the `tgprobe` PRs are merged; exact flags may change)

Create the session encryption key (it is written straight to a file, never shown):

```bash
sudo -u securebrand-tg sh -c 'umask 077; openssl rand -base64 32 > /etc/securebrand-tg/session.key'
```

Write the account settings with an editor. Type `TG_API_ID=…` and `TG_API_HASH=…` yourself; never paste them anywhere else:

```bash
sudo -u securebrand-tg nano /etc/securebrand-tg/telegram.env
```

Log in interactively (you type the phone number, the code and the 2FA password):

```bash
sudo -u securebrand-tg /opt/securebrand-tg/bin/tgprobe login --env /etc/securebrand-tg/telegram.env
```

Run one approved test, for example W1:

```bash
sudo -u securebrand-tg /opt/securebrand-tg/bin/tgprobe web profile --sample /etc/securebrand-tg/phase0-sample.yaml --out /var/lib/securebrand-tg/phase0
```

---

## 10. Main risks

| # | Risk | Effect | What we do |
|---|---|---|---|
| R1 | **Telegram's terms** (scraping ban since Feb 2026; AI/ML ban) | The product may be non-compliant for third-party channels; ML analysis is forbidden | Legal opinion first; start with owned channels; minimal data; no ML; consider licensed data or written permission from Telegram |
| R2 | **Account ban or freeze** | Lose comments, groups and search | Passive, slow, read-only, cached lookups, one stable server, aged real account with 2FA; degrade to logged-out; never replace accounts without you |
| R3 | **IP throttling or blocking of t.me** | Slower freshness | Measured budgets, honour 429, no proxy rotation, alerts |
| R4 | **HTML changes** in the web preview | Parser breaks | Selectors in one file, failure-rate alarm, account fallback |
| R5 | **No web preview** for some channels (political ones may be affected) | Need the account for them | Detect redirect; route to account |
| R6 | **Search changes:** Premium, quotas and Stars prices change | Mention discovery cost or coverage changes | Read the quota daily; budget cap; vendor alternative |
| R7 | **API changes** (new layers; gotd lagging) | Breakage | Keep gotd updated; contract tests against fakes; watch the changelog |
| R8 | **Personal data and law** (GDPR for EU/UK people or clients; local law) | Fines, complaints | Retention, pseudonymization, erasure plus suppression, DPIA with your adviser |
| R9 | **Session file theft** = account takeover | Account abuse | Encryption, `0600`, single host, 2FA, daily active-session check |
| R10 | **Losing the phone number** (SIM expires, number recycled) | Account lost | Keep the SIM active; recovery email; 2FA |
| R11 | **Incomplete data** (deleted comments, deletion timing) | Wrong numbers in reports | Reconciliation and completeness metrics; tell clients the limits |
| R12 | **Location mismatch** (Iraqi number, server in Germany or Finland) | May add suspicion | Log in once from the server and keep it stable; Phase 0 watches |
| R13 | **Vendor issues** (TGStat ruble payment, sanctions, their own ToS position) | No vendor option | Trial plus legal and payment check before relying on it |

## 11. Open questions (not answerable from here)

1. Exact current wording of the Content Licensing terms and Bot Developer Terms 4.3. Both pages were blocked; I have search summaries only.
2. Does the sponsored-messages clause (API ToS 3.3) apply to a back-end collector whose output is shown in SecureBrand dashboards? (Legal.)
3. Why some channels have no web preview, and whether "restrict saving content" channels show one. [P0]
4. How many comments the discussion widget returns. [P0]
5. Whether non-Premium accounts can use `channels.searchPosts` at all; the free quota and Stars price. [P0, via the free check]
6. Whether public groups can be read without joining. [P0]
7. Real FloodWait thresholds for our access pattern. [P0]
8. Whether poll results are visible without voting (we will never vote). [P0]
9. Whether the analysis service uses ML. (You / Facebook-project repo.)
10. **Communities** (Telegram feature from July 2026; channels now carry `linked_community_id` [V]): may help discovery later. Out of scope for now.

## 12. Implementation roadmap (after you approve)

Each item is a small **draft pull request** with unit and integration tests and green CI. I review each one, and merge after CI is green, as you asked.

**Before Phase 0:**

| PR | Content |
|---|---|
| 1 | Repo skeleton: Go module, Makefile, CI (lint, tests, gitleaks, fixture check), `.gitignore`, contribution notes. |
| 2 | Rate limiter: per-credential and per-method budgets, FloodWait parking, AIMD, circuit breaker; fake-clock tests. |
| 3 | Web-preview parser, synthetic fixture generator, fake t.me server. |
| 4 | MTProto adapter on gotd: method allowlist, encrypted session storage, interactive login, fake and tgmock tests. |
| 5 | Bot API receiver and fake Bot API. |
| 6 | `tgprobe` Phase 0 command with hard caps and aggregated reports; integration tests against the fakes. |

→ **Phase 0 run and report**, then a plan update.

**Phase 1** (re-planned after Phase 0):

- database schema and migrations;
- registry with suppression list;
- scheduler;
- collectors;
- ingest and versions;
- comments sync;
- mentions;
- privacy service (retention, erasure, audit);
- media;
- monitoring;
- deployment (Docker Compose or systemd), with your server commands in each PR.

## 13. Sources

**Official Telegram sources (read via GitHub):**

- Telegram Desktop API schema, layer 232: <https://github.com/telegramdesktop/tdesktop/blob/dev/Telegram/SourceFiles/mtproto/scheme/api.tl>
- TDLib API with documentation: <https://github.com/tdlib/td/blob/master/td/generate/scheme/td_api.tl>
- TDLib post-search code: <https://github.com/tdlib/td/blob/master/td/telegram/MessageQueryManager.cpp>
- Bot API 10.3 specification (generated from the official docs): <https://github.com/PaulSonOfLars/telegram-bot-api-spec>

**Archived official legal texts (Open Terms Archive):**

- Terms of Service: <https://github.com/OpenTermsArchive/contrib-versions/blob/main/Telegram/Terms%20of%20Service.md>
- API Terms of Service: <https://github.com/OpenTermsArchive/contrib-versions/blob/main/Telegram/Developer%20Terms.md>
- Privacy Policy: <https://github.com/OpenTermsArchive/contrib-versions/blob/main/Telegram/Privacy%20Policy.md>
- Memo on the 3 Feb 2026 change: <https://github.com/OpenTermsArchive/opentermsarchive.org/blob/main/content/memos/telegram-prohibits-collecting-data-for-ai-use/index.md>

**Official pages seen through search (please re-check live):**

- <https://telegram.org/tos/content-licensing>
- <https://telegram.org/tos/bot-developers>
- <https://core.telegram.org/api/obtaining_api_id>
- <https://core.telegram.org/bots/features>
- <https://core.telegram.org/bots/faq>
- <https://core.telegram.org/widgets/discussion>
- <https://core.telegram.org/method/channels.searchPosts>
- <https://core.telegram.org/type/SearchPostsFlood>
- <https://core.telegram.org/method/channels.getChannelRecommendations>
- <https://core.telegram.org/api/premium>
- <https://telegram.org/blog/post-search-story-albums-and-more>

**Web-preview parsers (code read for what the preview contains):**

- snscrape: <https://github.com/JustAnotherArchivist/snscrape/blob/master/snscrape/modules/telegram.py>
- RSSHub: <https://github.com/DIYgod/RSSHub/tree/master/lib/routes/telegram>
- minet: <https://github.com/medialab/minet/blob/master/minet/telegram/scraper.py>
- TeleGlance: <https://github.com/NWGKGIT/TeleGlance>
- BroadcastChannel: <https://github.com/miantiao-me/BroadcastChannel>

**Libraries:**

- gotd/td: <https://github.com/gotd/td> (and its "how not to get banned" guide: <https://github.com/gotd/td/blob/main/.github/SUPPORT.md>)
- gotd/contrib: <https://github.com/gotd/contrib>
- Telethon: <https://github.com/LonamiWebs/Telethon>
- Pyrogram: <https://github.com/pyrogram/pyrogram>
- TDLib: <https://github.com/tdlib/td>
- go-tdlib: <https://github.com/zelenin/go-tdlib>

**Prices and vendors (search summaries):**

- TGStat prices: <https://tgstat.com/p/prices>
- Hetzner 2026 prices: <https://costgoat.com/pricing/hetzner>, <https://www.bitdoze.com/hetzner-cloud-cost-optimized-plans/>
- Telegram Premium review: <https://tgsuperstars.com/en/blog/telegram-premium-features/>
- Bellingcat toolkit on TGStat: <https://github.com/bellingcat/toolkit/blob/main/gitbook/tools/tgstat/README.md>

**Other:**

- Telegram Communities (July 2026): <https://www.republicworld.com/tech/telegram-rolls-out-communities-rich-text-editor-and-private-bot-messages-in-major-update-2026-07-20-132913>
