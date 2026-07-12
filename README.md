## VulnMD

Vulnmd is a tool for cybersecurity professionals who wish to obtain source code for vulnerable products.

A sample database from cherrytree is the foundation of this project, to generate a local database containing fixed & vulnerable source code - accross multiple Linux distrobutions. 
* Debian
* Ubuntu
* Kylin Linux
* Astra Linux
* Kali Linux
* BOSS Linux
* RedStar OS
* Parrot OS

The goal of this project is to learn what advisories are, which vulnerabilities exist on commonly used Linux distrobutions.


## 1. Install system packages

```bash
sudo apt update
sudo apt install -y \
  python3 \
  python3-pytest \
  git \
  ca-certificates \
  jq \
  ripgrep \
  unzip \
  zip
```

No pip package, `deb-src`, or `apt-get source` is used.

Choose the Gemini CLI executable when it is called `gemini-cli`:

```bash
export VULNMD_GEMINI_COMMAND=gemini-cli
```


## 2. Build the local Markdown repository

```bash
python3 main.py -i database.json -o vuln_db
```

The included full database produces system, advisory, CVE, package, analysis,
Gemini, and source-target pages.


## 3. Fetch an advisory that does not require Gemini

`DSA-4502-1` is in the evidence-backed local catalog:

```bash
python3 main.py fetch-sources vuln_db \
  --advisory DSA-4502-1 \
  --no-resolve
```

It maps to:

```text
repository: https://github.com/FFmpeg/FFmpeg.git
vulnerable revision: n4.1.3
fixed revision: n4.1.4
```

`DLA-3068-1` is also pre-resolved:

```bash
python3 main.py fetch-sources vuln_db \
  --advisory DLA-3068-1 \
  --no-resolve
```


## 4. Fetch five advisories, Critical first

```bash
python3 main.py fetch-sources vuln_db \
  --limit 5 \
  --resume \
  --model gemini-3.1-pro-preview \
  --resolution-attempts 1 \
  --gemini-min-interval 70 \
  --gemini-max-requests 5 \
  --gemini-429-retries 0 \
  --gemini-timeout 1800
```


## 5. Deterministic comparison

This never invokes Gemini:

```bash
python3 main.py analyse-sources vuln_db \
  --advisory DSA-4502-1 \
  --no-gemini
```

Outputs:

```text
vuln_db/analysis/DSA-4502-1.md
vuln_db/source/reports/DSA-4502-1/
vuln_db/source/patches/DSA-4502-1/
```


## 6. Deterministic comparison plus one Gemini summary

Use a one-request summary budget:

```bash
python3 main.py analyse-sources vuln_db \
  --advisory DSA-4502-1 \
  --with-gemini \
  --model gemini-3.1-pro-preview \
  --force-gemini \
  --gemini-min-interval 70 \
  --gemini-max-requests 1 \
  --gemini-429-retries 0 \
  --gemini-timeout 1800
```
