# bank-feed-dist

A signed file of public facts about Indian banks, taken only from Reserve Bank of India (RBI) press releases and
the lists published by the Deposit Insurance and Credit Guarantee Corporation (DICGC). It states what was announced,
dated and linked to the source. It never rates or judges a bank.

Files are replaced by an automated build; please do not open pull requests. RBI and DICGC content is public record,
reproduced with attribution.

## Files

| File | What it is |
|---|---|
| `banks.signed.json` | The feed: one signed JSON envelope (below). About 1.5 MB, about 1,870 banks. |
| `ed25519_pub.pem` | The public key of the current signing key, for checking the file by hand. |
| `README.md` | This file. |

Download the raw file, for example
`https://raw.githubusercontent.com/rajagrawal1/bank-feed-dist/main/banks.signed.json`.
Take the whole file; there is no per-bank request. Send `If-None-Match` with the last ETag to get a `304` when
nothing changed.

## The envelope and its signature

```json
{"format": 1, "keyId": "<16 hex>", "sig": "<base64>", "payload": "<the feed, as a JSON string>"}
```

- `sig` is an Ed25519 signature over the **UTF-8 bytes of the `payload` string**, exactly as it appears (do not
  re-serialise it before checking).
- `keyId` is the first 16 hex characters of SHA-256 over the raw 32-byte public key.
- Keys in use:
  - `0d3f9f11ebb1bcc4`: the signing key (its public half is `ed25519_pub.pem`, raw base64
    `1ZlRtd/0Gj1etXzONMqnpby20QYUmG1Y1oj6hl/kV9E=`).
  - `febfd50aa7cb452e`: an offline backup key (raw base64 `T4JZw52Nv56/YwNIDLNlzaR7KV3ZfkHZbzi8tiNj204=`). If the
    signing key is ever replaced, files may be signed with this one, so a reader that trusts both keeps working.
- A reader should refuse: an unknown `format`, an unknown `keyId`, a bad signature, a payload whose own `keyId`
  differs from the envelope's, an expired file, and a file whose `version` is lower than the one it already holds.

Checking by hand with Python (`pip install cryptography`):

```python
import base64, json
from cryptography.hazmat.primitives.serialization import load_pem_public_key

env = json.load(open("banks.signed.json", encoding="utf-8"))
key = load_pem_public_key(open("ed25519_pub.pem", "rb").read())
key.verify(base64.b64decode(env["sig"]), env["payload"].encode("utf-8"))  # raises if altered
feed = json.loads(env["payload"])
print(feed["schema"], feed["version"], len(feed["banks"]))
```

## Freshness

| Field | Meaning |
|---|---|
| `version` | Unix seconds of the build. Always goes up; never accept a lower one. |
| `generated_at` | When the build ran (ISO, UTC). |
| `expires` | `generated_at` + 14 days. Do not apply an expired file; keep what you had and treat it as old. |
| `checked.rbi_releases_to` | The day of the last successful read of RBI's press releases (not the newest release's date). |
| `checked.dicgc_aid_list`, `checked.dicgc_insured_list`, `checked.dicgc_deregistered_list` | The "as on" date each DICGC list states. Any can be `null`. |

The file is rebuilt every 6 hours (RBI releases) and after a nightly full refresh. It is committed only when its
content changes, so a quiet day still gets about one new commit as `checked.rbi_releases_to` moves on. A file
older than about 3 days means the build has stopped; facts may have changed since.

Before publishing, the build refuses to replace the last good file when: an RBI restriction release from the last
120 days names no bank it knows (until a person reviews it), the number of restricted banks moves by more than 5,
the bank count drops by more than 2%, a previously published `id` is missing, or the version would not go up. A
blocked build leaves the last good file in place.

## The feed (schema 1)

Readers must ignore keys they do not know. Within schema 1 a key is only ever added, never renamed or removed. A
higher `schema` number means a reader that does not know it should keep what it has.

Top level: `schema` (1), `version`, `expires`, `keyId`, `generated_at`, `checked` (above), `banks` (sorted by `id`).

A bank:

| Key | Meaning |
|---|---|
| `id` | Stable identifier, `[a-z0-9_]+`. See "Stable ids" below. |
| `name` | The registry name, in RBI's or DICGC's spelling. |
| `aka` | Other names for search: short forms, banks it absorbed, other spellings. |
| `sms_names` | Sender names that banks use in SMS for this bank. Usually empty. |
| `type` | `psb` (public sector), `private`, `small_finance`, `payments`, `ucb` (urban co-operative), `stcb` (state co-operative), `dccb` (district central co-operative). |
| `state`, `city` | Co-operative banks only, from DICGC's list; else `null`. |
| `insured` | `true`: on DICGC's insured-banks list of `checked.dicgc_insured_list`. `false`: not on it. `null`: not matched to the list, so nothing is said. |
| `status` | `active`, `merged`, `licence_cancelled`. |
| `status_source` | `rbi_release` or `dicgc_list`; `null` when active. |
| `status_since`, `status_url` | When the source states them. |
| `merged_into` | `{"name", "id"}` for a merged bank; `id` is `null` when the successor is not in the file. |
| `restriction` | Always present: `under` (bool), `basis` (`rbi_release`, `dicgc_list`, both or none), `since`, `facts`, `rbi_date`, `rbi_url`, `dicgc_as_of`. |
| `record` | RBI actions on this bank in the last 5 years, newest first. |

`restriction.under` is `true` when DICGC lists the bank as under restrictions, or RBI's latest restriction release
for it in the last 12 months imposes, extends or changes them; a later withdrawal, licence cancellation or merger
ends the RBI basis. `restriction.facts` holds only what RBI's text states:

- `withdrawal_limit_inr`: the withdrawal cap per depositor, in whole rupees (`0` = no withdrawals allowed).
- `in_force_from`: the day the restrictions apply from.
- `runs_until`: an extension's stated end ("to close of business on February 07, 2027"), else a first release's
  stated period counted on from its start, same day ("six months from the close of business on October 21, 2026"
  gives `2027-04-21`). No date when that day does not exist. RBI's directions are "subject to review": they can be
  extended, changed or lifted before or after this day.

A `record` row: `date`, `kind` (`penalty`, `directions_new`, `directions_extension`, `directions_modified`,
`directions_withdrawal`, `supersession`, `administrator_change`, `licence_cancellation`, `moratorium`,
`merger_scheme`), `title` (RBI's own), `url` (RBI's release), `amount_inr` (fines, when stated), `why` (fines only:
codes for the RBI rules the release names, by fixed phrase rules: `kyc`, `deposit_interest`, `loan_interest`,
`lending_rules`, `loans_to_directors`, `exposure`, `asset_classification`, `fraud_reporting`, `cyber`,
`customer_protection`, `current_accounts`, `outsourcing`, `other`), and `plain_en` / `plain_hi`: one plain sentence
from a fixed template, with no dates and no bank names inside. Only releases whose title names the bank are
included.

## Stable ids

- An `id`, once published, never changes and never disappears. A build that drops one is refused.
- A merged bank keeps its `id`, with `status: merged` and `merged_into` pointing at the successor.
- A bank whose licence is cancelled keeps its `id`, with `status: licence_cancelled`.
- Names can change (a rename, a corrected spelling); ids do not. Store ids, never names.

## What is not here

Only RBI press releases and DICGC lists are sources. No ratings, scores, predictions, share prices, court cases or
agency investigations, and no news text. Foreign banks and regional rural banks are not in the registry yet.
