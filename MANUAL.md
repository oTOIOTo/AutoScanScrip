# AutoScanScrip Manual

## Summary

`Scrip` is a Bash automation wrapper around Gobuster that turns manual content discovery into a guided workflow. It helps you:

- Collect consistent scans with a **scan ID** and organized output folders.
- Run directory and file discovery with **profile-based wordlists**.
- Reduce false positives from wildcard/custom 404 behavior.
- Optionally use **seed discovery** from `robots.txt`/`sitemap.xml` and crawler output.
- Triage findings into ready-to-review reports (`2xx`, `3xx`, `401/403`, and "interesting" files).

In short: it is a structured, repeatable way to do Gobuster reconnaissance with less setup friction and better output hygiene.

---

## What it does (high level)

At a high level, the script does six things:

1. **Collects inputs**
   - Target, scan ID, profile (`quick`, `balanced`, `thorough`), auth/proxy/header options.
2. **Builds an output workspace**
   - Creates a timestamped run directory and prefixes outputs with your scan ID.
3. **Runs pre-scan intelligence**
   - Baseline checks for wildcard-like responses.
   - Quick checks (`robots.txt`, `sitemap.xml`, `.git/HEAD`, `.env`, etc.).
   - Optional seed list from `robots.txt` and `sitemap.xml`, plus optional crawler augmentation.
4. **Executes Gobuster scans**
   - Directory scan, optional deep scan into discovered directories, and file extension scan.
5. **Auto-adjusts for rate-limiting/WAF behavior**
   - Optionally reduces thread count / increases timeout when 429/503 rates are high.
6. **Post-processes and summarizes**
   - Splits results by status category and produces summary + "important findings" files.

---

## How it works (workflow details)

### 1) Startup and validation

- Parses CLI options:
  - `--id`, `--target`, `--profile`, `--non-interactive`, `--help`
- Verifies required binaries (`gobuster`, `curl`, `awk`, etc.).
- Detects whether Gobuster supports `--exclude-length`.

### 2) Input normalization

- Sanitizes scan ID for safe filenames.
- Ensures target has a scheme (tries `https://` then `http://` when omitted).
- Applies defaults in non-interactive mode.

### 3) Baseline + noise reduction

- Performs baseline requests to estimate common response size patterns.
- If baseline response sizes are stable, it can use the mode length as an exclusion length (`--exclude-length` or post-filter fallback).
- This helps suppress wildcard/custom-404 noise.

### 4) Seed discovery and optional crawler enrichment

- Pulls candidate paths from:
  - `robots.txt` (`Allow`, `Disallow`, `Sitemap`)
  - `sitemap.xml` (`<loc>` entries)
- Normalizes them into a seed wordlist.
- Optionally augments with URLs from `katana` or `gospider` (if installed).
- Optionally runs a fast seed-based Gobuster pass.

### 5) Wordlist strategy

- Auto-selects directory/file wordlists by profile.
- Searches common Kali/SecLists paths.
- Falls back to generated minimal wordlists when nothing is found.
- Optional menu-driven override in interactive mode.

### 6) Scanning phases

- **Directory scan** (`-f` style trailing slash checks where configured).
- **Deep scan (optional)** over discovered directories up to a user-defined limit.
- **File/extensions scan** using configurable extension list and backup discovery.

### 7) Auto-throttle behavior

When enabled, each raw result file is examined for 429/503 ratios:

- If rate-limit signals are high, the script lowers thread count (floor of 5).
- It may also increase timeout to 15s if currently too low.

### 8) Result splitting and artifacts

Each scan output is parsed and split into:

- `*_ALL.txt`
- `*_2xx.txt`
- `*_3xx.txt`
- `*_401_403.txt`
- `*_OTHER.txt`

Additional outputs include:

- `*_SUMMARY.txt` (line counts per artifact)
- `*_IMPORTANT_FINDINGS.txt` (aggregated high-priority URLs)
- `*_interesting_files.txt` (sensitive/interesting file patterns)
- command/meta logs for reproducibility

---

## Runtime diagram

```text
┌───────────────────────────────┐
│ User args / interactive input │
└───────────────┬───────────────┘
                │
                v
┌───────────────────────────────┐
│ Dependency + target checks    │
└───────────────┬───────────────┘
                │
                v
┌───────────────────────────────┐
│ Baseline + quick checks       │
│ (wildcard/noise estimation)   │
└───────────────┬───────────────┘
                │
                v
┌───────────────────────────────┐
│ Seed discovery                │
│ robots/sitemap (+ crawler)    │
└───────────────┬───────────────┘
                │
                v
┌───────────────────────────────┐
│ Wordlist auto-select/override │
└───────────────┬───────────────┘
                │
                v
┌────────────────────────────────────────────────┐
│ Scan phases                                    │
│ 1) Directories  2) Deep (optional)  3) Files   │
└───────────────┬────────────────────────────────┘
                │
                v
┌───────────────────────────────┐
│ Split + classify status codes │
│ (2xx/3xx/401-403/other)       │
└───────────────┬───────────────┘
                │
                v
┌───────────────────────────────┐
│ Summary + important findings  │
└───────────────────────────────┘
```

---

## Typical usage

```bash
# Interactive
./Scrip

# Non-interactive
./Scrip --id ACME-01 --target https://example.com --profile balanced --non-interactive
```

## Output layout (example)

```text
gobuster_run_ACME-01_20260306_101500_example.com/
├── ACME-01_SUMMARY.txt
├── ACME-01_IMPORTANT_FINDINGS.txt
├── ACME-01_commands.log
├── ACME-01_meta.log
├── 00_baseline/
├── 00_crawl/
├── 01_dirs/
├── 02_files/
└── deep/
```

## Review order recommendation

1. `*_401_403.txt`
2. `*_IMPORTANT_FINDINGS.txt`
3. `*_interesting_files.txt`
4. `*_2xx.txt`
5. `*_3xx.txt`

