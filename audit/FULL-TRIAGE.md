# StackAlign — Full Triage Report (consolidated)

Document type: Consolidated triage of the Phase 1–5 security review + K4 follow-up
Target: StackAlign (Bash/LEMP plan-first CLI)
Report date: 2026-11-22
Triage status: Complete for static/source review; incomplete for runtime validation

## 1. Executive summary

No candidate passed the 7-Question Gate as an exploitable high-severity issue under source inspection.  
One thread remains open: incomplete migration of call sites to the allowlisted executor (`sa_exec`), marked CHAIN-REQUIRED.

**Verdict: NO CONFIRMED FINDINGS / 1 OPEN THREAD (K4) / RUNTIME UNTESTED**

## 2. Ranked attack surface

1. Privileged apply paths (install, repo, sftp/sshd, nft, secret gen) — highest
2. Network / fetch / agent
3. Config / render / template / TOML
4. CLI dispatcher / history
5. DB client helpers
6. Plan-only surfaces — lowest

## 3. Killed / downgraded candidates

- K1 Remote agent diagnostic leakage — Killed (intentional + gated)
- K2 HIBP availability — Downgraded (operational, fail-closed)
- K3 mktemp under attacker TMPDIR — Killed (theoretical)
- K4 Incomplete sa_exec migration — **OPEN / CHAIN-REQUIRED**
- K5 Process-substitution cmp — Killed (already fixed)
- K6 Symlink / traversal in LEMP roots — Killed
- K7 SFTP sshd drop-in injection — Killed

## 4. K4 enumeration (remaining non-sa_exec sites of interest)

| Site | File:line | What runs | Attacker-controlled input? | Reachable under root apply? |
|------|-----------|-----------|----------------------------|-----------------------------|
| Bootstrap curl | install.sh:89 | bare curl (HTTPS + SHA-256) | No (constants) | Yes (installer) |
| Config ownership preserve | lib/config.sh:180 | bare chown/chmod/stat | Path = caller absolute path; content = stdin | Yes if caller allows |
| History undo execution | lib/history.sh:150 | bash -p on generated undo.sh (%q quoted) | Snapshot path from caller | Yes under root rollback |
| Egress DNS | lib/egress.sh:147 | bare getent | DNS names from plan args | Yes (resolution only) |
| Internet doctor | lib/internet.sh:234 | bare getent (hardcoded) | No | No impact |
| Agent JSON | lib/agent.sh:68,82 | python3 -c (fixed snippets) | Text via stdin only | Planning-only |

Core mutators (dnf/rpm/systemctl/useradd/nft primary paths/openssl/argon2/etc.) correctly use sa_exec.

No row currently yields a concrete “attacker-controlled value → non-allowlisted dangerous exec under root apply” payload from existing callers.

## 5. Recommendations

P1 Close K4 caller review (config_write_atomic + history_snapshot paths)  
P2 Disposable EL9 runtime pass on --apply surfaces  
P3 Treat current result as static triage, not final clearance

## 6. Bottom line

Defensive design is credible on paper.  
“No validated findings” = none visible without executing the tool.  
Close the residual sa_exec sites and run a live pass before circulating as clearance.
