# Zimbra Spoof Detection

Detect spoofed/phishing emails on Zimbra servers (concepts and deployment can be reused for other mail servers).

## Table of contents
- [Overview](#overview)
- [Features](#features)
- [How it works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Running the detector](#running-the-detector)
- [Integrating with Zimbra](#integrating-with-zimbra)
- [Integrating with other MTA/servers](#integrating-with-other-mtaservers)
- [Testing](#testing)
- [Tuning & Maintenance](#tuning--maintenance)
- [Logging & Alerts](#logging--alerts)
- [Contributing](#contributing)
- [License](#license)
- [Contact](#contact)

## Overview
Zimbra Spoof Detection is a Python-based tool that analyzes incoming messages and flags likely spoofed or forged emails before they reach users' mailboxes. While this repository includes examples for deploying on Zimbra (which uses Postfix as its MTA), the same detection concepts and integration approaches apply to other MTAs such as Postfix, Exim, Sendmail, or mail gateways.

## Features
- Validate SPF, DKIM and DMARC results for incoming messages
- Header inspection and heuristic checks for common spoof patterns
- Pluggable actions: tag subject/header, quarantine, drop, or generate alerts
- Lightweight Python implementation that can run as a service, milter, or content-filter
- Extensible rules and thresholds via configuration
- Logging and metrics hooks for monitoring/alerting

## How it works
The detector examines each incoming message using a layered approach:
1. DNS-based checks: SPF (policy), DKIM signature verification, and DMARC alignment.
2. Header analysis: From/Reply-To mismatches, suspicious "Received" chains, forged display names vs. envelope-from.
3. Heuristics: Presence of obfuscated links, look-alike domains, suspicious subject/body patterns.
4. Scoring & decision: Each check contributes to a score. Based on configurable thresholds, the detector takes an action (tag/quarantine/drop/alert).

Common libraries used:
- `dnspython` for DNS lookups
- `dkimpy` or similar for DKIM verification
- `pyspf` (or a SPF-check implementation)
- `mail-parser` / `email` for parsing message structure

## Prerequisites
- Python 3.8+ (preferably 3.10+)
- pip
- Access to the Zimbra server shell with privileges to install services and adjust MTA configuration
- Open inbound connection from Zimbra Postfix to the detector service if running remotely (e.g., 127.0.0.1:10025)

## Installation
1. Clone the repository:
   git clone https://github.com/saad-zaib/zimbra-spoof-detection.git
   cd zimbra-spoof-detection

2. Create and activate a virtual environment:
   python3 -m venv .venv
   source .venv/bin/activate

3. Install requirements:
   pip install -r requirements.txt

4. Copy the example configuration and edit:
   cp config.example.yml config.yml
   (Edit config.yml to match your environment and policy choices)

## Configuration
A YAML/JSON configuration file controls detector behavior. Example (config.example.yml):

```yaml
mode: "service"            # service | milter | cli
listen_address: "127.0.0.1"
listen_port: 10025
action_on_spoof: "quarantine"  # tag | quarantine | drop | alert
quarantine_folder: "Spam/Spoof"
score_thresholds:
  tag: 20
  quarantine: 50
  drop: 90
logging:
  level: "INFO"
  file: "/var/log/zimbra-spoof-detector.log"
dkim:
  verify: true
spf:
  verify: true
dmarc:
  verify: true
```

Key options:
- mode: how the script receives mail (as a service expecting SMTP via content_filter, a milter, or CLI for batch checks)
- listen_address/port: where the service listens
- action_on_spoof: default action
- score_thresholds: control sensitivity

## Running the detector
- As a simple command (manual/CLI mode):
  python detector.py --config config.yml --stdin

  (This mode reads a message from stdin and outputs a result. Useful for testing or hooking into other tools.)

- As a long-running service (recommended for production):
  python detector_service.py --config config.yml

  For production, create a systemd unit. Example unit file `/etc/systemd/system/zimbra-spoof-detector.service`:

```ini
[Unit]
Description=Zimbra Spoof Detector
After=network.target

[Service]
User=mailfilter
Group=mail
WorkingDirectory=/opt/zimbra-spoof-detection
ExecStart=/opt/zimbra-spoof-detection/.venv/bin/python /opt/zimbra-spoof-detection/detector_service.py --config /opt/zimbra-spoof-detection/config.yml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable & start:
sudo systemctl daemon-reload
sudo systemctl enable --now zimbra-spoof-detector

## Integrating with Zimbra
Zimbra's MTA is Postfix. The detector can be integrated in multiple ways — choose the approach that fits your operational model:

Option A — Postfix content_filter (recommended if you prefer an external content filter):
1. Configure detector to listen on TCP (127.0.0.1:10025).
2. Edit Postfix to route incoming mail through the content filter (main.cf):
   content_filter = smtp:[127.0.0.1:10025]

   In Zimbra, you may need to update Postfix configuration via the Zimbra mechanisms or directly update Postfix config under `/opt/zimbra/conf` and then restart Zimbra MTA. Common Zimbra restart:
   sudo -u zimbra zmcontrol restart

3. Ensure the content filter returns mail back to Postfix after processing (for example, on port 10026 or as configured).

Option B — Milter (low-latency integration):
- Implement or run the detector as a milter (Sendmail/Postfix milter interface). Python can act as a milter using a library like `pymilter`.
- Configure Postfix/Zimbra to use `smtpd_milters` or `non_smtpd_milters` pointing to the milter socket.
- Restart Zimbra services.

Option C — Amavis / existing content filter hooks:
- Insert detection logic as an additional policy step within your existing content-filter/MTA chain (e.g., amavis).
- This requires adjusting the message flow to call the detector and act on its verdict.

Important notes for Zimbra:
- Back up Zimbra configuration before any modifications.
- Always test changes in a staging environment if possible.
- After changing Postfix settings in a Zimbra appliance, use `su - zimbra -c "zmcontrol restart"` to apply changes.

## Integrating with other MTA/servers
- Postfix (standalone): Use content_filter or milter as above.
- Exim: Use system filter or transport that calls the detector.
- Sendmail: Use milter.
- Exchange / Office 365: Use mail gateway or transport agent on the edge/mail gateway that runs the detector — integration will vary by platform.

## Testing
- Use `swaks` or similar to send test messages through your MTA to the detector path and verify behavior:
  swaks --to user@example.com --from attacker@spoofed.com --server your.mail.server

- Create test messages with intentionally bad DKIM/SPF/DMARC to confirm detection.

- Use unit tests (if provided) and add synthetic samples to tune thresholds.

## Tuning & Maintenance
- Review false positives/negatives and adjust `score_thresholds`.
- Keep DNS/DKIM verification libraries up-to-date.
- Monitor performance: DNS and DKIM verifications can add latency; caching DNS results and parallel verification can help.
- Maintain allow-lists for trusted sending sources (e.g., partner domains using forwarding where SPF may fail but you trust them).

## Logging & Alerts
- Configure logging in `config.yml`.
- Integrate logs with your central logging/alerting (Syslog, ELK, Prometheus + Alertmanager).
- Consider sending high-confidence spoof detections to a quarantine mailbox for manual inspection.

## Extending detection
- Add ML/ML-assisted models for content analysis (be mindful of privacy and compute).
- Integrate reputation feeds / threat intelligence.
- Add automated takedown / reporting workflows for confirmed phishing domains.

## Contributing
Contributions welcome. Please:
1. Open issues to discuss major changes.
2. Submit PRs against the main branch.
3. Include tests and documentation for non-trivial changes.

## License
Specify your license here (e.g., MIT). If none provided, add one to the repository.

## Contact
For questions or help deploying this on your Zimbra servers, open an issue or contact the maintainer.
