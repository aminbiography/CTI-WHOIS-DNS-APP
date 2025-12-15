Live URL: https://aminbiography.github.io/CTI-WHOIS-DNS-APP/

---  

## Explanation for a user (analyst/operator)  

### What this page is

This is a **browser-only** “domain intelligence collector.” You paste a **domain or hostname** (for example, `example.com` or `sub.example.com`) and click **Run Collection**. The page then pulls two categories of information:

1. **DNS intelligence** (technical resolution data) using **DNS-over-HTTPS (DoH)** via Google.
2. **Registration/WHOIS-like data** using **RDAP**, via `rdap.org`.

No server is required; your browser directly calls public endpoints.

### What you get back

The UI shows three panels:

* **DNS Intelligence (DoH)**: raw DNS responses for common record types (A, AAAA, NS, MX, TXT, CAA, SOA).
* **RDAP Registration Data**: WHOIS-equivalent structured JSON (registrar, domain status, nameservers, important dates, etc.) when available.
* **Unified Output**: a single JSON object that combines everything into one artifact you can copy or download.

### Quick Summary box (what it means)

* **Normalized**: the cleaned hostname (lowercase, scheme/path removed).
* **Registrable**: a “best guess” base domain (the page uses a simple last-two-label heuristic; see developer notes).
* **RDAP server**: a hint of which RDAP server supplied or is authoritative for the record.
* **Nameservers**: derived from DNS NS answers and/or RDAP nameservers.
* **A / AAAA**: IPv4/IPv6 addresses if present.
* **MX**: mail exchanger targets if present.

### Buttons

* **Run Collection**: runs DNS + RDAP queries and builds a unified JSON.
* **Clear**: resets outputs.
* **Copy JSON**: copies the unified JSON panel to clipboard.
* **Download JSON**: downloads the unified JSON as a timestamped file.

### Important limitations (user-facing)

* **Some registries return limited RDAP data**, or redact contacts and details.
* DNS answers can differ based on resolver behavior/caching and record visibility.
* **IP literals are rejected** (this flow is for domains; RDAP for IP exists but is not implemented here).

---

## Explanation for a CTI developer (engineering/extension perspective)

### High-level architecture

This is a **single HTML file** with embedded JS and CSS. It performs two network “collection legs” in parallel:

1. **DNS via DoH**

   * Endpoint: `https://dns.google/resolve?name=<host>&type=<TYPE>`
   * Record types queried: `A`, `AAAA`, `NS`, `MX`, `TXT`, `CAA`, `SOA`
   * Each response is stored raw (success or error object).

2. **RDAP via rdap.org redirector**

   * Endpoint: `https://rdap.org/domain/<domain>`
   * The tool calls RDAP for the “registrable guess” domain, not necessarily the full hostname.
   * Response is stored in `output.rdap` as `{ url, data }`.

The unified collector output is constructed as:

* `collected_at`, `input`, `normalized_host`, `registrable_guess`
* `sources` (documents provenance)
* `dns`, `rdap` (raw)
* `summaries` (parsed extractions)
* `errors` (any top-level errors)

### Core functions and responsibilities

#### Input normalization

* `normalizeDomain(input)`

  * Accepts raw user input (domain, hostname, or URL).
  * Uses `URL()` parsing when possible; otherwise falls back to regex cleanup.
  * Produces `hostname` only, lowercase, without trailing dot.

* `isIpLiteral(host)`

  * Basic IPv4/IPv6 literal detection.
  * Used to block IP-based input (the RDAP lookup path is domain-based only).

#### Registrable domain heuristic

* `registrableGuess(host)`

  * Returns “last two labels” as an approximation of eTLD+1.
  * Example: `a.b.example.com` → `example.com`
  * This is **not correct** for many public suffixes (e.g., `example.co.uk` would incorrectly become `co.uk` if the host is deeper), so treat it as a placeholder.

Developer improvement: replace with a Public Suffix List implementation (client-side PSL is possible, but adds size and update burden; server-side is more reliable).

#### HTTP fetching and error handling

* `fetchJson(url, opts)`

  * `fetch()` with `mode: "cors"` and `cache: "no-store"`.
  * Parses text into JSON; if parse fails, wraps as `{ _nonJson: true, text }`.
  * If HTTP status is not OK, throws an Error containing `status` and `body`.

This produces consistent error objects for each DNS type, and a top-level error for the combined run if something fails outside of per-type handling.

#### DNS collection

* `dohQuery(name, type)` → calls Google DoH JSON API.
* `collectDns(host)`

  * Iterates types; each query is awaited sequentially (not parallelized).
  * Stores either the raw response or an `{ error, message, status, body }` object per record type.

Developer improvement: run DNS queries in parallel with a bounded concurrency pattern to reduce total runtime without spiking requests.

#### RDAP collection

* `collectRdap(domain)`

  * Fetches `https://rdap.org/domain/<domain>`
  * Returns `{ url, data }`

Developer improvement: handle redirects explicitly (or use authoritative server directly) and add support for:

* `ip/<address>` RDAP
* `autnum/<asn>` RDAP
* entity lookups when present

#### Summarization / extraction

* `extractDnsSummary(dns)`

  * Extracts `A`, `AAAA`, `NS`, `MX` into arrays from the `Answer` section of Google DoH JSON.
  * Note: MX “data” is typically `"priority host."` and is not parsed into priority vs host; it is just trimmed.

* `extractRdapSummary(rdapData)`

  * Extracts:

    * `handle`
    * `ld` (`ldhName` or `unicodeName`)
    * `status` array
    * `nameservers` (from RDAP `nameservers`)
    * `registrar` (search entities where roles include `"registrar"`; attempts to read vCard `fn`)
    * `events` (maps action → date)

* `findRdapServerHint(rdap)`

  * Attempts to infer server origin from RDAP `links` (`rel: self/related`) and returns `.origin`.

### UI flow

* `run()` is the main orchestrator:

  1. Normalize and validate input.
  2. Reset UI outputs and set status.
  3. Create the `output` object scaffold.
  4. `Promise.all([collectDns(host), collectRdap(registrable)])`
  5. Populate raw panels.
  6. Compute summaries, fill quick summary.
  7. Render final unified JSON and enable copy/download.

### Security, privacy, and operational notes (important for CTI usage)

* This is **not stealthy collection**: the user’s browser directly contacts Google DoH and rdap.org (and whatever RDAP infrastructure backs that). Expect these lookups to be observable by those services.
* For sensitive investigations, you may want:

  * Configurable DoH resolver (Cloudflare, Quad9, in-house) and/or Tor-aware approach.
  * A server-side proxy to control logging, caching, rate limits, and attribution.
* RDAP data can include PII-like contact data in some jurisdictions; your pipeline should respect legal/compliance requirements and retention policies.

### Recommended developer enhancements (CTI-friendly)

If you want to make this “production-grade” for CTI workflows, prioritize:

1. **Correct registrable domain computation**

   * Use PSL-based eTLD+1 rather than last-two-label heuristic.

2. **More DNS record coverage and parsing**

   * Add: `PTR` (if you later support IP input), `SRV`, `DMARC`/`SPF` extraction from TXT, `DS/DNSKEY` (DNSSEC), `HTTPS/SVCB`.
   * Parse MX into `{priority, host}`.

3. **Risk/CTI enrichments (still browser-only)**

   * Reverse IP ASN/Org via a public API (be mindful of CORS and rate limits).
   * Basic indicators: suspicious TTLs, fast-flux hints, parking providers, NS patterns.
   * Lightweight scoring: registrar reputation, domain age, recent changes.

4. **Better error reporting**

   * Separate DNS per-type errors from RDAP errors in the UI.
   * Show HTTP status and selected response snippets to help debugging.

5. **Export formats**

   * STIX 2.1 object mapping (DomainName, IPv4Address, IPv6Address, Relationship).
   * CSV summary export for quick triage.

---

