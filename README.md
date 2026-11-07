# UK Politics Decoded Parliamentary Dataset

MP Lookup Parliamentary Activity Dataset
The Parliamentary Term Statistics tab provides structured, JSON only analytics for debates, contributions, divisions, speeches, interventions, and party level activity across parliamentary terms. Powered by the Hansard Timeline Stats API, Hansard Debate API, and Members API, it offers a rolling 7 years of reliable parliamentary trend data, Free of charge.

We currently use this dataset for a MP lookup tool on the ukpoliticsdecoded.uk website. https://ukpoliticsdecoded.uk

Fetch files over HTTPS (or clone the repo) and use them directly.

The files are generated and refreshed automatically from official UK Parliament and ONS open data. Every commit contains only files whose content really changed, so the git history doubles as a change log.

> **Not an official Parliament product.** This is derived data. Check the
> [source APIs](#data-sources) for anything where accuracy matters.

## Quick start

Base URL for raw files:

```
https://raw.githubusercontent.com/UKPoliticsDecoded/MP-Lookup-Parliamentary-Activity/main/
```

**Find an MP from a postcode**

1. Take the outcode (the part before the space, e.g. `AB10` from `AB10 1AA`).
2. Read `postcodes/postcode_constituencies.json` → `outcodes["AB10"].constituencies`
   gives constituency slug(s). If `unique` is `false` the outcode spans several constituencies, resolve the full postcode with
   [Postcodes.io](https://postcodes.io) (`parliamentary_constituency` field), then map the name with `postcodes/constituency_lookup.json`.
3. Open `constituencies/{slug}.json` → `current_mp.file` points to the MP file.

**Find an MP by name or party:** use `mps/index.json` (one compact row per MP), then open
`mps/{member_id}.json`.

```python
import json, urllib.request

BASE = "https://raw.githubusercontent.com/UKPoliticsDecoded/MP-Lookup-Parliamentary-Activity/main/"
get = lambda p: json.load(urllib.request.urlopen(BASE + p))

slugs = get("postcodes/postcode_constituencies.json")["outcodes"]["AB10"]["constituencies"]
const = get(f"constituencies/{slugs[0]}.json")
mp = get(const["current_mp"]["file"])
print(mp["name"]["display"], mp["party"]["name"],
      mp["voting"]["voting_participation_pct"], "% voting participation")
```

## Layout

| Path | Contents |
| --- | --- |
| `mps/index.json` | Index of all MPs: id, name, party, constituency, file path. |
| `mps/{member_id}.json` | Full profile of one MP (see below). Includes former MPs who served in the window. |
| `constituencies/{slug}.json` | Constituency details, current and historic MPs, election results, boundary summary. |
| `constituencies/boundaries/{slug}.geojson` | Constituency boundary polygon (GeoJSON). |
| `committees/index.json` | Index of all committees. |
| `committees/{slug}.json` | Committee details, chairs, members, per-member attendance, activity. |
| `postcodes/postcode_constituencies.json` | Outcode → constituency slug(s), with a `unique` flag. |
| `postcodes/postcode_lookup.json` | Outcode → location, admin districts, country. |
| `postcodes/constituency_lookup.json` | Look up a constituency by name (normalised), ONS code or slug. |
| `statistics/voting_participation.json` | Voting participation for every MP. |
| `statistics/committee_attendance.json` | Committee attendance for every MP on a committee. |
| `statistics/member_activity.json` | Activity summary (debates, questions, ...) for every MP. |
| `statistics/debate_activity.json`, `committee_activity.json`, `parliamentary_activity.json` | Aggregate activity statistics. |
| `validation/report.json` | Result of the latest automated consistency checks. |
| `changelog/{YYYY-MM}.json` | What changed, by month. |
| `meta.json` | Generation time, record counts, window, sources and known limitations. |

All files include `schema_version`. Keys are sorted and output is deterministic, so diffs are
clean. Dates are ISO 8601 (`YYYY-MM-DD`).

## MP file (`mps/{member_id}.json`)

| Field | Meaning |
| --- | --- |
| `member_id`, `external_ids` | Parliament Members API id and other identifiers. |
| `name` | `display`, `list_as`, `address_as`, `full_title`. |
| `party`, `gender`, `is_current`, `thumbnail_url`, `synopsis` | Basic profile. |
| `membership` | Start/end date and status of the seat. |
| `constituency` | Name, slug, ONS code and path to the constituency file. |
| `contact` | Published parliamentary and constituency contact details. |
| `roles` | Government, opposition, ministerial and other roles. |
| `committees` | `current` and `history` committee memberships with roles and dates. |
| `committee_attendance` | Per committee: meetings attended / available and percentage. |
| `voting` | Divisions eligible/voted and participation %, by year and by Parliament. |
| `activity` | Written questions, oral questions, debate contributions, debates participated in, early day motions, bills sponsored, petitions presented; totals and `annual` breakdown. |
| `activity_summary` | Headline figures used by the statistics files. |
| `service_history` | Elections contested, party affiliations, past representations. |
| `window` | The rolling window the statistics cover. |

## How the statistics are defined

- **Voting participation** = divisions voted in ÷ Commons divisions the MP was eligible for (held while they were an MP), as a percentage. Tellers count as votes.
  Absence is not necessarily non-attendance: MPs can be paired, ill, on maternity or paternity leave, or serving as Speaker or Deputy Speaker (who do not normally vote).
- **Committee attendance** = formal meetings attended ÷ meetings for which Parliament published an attendance list while the MP was a member. Meetings without a published list are excluded, and `attendance_pct` is `null` when there is nothing to measure.
- **Activity counts** are taken from Hansard contributions, written and oral question records, early day motions and the Bills API within the window. Counts are per calendar year in `annual`.
- The window is rolling (default 7 years, see `meta.json → window`). Earlier history is not included.

## Known limitations

- Postcodes are published at **outcode** level, so some outcodes span several
  constituencies (`unique: false`). Use Postcodes.io for exact postcode matching.
- Petitions presented are inferred from Hansard petition debate sections.
- No source exists for MP interventions or bills supported (only sponsored), so these are not provided.
- Committee attendance only exists for meetings where Parliament publishes it.
- Former MPs are included only if they served within the window.
- Election results come from the Parliament Members API.

The full, current list is in `meta.json → limitations`.

## Data sources

- [UK Parliament Members API](https://members-api.parliament.uk/)
- [UK Parliament Committees API](https://committees-api.parliament.uk/)
- [UK Parliament Commons Votes API](https://commonsvotes-api.parliament.uk/)
- [Hansard API](https://hansard-api.parliament.uk/)
- Written Questions / Statements, Oral Questions and Motions, and Bills APIs (UK Parliament)
- [ONS Open Geography Portal](https://geoportal.statistics.gov.uk/) (constituency boundaries)
- [Postcodes.io](https://postcodes.io) (outcode geography)

## Update schedule

| Frequency | What is refreshed |
| --- | --- |
| Hourly | Hansard contributions and activity counts |
| Daily | MPs, committees, roles |
| Weekly | Validation and full rebuild |
| Monthly | Postcodes and boundaries |

## Licence and attribution

Contains Parliamentary information licensed under the
[Open Parliament Licence v3.0](https://www.parliament.uk/site-information/copyright-parliament/open-parliament-licence/).
Boundary data contains public sector information licensed under the [Open Government Licence v3.0](https://www.nationalarchives.gov.uk/doc/open-government-licence/version/3/) (Source: Office for National Statistics).
Postcode data courtesy of Postcodes.io.
Please credit "UK Politics Decoded" and the underlying sources when you reuse the data.
