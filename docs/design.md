# Booru Bookmarker — Design

> Status: Draft (high-level). This doc will grow more detailed over time.

## 1. Overview

Booru Bookmarker is a web app for creating, organizing, and saving favorite search queries for booru sites (Gelbooru, Danbooru, e621, etc.).

**Platform:** a responsive web app that works on both desktop and mobile browsers. There are no native apps; installing it as a progressive web app (PWA) could be added later.

Its main feature is building a query in the UI for one site and **converting it to work on another site**. Conversion handles two things:

1. **Syntax translation**: rewriting the query in the target site's search syntax.
2. **Tag mapping**: replacing each tag with its equivalent on the target site.

## 2. Supported Sites

| Phase   | Sites                                   |
|---------|-----------------------------------------|
| Initial | Rule34, e621, Danbooru                  |
| Later   | Other booru sites                       |

## 3. Core Features

### 3.1 Query Builder
- Users pick a **base site**, then either build a query through the UI or paste an existing query directly.
- Users can save and organize their favorite queries.
- Each saved query has a link that opens the target site with the search prefilled.
- A query shown for its base site must come back **exactly as the user entered it**.
- Some presets may be offered later on that add certain exclusions or a group of tags together as to assist the user.
- Anything redundant (repeated tags, exclusions and inclusions of the same tag) should be sorted out before saving.
- When the user uses query form to save, we probably will need to show more warnings if any issues are encountered.

### 3.2 Cross-Site Query Conversion

#### Syntax Translation
- Each query is held in a single **site-independent internal form**.
- Each site has an **adapter** that:
  - parses that site's query syntax into the internal form, and
  - writes the internal form back out in that site's syntax.
- Converting a query means parsing it with the source site's adapter, then writing it out with the target site's adapter.
- When saving queries, we must make sure there is a way to retrieve the original query for the user in their base site's syntax. For now, I think we can just store the full query, as each query should be no more than a kilobyte. If we find a more efficient way to do so, we will move forward with that.
- **Initial scope:** basic features only (general tags, artists, OR/NOT operators). Advanced features (e.g. wildcards) are out of scope for now.
- The design stays open-ended so advanced features can be added later, with caveats where a site doesn't support them.

#### Tag Mapping
- Tags are mapped from their name on one site to the matching name on another.
- Mapping won't always be exact:
  - Some tags have no exact equivalent on the other site.
  - Several tags on one site may collapse into one tag on another.
    - *Example:* `size_difference` and `height_difference` are separate tags on Rule34, but both map to `size_difference` on e621.
- **Mappings can differ by direction.** When a tag covers content that the target site splits across several tags, it maps to **all** of them.
  - *Example:* e621 `size_difference` maps to both `size_difference` and `height_difference` on Rule34.

#### Master Tag Map
- A master map assigns a **common name** to each group of similar or matching tags across sites.
- Each mapping has one of three match levels:

| Level        | Meaning                                    |
|--------------|--------------------------------------------|
| Exact match  | Tag names and meanings are identical across sites       |
| Strong match | Tags are not identical but closely related |
| Weak match   | Tags are only loosely related              |

- Users can set a preferred **match confidence level**, with the following options
  - Exact matches only 
  - No weak matches (Exact and strong only) (Default)
  - All matches (Exact, strong, and weak included)
- We will also need some sort of artist mapping as artist names sometimes differ across sites, although I think this would be a bit more simple and does not need a confidence level or anything
- Similarly, we will need a character mapping. Since some sites split characters across different versions, this might actually need a confidence level.

#### Building the Map
- Curating mappings is expected to be the hardest part of the project.
- Planned approach:
  - Use AI/ML in some form to generate mappings.
  - Manually curate the most popular few hundred tags over time.
  - If traffic grows, let users submit mapping suggestions.

#### Sensitive Tags
- **Requirement:** tags for content that most booru users find extremely disgusting must never be mapped to tags that are unrelated or only loosely related.
  - A user should never get extreme content they didn't search for because of a loose mapping.
  - An extreme tag in a user's query should not quietly turn into something unrelated.

**Proposed approach (ideas, not yet decided):**
- **Sensitive flag:** each tag in the master map can be marked *sensitive*.
  - Seed the list from the booru sites' own default blacklists, where they have them, then curate it by hand.
- **No loose matches:** a sensitive tag can only be mapped as an *exact* match or perhaps a strong match. If we do go with strong matches as well, we must make certain the criteria is even stricter to set a strong match for sensitive content.
- **Mapping *into* sensitive is the main risk:** a non-sensitive tag must never map to a sensitive tag unless that mapping was reviewed by a person.
- **Treat included and excluded tags differently:**
  - *Included* sensitive tag (`gore`): map narrowly, exact or reviewed only.
  - *Excluded* sensitive tag (`-gore`): map broadly, to every related tag on the target site. Blocking too much is safer than blocking too little.
- **AI suggestions are never auto-accepted** for sensitive tags. They go into a manual review queue.
- **User suggestions** that involve a sensitive tag also require review.
- **Disable sensitive tags:** Users should be able to disable sensitive tags for any new queries they create. They won't see such tags in the search list, and if they manually add a query including them they will be warned before creation (or even blocked)
- **Sensitive tag warning:** If any sensitive tags show up during query creation/conversion, alert the user and ask for them permission before creating the query. Allow the user to disable this popup if they so desire
- **Automated test:** a check that fails if any non-sensitive tag maps to a sensitive tag without a review record, so a bad mapping can't slip in unnoticed.

### 3.3 Tag Data Caching
- Tag data and mappings are stored in our own database cache.
- The external sites' APIs are called from the backend only, never directly from the browser, to avoid sending them too many requests.
- Perhaps we just need to run some daily or weekly batch job, as new tags don't show up too frequently. The main concern would be recently created popular characters, and perhaps new artists who have shown up. But I think once a day should be fine.

## 4. Accounts & Authentication

- **Required:** username and password.
- **Optional:** email address, used only for account recovery.
  - No verification at sign-up.
  - An email address is **verified when it's added** to an account, so recovery emails can't go to a mistyped address or to someone else.
- If a user has no recovery email and loses access to their account, the account **cannot be recovered**, since there's no way to verify their identity.
- Users can set their accounts as public if so desired, so they can view their favorite queries without logging in, or share them with their degen friends
- Security basics:
  - Passwords are hashed with a strong algorithm (argon2id or bcrypt).
  - Password-reset tokens are single-use and expire quickly.
  - Login attempts are rate-limited.
  - Stored emails are treated as sensitive data.

## 5. Limits & Abuse Prevention

- **Global storage limit:** a size limit on the database. Once it's reached, new data and new accounts are blocked.
- **Per-user limit:** a cap on how many queries each user can save. Maybe 100? They shouldn't take up too much space I think, but I can't really see a reason a user might need more. Maybe I'm just not a real one
- **Spam protection:** prevent mass account creation by combining a CAPTCHA with rate limiting, not by IP limits alone.

## 6. Content & Linking

- The site does **not** host any NSFW content directly, apart from tag names.
- Saved queries link out to the external booru site with the search prefilled. The user can also click a button to copy the query to clipboard.
- An **18+ age gate** is shown to visitors along with a Terms of Use policy on first visit. If you accept both that will be cached in cookies, and you will only see the popup again if their is a change to the terms.
- **Long queries:** users get a popup advising them not to page through results with the Next button, and to open individual results in new tabs instead. This keeps load on the booru sites down and helps users avoid search-overload errors.

## 7. Data Model 

- Queries are represented as objects in the site-independent internal form, using an OOP approach.
- Site-specific queries are generated from that form by each site's adapter.
- Converted queries are generated when displayed, not stored, so improvements to the mapping apply to existing queries.
- **Proposed: exact round-trip.** Store the original query text exactly as entered, with its base site, alongside the parsed internal form.
  - On its base site, the query is shown from the stored original text.
  - The parsed form is used only for converting to other sites.
  - If the user edits the query in the UI builder, the text is regenerated from the edited form.

> Storage details to be decided.

## 8. Tech Stack (proposed)

| Layer      | Choice              | Role                                                        |
|------------|---------------------|-------------------------------------------------------------|
| Frontend   | React with Vite     | Runs in the browser: pages and the query builder UI         |
| Backend    | Express (Node)      | Runs on the server: accounts, saved queries, booru API calls |
| Database   | PostgreSQL          | Stores users, queries, tags, and mappings                   |
| DB access  | Drizzle or Prisma   | Lets the backend work with the database from TypeScript     |

- **Language:** both frontend and backend are written in TypeScript, so they can share the query model and adapter code.
- Use somewhat strict Typescript
- An ORM lets most database work happen in TypeScript instead of raw SQL.

### 8.1 Repository Layout
- **One repo** for frontend and backend, split into parts with npm workspaces.
- The query model and site adapters live in a shared `core` package that both the frontend and backend import.

```
booru-bookmarker/
├── apps/
│   ├── web/        # React + Vite frontend
│   └── api/        # Express backend
├── packages/
│   └── core/       # query model, site adapters, shared types
├── docs/
│   └── design.md
└── package.json    # declares the workspaces
```

- No other repos are needed for now. Tag-sync and seeding scripts can live in a `scripts/` or `data/` folder.
- **Possible later:** a separate repo for tag mappings, if they're curated as files that people can submit changes to.

## 9. Research Notes

> ⚠️ **Gathered by Claude only, not yet verified.** These notes come from web searches, and many sources are third-party (library docs, blogs, news). Check every claim and source yourself before relying on it. Notes were collected 2026-11-11.

### 9.1 Site APIs
- **e621**
  - Every request needs a descriptive User-Agent (e.g. `MyProject/1.0 (by username on e621)`). Browser user agents may get blocked.
  - Hard rate limit of 2 requests per second; about 1 per second is the suggested pace.
  - Daily bulk data exports (including tags) are available, and heavy users are pointed to them instead of the API. These could be used to fill the tag cache.
- **Rule34**
  - The API requires the account's `user_id` and `api_key`, found in the site's account options.
  - Missing credentials return HTTP 200 with an error message in the body, not a 401.
- **Gelbooru:** likely similar to Rule34 (nearly the same software). Not researched.
- **General guidance:**
  - Have each adapter respect the target site's tag limit so converted queries aren't rejected.
  - Avoid showing thumbnails from the booru sites' servers. It uses their bandwidth and puts NSFW images on our pages.

### 9.2 Query Syntax by Site (from general knowledge, not looked up)

| Feature       | Rule34 / Gelbooru         | e621                  | Danbooru                     |
|---------------|---------------------------|-----------------------|------------------------------|
| Software      | Gelbooru-based            | Danbooru fork         | Danbooru                     |
| Exclude tag   | `-tag`                    | `-tag`                | `-tag`                       |
| OR            | `{a ~ b}`                 | `~a ~b`               | `~a ~b`                      |
| Sort          | `sort:score`              | `order:score`         | `order:score`                |
| Rating values | `general`, `sensitive`, `questionable`, `explicit` (Gelbooru) | `s`, `q`, `e` | `g`, `s`, `q`, `e` |

- Danbooru's syntax is much closer to e621's than to Gelbooru's.
- Gelbooru's tag names largely follow Danbooru's, so tag mapping between those two should be easier than their syntax suggests.
- **Danbooru tag limit:** free accounts can search only about 2 tags at once; paid tiers allow more. Long converted queries would often hit this.
- Rule34's rating values may differ from Gelbooru's. Check them.

### 9.3 Sources
- [rs621 docs (e621 API rules)](https://docs.rs/rs621)
- [xe621 docs](https://docs.rs/xe621)
- [booru-rs Rule34 client](https://docs.rs/booru-rs/0.3.1/src/booru_rs/client/rule34.rs.html)
- **TODO:** Please find additional sources

## 10. Open Questions

- How exactly will AI/ML be used to generate mappings?
- How will user mapping suggestions be submitted and reviewed?
- How is the initial sensitive tag list built, and who reviews sensitive mappings?
- Which hosting provider should we use?
- How should queries be stored in detail?
