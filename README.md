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

No cloud key is required. If LM Studio authentication is enabled, set
`VULNMD_LMSTUDIO_API_KEY`.

## Complete workflow

```bash
# Build the Markdown database from CherryTree.
python3 main.py build -i Investigation01.ctd -o vuln_db

# If that long pass is interrupted, continue it without --reresolve-all.
python3 main.py resolve-git-sources vuln_db --resume

# Materialize the validated mappings with the original v1 Git fetch/archive logic.
python3 main.py fetch-sources vuln_db --no-resolve --resume

# Deterministic analysis reads those source/trees and does not start AI.
python3 main.py analyse-sources vuln_db --no-gemma --resume

# Build and publish the static site.
python3 main.py build-site vuln_db --output builder/www
sudo rsync -a --delete --links builder/www/ /var/www/html/
```

Useful controls:

```bash
python3 main.py analyse-advisory vuln_db DSA-5885-1
python3 main.py analyse-sources vuln_db --with-gemma --limit 5 --resume
python3 main.py analyse-advisory vuln_db DSA-5885-1 --lmstudio-url http://127.0.0.1:1234

# Stage the PCAP source
python3 main.py import-pcap x.pcap --output builder/pcap

# Correlate with the Markdown database
python3 main.py analyse-pcap vuln_db  --output builder/pcap  --maxmind-db builder/GeoLite2-Country.mmdb
```

## Maltego export

Run the v1.6 exporter after building `vuln_db` and, when desired, after PCAP
analysis:

```bash
python3 main.py export-maltego vuln_db --output builder
```

## Website

```bash
python3 main.py build-site vuln_db --output builder/www
sudo rsync -a --delete --links builder/www/ /var/www/html/
```
<img width="1677" height="887" alt="Screenshot from 2026-08-02 20-24-00" src="https://github.com/user-attachments/assets/7f902678-55b3-48c6-b2c6-08971174a33f" />

## Release boundary and roadmap

The later IDS commands remain intentionally outside this release:

```bash
# v1.7/v1.8 — not present in this release
python3 main.py analyse-pcap vuln_db --output builder/pcap --snort
python3 main.py write-status vuln_db
```
