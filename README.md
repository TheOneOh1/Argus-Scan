# Argus-Scan: Automated Vulnerability Assessment Tool

Argus-Scan is a comprehensive, open-source vulnerability assessment tool designed to automate security scanning for web applications and internal services. It orchestrates industry-standard tools (Nmap, Nikto, Nuclei) alongside custom Python-based checks to provide a 360-degree security view.

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Python](https://img.shields.io/badge/python-3.11+-yellow.svg)
![Docker](https://img.shields.io/badge/docker-ready-blue.svg)

---

## Table of Contents

- [Features](#-features)
- [Project Structure](#-project-structure)
- [Installation](#-installation)
- [Usage](#-usage)
- [Reporting](#-reporting)
- [CI/CD Integration](#-cicd-integration)
- [How It Works](#-how-it-works)
- [Troubleshooting](#-troubleshooting)
- [Future Roadmap](#-future-roadmap)
- [Disclaimer](#-disclaimer)

---

## 🚀 Features

| Feature | Description |
|--------|-------------|
| **Port Scanning** | Identifies open ports and running services using Nmap. Flags risky services (FTP, Telnet, RDP, SMB, MySQL). |
| **Security Headers** | Checks for missing or misconfigured HTTP security headers (HSTS, CSP, X-Frame-Options, X-Content-Type-Options, Referrer-Policy). TLS verification is **on by default**; use `--insecure` only for lab or self-signed targets. |
| **SSL/TLS Inspection** | Validates certificate expiry and connection to port 443. |
| **Vulnerability Scanning** | With `--full`, runs Nikto (web vulnerability scanner) and Nuclei (template-based scanning). |
| **Dashboard Reporting** | Generates an elegant "Intelligence Dossier" HTML report (with dark/light mode toggle), and a detailed Markdown report with severity counts and recommendations. |
| **CI/CD Ready** | GitLab CI configuration for building the image and running scans with reports as artifacts. |

---

## 📂 Project Structure

```text
.
├── src/
│   ├── vapt.py             # Main entrypoint and CLI
│   ├── scanners.py         # Scanner modules (Port, Header, SSL, Tool)
│   ├── reporter.py         # Reporting logic (CLI, HTML, Markdown)
│   └── templates/          # Jinja2 HTML report template
├── reports/                # Generated reports (created if missing)
├── setup.sh                # Setup script for Linux/WSL
├── requirements.txt        # Python dependencies
├── Dockerfile              # Container image definition
├── .dockerignore           # Build context exclusions
└── .gitlab-ci.yml          # GitLab CI pipeline
```

**Run and import expectations**

- **Native run:** Execute from the **repository root** so that `python src/vapt.py` resolves imports correctly (the directory of the script is on `sys.path`).
- **Docker:** The image sets `PYTHONPATH=/app/src` and runs `python src/vapt.py` from `/app`. No extra setup is required.

---

## 🛠️ Installation

### Method 1: Docker (Recommended)

Docker bundles Nmap, Nikto, Nuclei, and Python dependencies. No host install needed.

**Build the image:**

```bash
docker build -t Argus-Scan .
```

**Run a scan:**

```bash
docker run --rm -v $(pwd)/reports:/app/reports Argus-Scan --target example.com
```

### Method 2: Manual Installation (Linux / WSL)

**Prerequisites:** Python 3.8+ (3.11+ recommended), sudo access for system tools.

1. **Run the setup script:**

   ```bash
   chmod +x setup.sh
   ./setup.sh
   ```

   This creates a virtual environment (`venv`) and checks for Nmap, Nikto, and Nuclei. Missing tools are reported; the script does not install them on all systems.

2. **Activate the environment and run:**

   ```bash
   source venv/bin/activate
   python src/vapt.py --target example.com
   ```

   Always run `python src/vapt.py` from the **project root** (the directory that contains `src/` and `requirements.txt`).

**Windows**

- Use **Docker** or **WSL** and follow the Linux instructions. The native setup script is intended for Unix-like environments.

---

## 💻 Usage

### Basic scan

Runs port scan, security headers, and SSL checks. Does **not** run Nikto or Nuclei.

**Docker:**

```bash
docker run --rm -v $(pwd)/reports:/app/reports Argus-Scan --target example.com
```

**Native (from repo root):**

```bash
python src/vapt.py --target example.com
```

### Full scan

Adds Nikto and Nuclei. Takes longer and requires those tools to be installed (or use Docker).

**Docker:**

```bash
docker run --rm -v $(pwd)/reports:/app/reports Argus-Scan --target example.com --full
```

**Native:**

```bash
python src/vapt.py --target example.com --full
```

### All options

| Flag | Short | Description |
|------|-------|-------------|
| `--target` | — | **Required.** URL or hostname to scan (e.g. `https://example.com`, `example.com`, `192.168.1.1`). |
| `--output` | — | Directory for report files (default: `reports`). |
| `--full` | — | Run Nikto and Nuclei in addition to port/header/SSL checks. |
| `--no-tool-check` | — | Skip checks for Nmap/Nikto/Nuclei. Use when you only want Python-based checks or in CI where tools are guaranteed. |
| `--verbose` | `-v` | Print tool commands and live tool output during scans. |
| `--insecure` | — | Disable TLS verification for the security-header HTTP request. Use only for lab or self-signed targets. |

**Examples:**

```bash
# Custom output directory
python src/vapt.py --target example.com --output ./my-reports

# Full scan with verbose tool output
python src/vapt.py --target example.com --full --verbose

# Skip dependency check (e.g. in Docker or CI)
docker run --rm -v $(pwd)/reports:/app/reports Argus-Scan --target example.com --no-tool-check

# Self-signed or internal HTTPS
python src/vapt.py --target https://internal.example.com --insecure
```

---

## 📊 Reporting

Reports are written under the directory given by `--output` (default: `reports/`).

| Type | File pattern | Purpose |
|------|--------------|--------|
| **HTML** | `vapt_report_<target>_<YYYYMMDD_HHMMSS>.html` | "Intelligence Dossier" aesthetic dashboard with dark/light mode toggle. |
| **Markdown** | `vapt_report_<target>_<YYYYMMDD_HHMMSS>.md` | Technical summary and raw module output in collapsible sections. |

Both include:

- Security score (0–100) and grade (A–F)
- Severity counts (CRITICAL, HIGH, MEDIUM, LOW, INFO)
- Per-finding description, current state, and recommendation
- Raw details for headers, ports, and (when run) Nikto/Nuclei output

---

## 🚀 CI/CD Integration

### GitLab CI (self-hosted)

The included `.gitlab-ci.yml` is meant for self-hosted runners with Docker available.

1. **Variables:** In the project’s CI/CD settings, set `TARGET_URL` (e.g. `https://your-app.example.com`). Override or add variables as needed for your pipeline.
2. **Runner:** Use tags (e.g. `self-hosted`, `shell`) so jobs run on machines that can build and run the Argus-Scan image.
3. **Artifacts:** Reports are saved under `reports/` and published as job artifacts (e.g. expire in 1 week).
4. **Branches:** The default pipeline triggers on `main` and `web`; adjust `only` in `.gitlab-ci.yml` to match your workflow.

---

## 🔬 How It Works

1. **Dependency check** (unless `--no-tool-check`): Ensures `nmap`, `nikto`, and `nuclei` are on `PATH`. If any are missing and `--full` is set, the run exits with instructions; otherwise it continues with Python-only checks.
2. **Port scan:** Nmap fast scan (`-F -sV`) on the resolved IP. Open ports are listed; known risky ports (e.g. 21, 23, 3389, 445, 3306) generate findings.
3. **Header scan:** HTTP GET to the target (scheme added if missing). Presence of security headers is checked; missing ones are reported with severity and remediation.
4. **SSL scan:** Connection to port 443; certificate expiry and basic validity are checked.
5. **Full scan (optional):** Runs Nikto and Nuclei with fixed tuning/options. Target and hostname are shell-quoted before use to reduce injection risk.
6. **Reporting:** Aggregates findings, computes score and grade, then writes CLI summary, Markdown, and HTML.

---

## 🔧 Troubleshooting

| Issue | What to do |
|-------|------------|
| **`ModuleNotFoundError` for `reporter` or `scanners`** | Run from the **repository root**: `python src/vapt.py ...`. Do not run from inside `src/` unless you adjust `PYTHONPATH` to include the project root or `src/`. |
| **“Missing external tools”** | Install Nmap/Nikto/Nuclei (see `setup.sh` messages or your distro docs), or run with `--no-tool-check` for Python-only checks. |
| **SSL/certificate errors on header check** | Use `--insecure` only for lab or self-signed targets. In production, fix the certificate or use a valid hostname. |
| **Nikto/Nuclei not run** | Use `--full`. Ensure the tools are on `PATH` or run via the Docker image. |
| **No or empty reports** | Confirm `--output` is writable and that the run completed (no early exit). Check the path reported in the “HTML report saved to” message. |

---

## 🔮 Future Roadmap

- **Deep fuzzing:** Integration with directory brute-forcing tools.
- **Authentication:** Support for cookies or tokens for authenticated scanning.
- **API security:** Dedicated checks for REST/GraphQL endpoints.

---

## ⚠️ Disclaimer

This tool is for **educational and authorized testing only**. Use it only on systems you own or have explicit permission to test. Unauthorized scanning may violate laws or policies.
