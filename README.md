# VoidOrigin

Finds the real server behind a CDN — and proves it's the right one.

![license](https://img.shields.io/badge/license-MIT-blue?style=flat-square)
![python](https://img.shields.io/badge/python-3.8%2B-3776AB?style=flat-square)

## What it does

When a site sits behind Cloudflare, Akamai, Fastly or similar, visitors never
reach the actual server. The CDN takes the request, filters it, and passes on
what it allows. That's the point — the WAF, the rate limiting and the DDoS
protection all live at that edge.

None of it helps if the origin server is still reachable directly. If you can
find the IP the CDN forwards to, you can talk to the application with the whole
edge stepped over: no WAF rules, no rate limits, no logging in the CDN dashboard.
Origins get left exposed constantly, usually because the site moved behind a CDN
years after the server was first set up and nobody restricted it afterwards.

VoidOrigin looks for that address. It gathers candidate IPs from several
independent places — Certificate Transparency logs, a subdomain brute force,
and Shodan if you give it a key — then throws away everything that belongs to a
known CDN range.

The part that matters is what it does next. Most tools stop at a list of
leftover IPs and leave you guessing. VoidOrigin connects to each candidate
**directly**, presenting the target's hostname in the TLS handshake and the
`Host` header, and compares what comes back against the real site: does the
certificate cover the domain, does the page title match, does the body look the
same? Each candidate gets a score and the reasons behind it, so you get "this
one is the origin, and here's why" instead of a pile of addresses to work
through by hand.

## Why you'd use it

- **It confirms, rather than guessing.** Candidates are verified against a live
  baseline and scored on certificate, title and body evidence.
- **Several independent sources**, so a site that hid one trail is still found
  through another.
- **Knows the CDN ranges** and removes them, including fetching Cloudflare's
  published lists rather than relying on a stale copy.
- **Flags side-findings**, like internal RFC1918 addresses leaking into public
  DNS.
- **Offline mode** for when you only want DNS and no outbound HTTP.

## Install

```bash
git clone https://github.com/CypherNova1337/VoidOrigin
cd VoidOrigin
pip install -r requirements.txt
```

Needs Python 3.8 or newer.

## Usage

```bash
python3 voidorigin.py example.com
```

That runs everything: collects candidates, filters the CDN, verifies what's
left, and prints a report.

**Several domains at once**

```bash
python3 voidorigin.py example.com example.org
python3 voidorigin.py --targets scope.txt
```

**Save the results**

```bash
python3 voidorigin.py example.com -o results.json
python3 voidorigin.py example.com --csv candidates.csv
```

**Add Shodan**

```bash
export SHODAN_API_KEY=...
python3 voidorigin.py example.com
```

Shodan often knows about hosts that were indexed before the site moved behind a
CDN, which is exactly the history you're looking for.

**Quiet the noise**

```bash
python3 voidorigin.py example.com --no-brute --no-ct
```

Useful when you already have candidates and only want the verification step.

**Look, but touch nothing**

```bash
python3 voidorigin.py example.com --offline
```

DNS only — no outbound HTTP, no connections to the target.

## Options

| Flag | Default | What it does |
|---|---|---|
| `domain` | — | One or more target domains |
| `--targets FILE` | — | File of domains, one per line (`#` comments allowed) |
| `-o FILE` | — | Write full results as JSON |
| `--csv FILE` | — | Write candidate rows as CSV |
| `--json` | off | Print JSON to stdout instead of a report |
| `-w FILE` | bundled | Custom subdomain wordlist |
| `-t` | `40` | Concurrent DNS and probe workers |
| `--dns-timeout` | `5.0` | Per-query DNS timeout, in seconds |
| `--http-timeout` | `8.0` | Probe timeout, in seconds |
| `--resolver IP[,IP]` | system | Use specific DNS resolvers |
| `--shodan-key KEY` | — | Shodan API key, or set `SHODAN_API_KEY` |
| `--no-ct` | off | Skip Certificate Transparency lookups |
| `--no-brute` | off | Skip the subdomain brute force |
| `--no-verify` | off | Skip active verification — collect candidates only |
| `--verify-all` | off | Verify every non-CDN IP, including the apex |
| `--offline` | off | DNS only; no outbound HTTP at all |
| `-q` | off | Print the final report and nothing else |
| `-v` | off | Verbose progress |
| `--no-color` | off | Disable coloured output |
| `-V` | — | Print version and exit |

## Reading the results

Each candidate gets a score and the evidence behind it. The strongest signals
are a **certificate that covers the domain** and a **page title matching the
live site** — together they're hard to explain any other way. Body similarity
supports those but is weaker on its own, since shared hosting and default pages
can look alike.

A high score means the IP served the target's site when asked directly. That is
the finding: the origin is reachable without going through the CDN.

## Good to know

- **Verification connects to the candidate**, so it is not a passive scan.
  `--offline` or `--no-verify` if you need to stay quiet.
- **A clean run doesn't prove the origin is locked down.** It proves these
  sources didn't reveal it. A properly restricted origin looks the same as one
  you simply haven't found yet.
- **Shodan changes the results noticeably.** If a target looks like a dead end
  without a key, it's worth trying again with one.
- **Cloud front-ends move.** An IP that verified last month may belong to
  somebody else now, so re-check before acting on old output.

## Authorised use

Only run this against domains you own or have written permission to test.
Connecting directly to an origin is an interaction with that host, and
reaching a server the owner believes is protected is exactly the kind of
finding that needs to be in scope before you go looking.

## License

MIT — see [LICENSE](LICENSE).
