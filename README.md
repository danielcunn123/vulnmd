# VulnMD

VulnMD expands a CherryTree investigation database into a small, inspectable
Markdown vulnerability database. Each managed machine receives a page linking
its imported findings to advisory, CVE, source-tree, deterministic-diff,
Windows-update, and optional Gemini evidence. The same repository can be
published as a static Apache website.

v1.6 keeps the renewed large-capture and package/update workflows from v1.5.5
and adds a portable Maltego graph export. One source PCAP is staged under
`builder/pcap`, then analyzed
against literal IP/MAC identities parsed from each machine's Nmap scan.
VulnMD writes directional endpoint statistics and one analyst-ready PCAP per
declared machine. Capture-only hosts are reported as unmatched observations;
they are never added as managed machines.

This release adds a separate package/update capture path. `extract-pcap`
streams a second capture, writes another per-machine PCAP set, aggregates
visible clear-text HTTP requests, recognizes common package filenames, and
links exact package/source names and versions to existing advisory source
manifests. Download evidence never becomes an automatic installed/patched
verdict.

`export-maltego` writes an importable MTGX graph, raw GraphML, and an audit
manifest. The graph uses standard Netblock, IPv4 Address, Port, and CVE
entities for transform compatibility. Network parents contain declared
devices, machines link to Nmap ports and advisories, and advisories link to
CVEs. Critical CVEs are red, high CVEs orange, and medium CVEs yellow.

## Debian 13 installation

```bash
sudo apt update
sudo apt install -y \
  python3 python3-pytest git ca-certificates jq ripgrep unzip zip rsync \
  apache2 tshark python3-maxminddb
```

Normal operation has no pip dependency. `tshark` is required for PCAPNG and is
recommended for full Wireshark dissection and HTTP object export. VulnMD also includes a
standard-library, single-pass parser for classic Ethernet/Linux-cooked PCAP,
so large classic captures can be processed without rereading the file once per
machine.

The MaxMind database is not bundled. Place your Country database at one of:

- `builder/GeoLite2-Country.mmdb`
- `builder/maxmind/GeoLite2-Country.mmdb`
- `vuln_db/network/GeoLite2-Country.mmdb`
- a standard Debian GeoIP path

You can instead pass `--maxmind-db PATH` or set `VULNMD_MAXMIND_DB`.
`mmdb-bin` is optional and supplies `mmdblookup` when `python3-maxminddb` is
unavailable.

## Complete v1.6 workflow

```bash
# 1. Expand CherryTree into the canonical Markdown database.
python3 main.py build -i Investigation01.ctd -o vuln_db

# 2. Fetch already-resolved vulnerable/fixed upstream Git revisions.
python3 main.py fetch-sources vuln_db --no-resolve --resume

# 3. Generate deterministic source comparisons without Gemini.
python3 main.py analyse-sources vuln_db --no-gemini --resume

# 4. Optionally add one Gemini summary per advisory.
python3 main.py analyse-sources vuln_db --with-gemini --resume \
  --model gemini-3.1-pro-preview \
  --gemini-max-requests 1 \
  --gemini-429-retries 0 \
  --gemini-timeout 1800

# 5. Refresh source/fetch status.
python3 main.py write-status vuln_db

# 6. Stage and analyze one background LAN capture.
python3 main.py import-pcap x.pcap --output builder/pcap
python3 main.py analyse-pcap vuln_db --output builder/pcap

# 7. Optionally split a separate package/update capture and extract HTTP evidence.
python3 main.py extract-pcap packages.pcap --output builder/pcap

# 8. Export the portable Maltego network/vulnerability graph.
python3 main.py export-maltego vuln_db --output builder

# 9. Build and publish the optional static website.
python3 main.py build-site vuln_db --output builder/www
sudo rsync -a --delete --links builder/www/ /var/www/html/
```

JSON import remains supported:

```bash
python3 main.py build -i database.json -o vuln_db
```

To convert CTD without building:

```bash
python3 main.py extract-ctd Investigation01.ctd builder/imported_from_ctd.json
```

## PCAP source and analysis stages

### Stage the source

```bash
python3 main.py import-pcap x.pcap --output builder/pcap
```

`source-pcap` is an alias for this command. The stage records the source format,
size, SHA-256, and local artifact path in `builder/pcap/source.json`. It uses a
same-filesystem hardlink when possible and otherwise copies the capture.
Use `--copy-source` when a separate physical copy is required.

### Correlate with the Markdown database

```bash
python3 main.py analyse-pcap vuln_db \
  --output builder/pcap \
  --maxmind-db builder/GeoLite2-Country.mmdb
```

`analyze-pcap` is an alias. Parser choices are:

```bash
python3 main.py analyse-pcap vuln_db --output builder/pcap --parser auto
python3 main.py analyse-pcap vuln_db --output builder/pcap --parser native
python3 main.py analyse-pcap vuln_db --output builder/pcap --parser tshark
```

- `auto` uses the one-pass native backend for supported classic PCAP and
  TShark for PCAPNG.
- `native` reads classic Ethernet, raw-IP, Linux SLL, and Linux SLL2 PCAP.
- `tshark` provides broader link/protocol support and richer DNS/TLS/HTTP
  dissection.

For v1.5 compatibility, the earlier all-in-one form still works:

```bash
python3 main.py import-pcap x.pcap --repository vuln_db
```

### Identity rules

For every `vuln_db/systems/<machine>.md`, analysis reads:

- literal IPv4/IPv6 values from `Nmap scan report for ...` and `Host:` lines;
- Nmap `MAC Address:` values;
- IP/address/MAC values in the machine metadata table;
- open/filtered Nmap port rows and associated NSE details for Maltego entities.

The root machine-node content is checked first. A Nmap scan in the first direct
child is also accepted. A masked address such as `192.168.2.XXX` cannot be
matched.

Packets are assigned only to declared identities. Unknown private/non-global
addresses appear in the unmatched-observations table and JSON, but do not
become system pages, captures, or future transform entities.

### Endpoint statistics

Each machine's external endpoints are listed first, ordered by total traffic.
LAN endpoints follow, also ordered by traffic. Reports include:

- sent/received packets and bytes;
- first and last timestamps;
- protocols and observed remote TCP/UDP/SCTP ports;
- DNS names, HTTP Host values, and TLS SNI when visible;
- links between known LAN endpoints and their machine pages;
- `https://shodan.io/host/<external-ip>` links for global addresses;
- countries resolved only from the configured local MaxMind database.

The Markdown/HTML table displays 500 endpoints by default while JSON retains
all endpoints:

```bash
python3 main.py analyse-pcap vuln_db --output builder/pcap --endpoint-limit 1000
python3 main.py analyse-pcap vuln_db --output builder/pcap --endpoint-limit 0
```

Use `--no-machine-pcaps` to calculate statistics without retaining filtered
captures.

### PCAP outputs

- `builder/pcap/source.json` — source evidence manifest and SHA-256.
- `builder/pcap/source/` — staged source capture.
- `builder/pcap/import.json` — completed correlation manifest.
- `builder/pcap/machines/` — full per-machine endpoint JSON.
- `builder/pcap/pcaps/` — filtered machine captures.
- `vuln_db/network/README.md` — Markdown capture/network index.
- `vuln_db/network/machines/` — website/database endpoint JSON.
- `vuln_db/network/pcaps/` — efficient hardlinks or synchronized files for
  Markdown and website downloads.

PCAPs can contain payloads, credentials, cookies, hostnames, or personal data.
Restrict Apache access to `/network/` as appropriate.

The release ZIP includes the generated per-machine sample captures and their
statistics but omits the main staged source PCAP, which was supplied
separately. Import your own source capture before rerunning analysis.

## Maltego export

Run the v1.6 exporter after building `vuln_db` and, when desired, after PCAP
analysis:

```bash
python3 main.py export-maltego vuln_db --output builder
```

Outputs are:

- `builder/Investigation01-v1.6.mtgx` — portable graph to open in Maltego;
- `builder/Investigation01-v1.6.graphml` — transparent raw graph;
- `builder/Investigation01-v1.6.json` — counts, palette, identity policy, and
  export audit information;
- `vuln_db/maltego/exports/` — website-compatible copies;
- `vuln_db/maltego/README.md` — analyst-facing graph summary.

The supplied Investigation01 reference contains 19 device entities beneath
`192.168.122.0/24` and `192.168.2.0/24`. The bundled
`vuln_db/maltego/reference.json` reconciles those devices with the updated
CherryTree: matching literal IPs are merged, while seven reference-only
devices remain declared graph hosts. This does not change PCAP identity
policy—unknown LAN addresses observed only in a capture are never promoted.

Open the MTGX with **File → Open** (or double-click it); do not use Maltego's
configuration-package importer. v1.6.1 writes Maltego-native entity, position,
link, and renderer records so Graph Desktop registers its required `Main`
view. The raw GraphML remains available as a transparent recovery/import
format.

Each declared machine uses its IPv4 address as the entity value and retains
the machine name and full Nmap scan in its properties. Every port stores:

- port number and transport;
- Nmap state, service, and version;
- the complete retained Nmap row and adjacent NSE result in `nmap.result`;
- the owning machine name and IP.

Advisories link machines to CVEs and include local Markdown, deterministic
analysis, Gemini report, source-target manifest, and published-site paths.
CVEs use Maltego's `cvssRatingColor` overlay:

| Severity | Overlay |
|---|---|
| Critical | `#f51c24` red |
| High | `#ff7f27` orange |
| Medium | `#ffe100` yellow |
| Low | `#78d663` green |
| Unknown | `#7f7f7f` grey |

Standard IPv4/CVE entities remain ready for installed Shodan, IPinfo, and NVD
transforms. VulnMD does not bundle transform credentials or query those
services during export. Captured external endpoints include Shodan and IPinfo
URLs plus MaxMind country data when PCAP analysis had a local database.

External PCAP endpoints are ordered by bytes, then packets. The busiest 100
per machine are exported by default:

```bash
python3 main.py export-maltego vuln_db --output builder --endpoint-limit 500
python3 main.py export-maltego vuln_db --output builder --endpoint-limit 0
python3 main.py export-maltego vuln_db --output builder --no-endpoints
```

Use `--site-base-url https://example.invalid/vulnmd` to add published page URLs
to machine, advisory, and CVE properties.

MTGL is Maltego's Lucene-backed editable case database. VulnMD retains the
supplied MTGL as reference evidence and exports the supported portable MTGX
graph exchange rather than pretending to rewrite MTGL internals.

## Separate package/update capture extraction

Use a capture that contains package-manager/update traffic:

```bash
python3 main.py extract-pcap x.pcap --output builder/pcap
```

The command uses `vuln_db` by default. An alternate generated repository can be
selected with `--repository PATH`. It applies the same strict Nmap identity
rules as background analysis and never promotes capture-only hosts.

For supported classic PCAP, the native backend streams the input once while
writing the second per-machine capture set. For PCAPNG or wider HTTP
reassembly, use TShark:

```bash
python3 main.py extract-pcap x.pcap \
  --output builder/pcap \
  --repository vuln_db \
  --parser tshark
```

When TShark is available, clear-text HTTP response objects are exported by
default. Use `--no-http-objects` to retain only split PCAPs and HTTP metadata.
To bound JSON memory/output for a very large capture, only 5,000 unique
non-package HTTP requests are retained per machine by default; package
evidence continues to be collected. Use `--http-limit 0` to retain all unique
requests.

Recognized evidence includes Debian `.deb`, RPM `.rpm`, Alpine `.apk`, Arch
package archives, Windows `.msu`/`.cab`/`.msi`, and common installer/archive
names. Package name/version matches link to the machine's
`vuln_db/source/targets/*.json` records and report whether the observed version
exactly matches a vulnerable or fixed version boundary. Versions that do not
match exactly remain unresolved for analyst review.

The two capture streams do not overwrite one another:

- `builder/pcap/pcaps/` — background traffic PCAPs from `analyse-pcap`.
- `builder/pcap/packages/pcaps/` — package/update PCAPs from `extract-pcap`.
- `builder/pcap/packages/machines/` — HTTP requests, package evidence, and
  source-target correlations.
- `builder/pcap/packages/objects/` — TShark-exported clear-text HTTP objects.
- `vuln_db/network/package_captures/` — canonical Markdown/site evidence.

An observed download can support a patching hypothesis, but does not establish
that the package was installed or that the endpoint is patched. Confirm the
installed package inventory on the endpoint. HTTPS names and bodies remain
opaque unless the traffic is decrypted.

## Source evidence workflow

The source policy remains Git-only for open-source targets. Existing manifests
under `vuln_db/source/targets/` provide the repository and vulnerable/fixed
revisions:

```bash
python3 main.py resolve-git-sources vuln_db --resume \
  --model gemini-3.1-pro-preview
python3 main.py fetch-sources vuln_db --no-resolve --resume
python3 main.py analyse-sources vuln_db --no-gemini --resume
python3 main.py analyse-sources vuln_db --with-gemini --resume
python3 main.py write-status vuln_db
```

Deterministic reports and patches remain valid when Gemini is unavailable or
over quota. Gemini is an optional interpretation layer and failure is
non-fatal unless `--strict-gemini` is requested.

Open-source Windows applications such as OpenVPN remain Git-backed.
Microsoft/closed-source findings use the WID/KB/CVE, local update-payload, and
Systracer evidence model:

```bash
python3 main.py index-windows-updates vuln_db --updates-dir /path/to/wsus
python3 main.py link-ms-updates vuln_db
python3 main.py diff-windows-snapshots vuln_db \
  --before before.snp \
  --after after.snp \
  --advisory MSRC-CVE-2024-0000 \
  --kb-after KB5030000
```

## Website

```bash
python3 main.py build-site vuln_db --output builder/www
sudo rsync -a --delete --links builder/www/ /var/www/html/
```

The website includes list/search views for systems and advisories, Nmap
details, vulnerability counts, advisory/CVE/KB external links, vulnerable and
fixed source trees, deterministic/Gemini analyses, network parents,
per-machine endpoint tables, Shodan links, countries, separate package/update
evidence panels, source matches, and both downloadable PCAP sets.
The `.mmdb` database is deliberately excluded.

## Release boundary and roadmap

v1.6 implements `export-maltego` from the existing schema v2 network and
vulnerability data:

- `/24` IPv4 and `/64` IPv6 parent groups derived from declared Nmap identities;
- machine IP/MAC identities;
- parsed Nmap port/service/version rows plus retained NSE details;
- machine → advisory → CVE relationships already present in `vuln_db`;
- known LAN endpoints, external endpoints, Shodan URLs, and country fields;
- an explicit `unknown_hosts_promoted: false` invariant.

The later IDS commands remain intentionally outside this release:

```bash
# v1.7/v1.8 — not present in this release
python3 main.py analyse-pcap vuln_db --output builder/pcap --snort
python3 main.py write-status vuln_db
```

The supplied `.mtgl` sample remains a design/reference fixture and is not
rewritten. The v1.6 portable export groups `192.168.122.0/24` and
`192.168.2.0/24` independently, retains the 19 reference devices, populates
Nmap ports, and preserves Shodan, NVD, IPinfo, advisory, and CVE transform
compatibility.

## Main repository layout

- `vuln_db/systems/` — one Markdown page per machine.
- `vuln_db/advisories/` and `vuln_db/cves/` — linked finding records.
- `vuln_db/source/targets/` — editable source/update manifests.
- `vuln_db/source/trees/` — vulnerable and fixed Git trees.
- `vuln_db/source/patches/` and `source/reports/` — deterministic evidence.
- `vuln_db/analysis/` and `vuln_db/gemini/` — human-readable analyses.
- `vuln_db/windows/` — KB/update/Systracer evidence.
- `vuln_db/network/` — endpoint and package/update capture evidence.
- `vuln_db/maltego/` — reference inventory and website-ready graph exports.
- `builder/pcap/` — background capture workspace and machine PCAPs.
- `builder/pcap/packages/` — separate package/update workspace and PCAPs.
- `builder/Investigation01-v1.6.*` — portable Maltego artifacts.
- `builder/www/` — Apache-ready static site.

## Tests

```bash
python3 -m pytest -q
```

The suite covers CTD parsing, advisory/source/Windows linking, deterministic
analysis, website generation, the legacy PCAP command, the v1.5.1 staged
native-PCAP workflow, v1.5.5 package extraction/source matching, Nmap row
retention, MTGX structure/link validation, 19-device reconciliation, CVE
severity colours, and the no-PCAP-host-promotion invariant.
