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


# Confirm VulnMD can see the model.
python3 main.py doctor vuln_db --probe-gemma
```

If the `lms` daemon works on your installation, `lms server start --port 1234`
is an optional command-line alternative. The GUI server does not require the
`lms` daemon to be healthy.

VulnMD sends generation requests to LM Studio's native
`http://127.0.0.1:1234/api/v1/chat` endpoint and uses `/v1/models` only for
model discovery. Configure the server root if your server differs:

```bash
export VULNMD_LMSTUDIO_URL=http://127.0.0.1:1234
export VULNMD_GEMMA_MODEL=google/gemma-4-26b-a4b-qat
```

No cloud key is required. If LM Studio authentication is enabled, set
`VULNMD_LMSTUDIO_API_KEY`.

## Complete workflow

```bash
# Build the Markdown database from CherryTree.
python3 main.py build -i Investigation01.ctd -o vuln_db

# Start one clean pass over every eligible advisory. Previous v1/automatic
# mappings are archived but are not reused. Gemma may refine a partial answer
# across three prompts; Python validates the resulting distinct Git objects.
python3 main.py resolve-git-sources vuln_db --reresolve-all

# If that long pass is interrupted, continue it without --reresolve-all.
python3 main.py resolve-git-sources vuln_db --resume

# Materialize the validated mappings with the original v1 Git fetch/archive
# logic. This step does not invoke LM Studio.
python3 main.py fetch-sources vuln_db --no-resolve --resume

# Deterministic analysis reads those source/trees and does not start AI.
python3 main.py analyse-sources vuln_db --no-gemma --resume

# Analyze one advisory with local Gemma.
python3 main.py analyse-advisory vuln_db DSA-5885-1

# Or enrich every ready advisory, resumably.
python3 main.py analyse-sources vuln_db --with-gemma --resume
python3 main.py write-status vuln_db

# Build and publish the static site.
python3 main.py build-site vuln_db --output builder/www
sudo rsync -a --delete --links builder/www/ /var/www/html/
```

Local reports are written to `vuln_db/gemma/`; reproducible prompt, evidence,
metadata, hashes, model ID, token usage, and timings go to
`vuln_db/source/gemma/`. Generated analysis pages embed the result between
protected `GEMMA:ANALYSIS` markers while keeping research notes.

Useful controls:

```bash
python3 main.py gemma-analyze vuln_db --advisory DSA-5885-1 --force
python3 main.py analyse-sources vuln_db --with-gemma --limit 5 --resume
python3 main.py analyse-advisory vuln_db DSA-5885-1 --lmstudio-url http://127.0.0.1:1234
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
<img width="1677" height="887" alt="Screenshot from 2026-08-02 20-24-00" src="https://github.com/user-attachments/assets/7f902678-55b3-48c6-b2c6-08971174a33f" />

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
