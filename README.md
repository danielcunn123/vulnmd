# VulnMD

VulnMD expands a CherryTree investigation database into a small, inspectable
Markdown vulnerability database. Each managed machine receives a page linking
its imported findings to advisory, CVE, source-tree, deterministic-diff,
Windows-update, and optional Gemini evidence. The same repository can be
published as a static Apache website.

`export-maltego` writes an importable MTGX graph, raw GraphML, and an audit
manifest. The graph uses standard Netblock, IPv4 Address, Port, and CVE
entities for transform compatibility. Network parents contain declared
devices, machines link to Nmap ports and advisories, and advisories link to
CVEs. Critical CVEs are red, high CVEs orange, and medium CVEs yellow.

## Debian installation

```bash
sudo apt update
sudo apt install -y \
  python3 python3-pytest git ca-certificates jq ripgrep unzip zip rsync \
  apache2 tshark python3-maxminddb
```


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
  --model gemini-3.1-pro-preview

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

### Stage the PCAP source

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

## Maltego export

Run the v1.6 exporter after building `vuln_db` and, when desired, after PCAP
analysis:

```bash
python3 main.py export-maltego vuln_db --output builder
```

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

The later IDS commands remain intentionally outside this release:

```bash
# v1.7/v1.8 — not present in this release
python3 main.py analyse-pcap vuln_db --output builder/pcap --snort
python3 main.py write-status vuln_db
```
