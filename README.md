# ATS Job Board APIs — public endpoints for 11 applicant tracking systems

A practical, tested reference of the **public, unauthenticated endpoints** that applicant tracking systems (ATS)
expose for company career pages: **Workday, Greenhouse, Lever, Ashby, Workable, Recruitee, Personio,
SmartRecruiters, Breezy HR, Teamtailor, Rippling**.

Use it to build job boards, job alerts, hiring-trend dashboards or to feed AI agents with fresh job data.
Every endpoint below was verified in October 2026.

> **Don't want to maintain 11 integrations?** The
> [Career Sites Jobs Feed](https://apify.com/mersetti/ats-jobs-feed) Actor wraps all of them into one unified
> schema (salary, remote flag, description, apply URL) with new/closed job monitoring.

---

## Quick reference

| ATS | Careers URL pattern | Public endpoint | Format | Pagination | Description included |
|---|---|---|---|---|---|
| Greenhouse | `boards.greenhouse.io/{slug}` | `GET https://boards-api.greenhouse.io/v1/boards/{slug}/jobs?content=true` | JSON | none (all jobs) | yes (`content=true`, HTML-escaped) |
| Lever | `jobs.lever.co/{slug}` | `GET https://api.lever.co/v0/postings/{slug}?mode=json` (EU: `api.eu.lever.co`) | JSON | none | yes (`descriptionPlain`, `lists`) |
| Ashby | `jobs.ashbyhq.com/{slug}` | `GET https://api.ashbyhq.com/posting-api/job-board/{slug}?includeCompensation=true` | JSON | none | yes (`descriptionPlain`) |
| Workable | `apply.workable.com/{slug}` | `GET https://apply.workable.com/api/v1/widget/accounts/{slug}?details=true` | JSON | none | yes (`description` HTML) |
| Recruitee | `{slug}.recruitee.com` | `GET https://{slug}.recruitee.com/api/offers/` | JSON | none | yes (`description` + `requirements`) |
| Personio | `{slug}.jobs.personio.de` | `GET https://{slug}.jobs.personio.de/xml?language=en` (or `.com`) | XML | none | yes (`jobDescriptions`) |
| SmartRecruiters | `jobs.smartrecruiters.com/{slug}` | `GET https://api.smartrecruiters.com/v1/companies/{slug}/postings?limit=100&offset=N` | JSON | `offset`/`limit` (max 100) | detail call: `/postings/{id}` |
| Workday | `{tenant}.wd{N}.myworkdayjobs.com/{site}` | `POST https://{tenant}.wd{N}.myworkdayjobs.com/wday/cxs/{tenant}/{site}/jobs` | JSON | `offset`/`limit` (max 20) | detail call: `GET …/{site}{externalPath}` |
| Breezy HR | `{slug}.breezy.hr` | `GET https://{slug}.breezy.hr/json` | JSON | none | no |
| Teamtailor | `{slug}.teamtailor.com` or custom domain | `GET https://{careers-domain}/jobs.rss` | RSS/XML | none | yes (`description`, HTML-escaped) |
| Rippling | `ats.rippling.com/{slug}/jobs` | `GET https://ats.rippling.com/api/v2/board/{slug}/jobs?page=N` | JSON | `page` (20/page, `totalPages`) | detail call: `/jobs/{id}` |

---

## Details and gotchas

### Greenhouse
```bash
curl "https://boards-api.greenhouse.io/v1/boards/stripe/jobs?content=true"
```
- Response: `{ jobs: [{ id, title, location.name, departments[], offices[], first_published, updated_at, absolute_url, content }] }`.
- `content` is **double-encoded HTML** (`&lt;p&gt;…`) — decode entities before stripping tags.
- `absolute_url` often points to the company's own careers site (e.g. `stripe.com/jobs/search?gh_jid=…`).
- Salary is usually only inside `content` text ("The annual base salary range is $X – $Y").

### Lever
```bash
curl "https://api.lever.co/v0/postings/palantir?mode=json"
```
- Array of postings: `id, text (title), categories{team, department, location, commitment, allLocations}, workplaceType, country, createdAt (ms), hostedUrl, applyUrl, descriptionPlain, lists[], salaryRange?`.
- `workplaceType` is `remote | hybrid | on-site | unspecified`.
- EU-hosted companies live on `api.eu.lever.co` / `jobs.eu.lever.co`.

### Ashby
```bash
curl "https://api.ashbyhq.com/posting-api/job-board/openai?includeCompensation=true"
```
- `jobs[]`: `id, title, department, team, employmentType, location, secondaryLocations[], publishedAt, isRemote, workplaceType, isListed, jobUrl, applyUrl, descriptionPlain, compensation`.
- `compensation.summaryComponents[]` has structured `minValue / maxValue / currencyCode / interval` — the best salary data of any ATS.
- Filter out `isListed: false`.

### Workable
```bash
curl "https://apply.workable.com/api/v1/widget/accounts/huggingface?details=true"
```
- `{ name, description, jobs: [{ title, shortcode, employment_type, telecommuting, department, url, application_url, published_on, city, state, country, locations[], description }] }`.
- `telecommuting: true` = remote.

### Recruitee
```bash
curl "https://bunq.recruitee.com/api/offers/"
```
- `offers[]` with `id, title, department, location, city, country_code, remote, hybrid, on_site, employment_type_code, experience_code, salary{min,max,period,currency}, published_at ("2026-10-13 10:58:56 UTC"), careers_url, careers_apply_url, description, requirements`.
- Dates are **not ISO** — replace the space and ` UTC` before parsing.

### Personio
```bash
curl "https://personio.jobs.personio.de/xml?language=en"
```
- XML `<position>` elements: `id, subcompany, office, additionalOffices/office, department, recruitingCategory, name, employmentType, seniority, schedule, yearsOfExperience, createdAt, jobDescriptions/jobDescription{name, value (CDATA HTML)}`.
- Some companies are on `.jobs.personio.com` instead of `.de` — try both.
- Job URL: `https://{slug}.jobs.personio.de/job/{id}`.

### SmartRecruiters
```bash
curl "https://api.smartrecruiters.com/v1/companies/BoschGroup/postings?limit=100&offset=0"
curl "https://api.smartrecruiters.com/v1/companies/BoschGroup/postings/{id}"
```
- List: `totalFound, content[{ id, name, company.name, releasedDate, location{city, region, country, remote, hybrid, fullLocation}, department.label, function.label, typeOfEmployment.label, experienceLevel.label }]`.
- Descriptions only in the detail call: `jobAd.sections.{companyDescription, jobDescription, qualifications, additionalInformation}.text`.
- The company identifier is case-sensitive and not always the brand name (`BoschGroup`, not `Bosch`).

### Workday
```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"limit":20,"offset":0,"searchText":"","appliedFacets":{}}' \
  "https://nvidia.wd5.myworkdayjobs.com/wday/cxs/nvidia/NVIDIAExternalCareerSite/jobs"
```
- List: `total, jobPostings[{ title, externalPath, locationsText, postedOn, bulletFields[] }]`. `limit` max is **20**.
- `total` is only reliable on the first page; later pages may return `0`.
- `postedOn` is human text: `Posted Today`, `Posted Yesterday`, `Posted 3 Days Ago`, `Posted 30+ Days Ago`.
- `locationsText` can be `"2 Locations"` — the detail call gives the real list.
- Detail: `GET https://{tenant}.wd{N}.myworkdayjobs.com/wday/cxs/{tenant}/{site}{externalPath}` →
  `jobPostingInfo{ title, jobDescription, location, additionalLocations, startDate, timeType, jobReqId, country.descriptor, externalUrl }`.
- `searchText` does server-side search — for big tenants (thousands of jobs) query per keyword instead of paging everything.
- The site name is case-sensitive; strip locale segments like `/en-US/` from careers URLs.

### Breezy HR
```bash
curl "https://breezy.breezy.hr/json"
```
- Array: `id, friendly_id, name, url, published_date, type.name, location{city, state, country, is_remote}, locations[], department, salary (free text), company.name`.
- No descriptions in the JSON.

### Teamtailor
```bash
curl "https://career.teamtailor.com/jobs.rss"
```
- RSS `<item>`: `title, description (escaped HTML), pubDate, link, remoteStatus (none | hybrid | fully), guid, tt:locations/tt:location{tt:name, tt:city, tt:country}, tt:department, tt:role`.
- Most companies use a **custom domain** — detect Teamtailor by `teamtailor` assets in the careers page HTML.

### Rippling
```bash
curl "https://ats.rippling.com/api/v2/board/rippling/jobs?page=0"
curl "https://ats.rippling.com/api/v2/board/rippling/jobs/{id}"
```
- List: `items[{ id, name, url, department.name, locations[{name, country, city}] }], page, pageSize, totalItems, totalPages`.
- No dates in the list — the detail call has `createdOn, employmentType, workLocations[], payRangeDetails[{rangeStart, rangeEnd, currency, frequency}], description{company, role}`.

---

## Detecting the ATS behind a careers page

Most "careers" pages embed or link to their ATS. Fetch the HTML and look for:

| Pattern in HTML | ATS |
|---|---|
| `boards.greenhouse.io/…`, `job-boards.greenhouse.io/…`, `embed/job_board?for=` | Greenhouse |
| `jobs.lever.co/…` | Lever |
| `jobs.ashbyhq.com/…` | Ashby |
| `apply.workable.com/…` | Workable |
| `….recruitee.com` | Recruitee |
| `….jobs.personio.de` / `.com` | Personio |
| `jobs.smartrecruiters.com/…`, `careers.smartrecruiters.com/…` | SmartRecruiters |
| `….wd5.myworkdayjobs.com/…` | Workday |
| `….breezy.hr` | Breezy HR |
| `teamtailor-cdn`, `assets.teamtailor` | Teamtailor (custom domain) |
| `ats.rippling.com/…` | Rippling |

Some sites (e.g. Stripe) render jobs server-side from the ATS API without exposing any link — use the
ATS slug directly in that case.

## Normalizing salaries from descriptions

Many ATS don't have a salary field, but US postings include pay ranges in the text. A regex that works well:

```js
/(US\$|CA\$|A\$|[$€£])\s?(\d{1,3}(?:[,.\s]\d{3})*|\d+)(?:\.\d+)?\s?([kK])?\s?(?:-|–|—|to)\s?(?:US\$|CA\$|A\$|[$€£])?\s?(\d{1,3}(?:[,.\s]\d{3})*|\d+)(?:\.\d+)?\s?([kK])?/
```

Reject implausible matches (yearly < 10,000, hourly > 1,000) to avoid catching signing bonuses or company sizes.

## Being a good citizen

- These are public endpoints meant for publishing jobs on career sites and job boards. Cache results,
  keep request rates low, and identify your client with a User-Agent.
- Job postings contain no personal data; descriptions remain the employer's content.

## Ready-made solution

[**Career Sites Jobs Feed on Apify**](https://apify.com/mersetti/ats-jobs-feed) — paste careers URLs, get one unified
dataset from all 11 ATS, with keyword/location/remote/date filters and a monitor mode that outputs only new jobs.
Pay per result, schedule it daily, export to JSON/CSV/Excel or connect to Sheets, Slack, Zapier, Make.

---

Contributions welcome — open an issue if an endpoint changes or you know another public ATS feed.
