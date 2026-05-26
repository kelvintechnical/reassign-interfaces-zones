# Lab: Reassigning Interfaces Between firewalld Zones — Runtime vs Permanent

- **Series:** linux-ops-mastery — RHCSA Firewall
- **Subjects covered:** `firewall-cmd --change-interface`, `--permanent --change-interface`, `firewall-cmd --reload`, verifying with `--get-active-zones`, safe service checks before moves, rollback to `public`
- **Career arcs covered:** RHCSA (NIC-to-zone tasks), RHCE (interface parameters + handler notify reload), SRE (maintenance windows on dual-homed hosts), DevOps (immutable config vs live drift), AI/MLOps (isolating data-plane NICs)
- **Prerequisite:** Labs **firewalld-zones** (vocabulary) and **active-firewall-zones** (audit loop)
- **Time Estimate:** 30 to 45 minutes
- **Difficulty arc:** Task 1 snapshot · 2–3 runtime moves · 4 permanent mirror · 5 reload verification · 6 capstone + full revert

---

## Objective

Zone **catalogs** (Lab 56) and **active audits** (Lab 60) mean nothing if you cannot **rewire** NICs. `--change-interface` is the supported verb: it **moves** an interface from whatever zone currently owns it into the zone you name — first at **runtime**, then (with `--permanent` + `--reload`) across **reboot**.

This lab deliberately overlaps the *mechanic* you touched in **default-firewall-zone** — but here the **protagonist** is the interface reassignment workflow end-to-end, including **persistence** and **reload discipline**, not changing the global default (unless your instructor adds a bonus).

> **Lab safety note:** Moving your only SSH NIC into a zone without `ssh` **locks you out**. Always verify `services:` first. Task 6 returns everything to **`public`**.

---

## Concept: Interfaces Are Zone Membership Cards

```
Before:
  public   ← ens160
  dmz      ← (empty)

After: firewall-cmd --zone=dmz --change-interface=ens160

  public   ← (ens160 removed)
  dmz      ← ens160 inherits dmz services/ports
```

Permanent configuration remembers the membership card across **`reload`** and **reboot** once you mirror with `--permanent`.

> **Why this matters:** Dual-homed servers fail open/closed in the wrong direction when the **management** NIC accidentally inherits **`dmz`** rules (or vice versa). Moves must be deliberate and verified.

---

## 📜 Why Interface-to-Zone Binding Matters — The Story

Before zones, operators matched **interface names inside raw iptables rules** — correct, but fragile: renaming NICs (`eth0` → `ens3`) broke scripts silently. `firewalld` elevated interfaces to first-class **binding objects** attached to a zone, so policy followed the NIC identity the kernel reports, not a scattered list of `-i eth0` matches.

Enterprise patterns emerged: **internet-facing NIC** in `dmz` or `public`, **management** NIC in `internal`, **storage replication** NIC sometimes in a custom zone with tight `rich rules`. Red Hat Enterprise Linux has shipped `firewalld` as the default firewall service since **RHEL 7**, refining D-Bus APIs so automation could reassign interfaces without hand-editing XML.

> **The point of the story:** `--change-interface` is how you express physical reality (“this cable is untrusted”) in the same language as the security team’s diagrams.

---

## 👪 The Interface Reassignment Family — Who Lives There

### By persistence

| Step | Command idea |
|---|---|
| Runtime move | `firewall-cmd --zone=TARGET --change-interface=IF` |
| Permanent move | `firewall-cmd --permanent --zone=TARGET --change-interface=IF` |
| Apply saved | `firewall-cmd --reload` |

### By verification

| Question | Command |
|---|---|
| Where did the NIC go? | `firewall-cmd --get-active-zones` |
| What does TARGET allow? | `firewall-cmd --list-all --zone=TARGET` |

### By contrast with Lab 57

| Lab 57 emphasis | Lab 61 emphasis |
|---|---|
| Default zone + NIC move story | **Persistence** of NIC moves + reload proof |
| Combined narrative | `--change-interface` as hero command |

> **The point of the family tree:** Same underlying knob — different learning lens.

---

## 🔬 The Anatomy of `--change-interface` — In One Diagram

```
$ firewall-cmd --zone=dmz --change-interface=ens160
success

$ firewall-cmd --get-active-zones
dmz
  interfaces: ens160
```

Interpretation:

```
--zone=dmz                ← destination club membership
--change-interface=ens160 ← NIC card taken from old club, given to dmz
success                   ← D-Bus accepted request; still verify map
```

> **Reading rule:** `success` without `get-active-zones` is superstition, not engineering.

---

## 📚 Interface Reassignment Reference Table

| Task | Command | Notes |
|---|---|---|
| Runtime move | `firewall-cmd --zone=dmz --change-interface=ens160` | Immediate |
| Permanent move | `firewall-cmd --permanent --zone=dmz --change-interface=ens160` | Writes config |
| Reload | `firewall-cmd --reload` | Align runtime to permanent |
| Undo runtime | `firewall-cmd --zone=public --change-interface=ens160` | Common safe return |
| Undo permanent | Mirror removal pattern in Cleanup | Remove from zone files via CLI |
| List NICs | `ip -br link` | Name resolution |

> **Rule one of moves:** **SSH service** must exist in TARGET before you move the admin NIC.

---

## 🧪 Extended Verification Playbook (Optional Depth)

| Situation | Command | What “good” looks like |
|---|---|---|
| NM zone property | `nmcli -f connection.zone con show --active` | Should align with `firewalld` intent on NM-managed hosts |
| Connection name | `nmcli -t -f DEVICE,CONNECTION dev status` | Maps `IFACE` → profile for documentation |
| D-Bus sanity | `busctl tree org.fedoraproject.FirewallD1 \| head` | Proves daemon object path exists (advanced) |
| Log tail | `journalctl -u firewalld -n 30 --no-pager` | Errors after failed `--change-interface` |
| Secondary NIC drill | Repeat `get-active-zones` after unplugging lab cable | Understand hotplug behavior (optional) |
| Rollback evidence | `diff -u /tmp/before.txt /tmp/after.txt` | If you saved snapshots — professional habit |
| Package drift | `rpm -V firewalld` | Non-zero → config files changed outside package (rare) |
| Documentation | `man firewalld.zone` | Deepens XML field knowledge beyond CLI |

Optional enrichment — revisit after you can perform the six-task spine without notes.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Interface moves + reload are a frequent grader verification path. |
| **RHCE candidate** | Handlers: `notify: reload firewalld` after permanent interface edits. |
| **SRE / Platform** | Maintenance: migrate NIC between zones during rack renumbering. |
| **DevOps** | Cloud images with ens+ names — never hardcode `eth0`. |
| **AI / MLOps** | Data ingestion NIC should not inherit management `internal` blindly. |

---

## 🔧 The 6 Tasks

> Pick **`TARGET=dmz`** or **`TARGET=internal`** per classroom policy — examples use **`dmz`**.

---

### Task 1 — Snapshot current bindings and derive `$IFACE`

**Purpose:** Record reversible baseline before touching membership.

```bash
sudo -i
ip -br link
firewall-cmd --get-active-zones
IFACE=$(ip -br link | awk '!/lo/ && $2=="UP" {print $1; exit}' | sed 's/@.*//')
echo "IFACE=$IFACE"
TARGET=dmz
```

**Human-Readable Breakdown:** `$IFACE` should be the NIC you intend to move — in classrooms, often the primary uplink.

**Reading it left to right:** `TARGET` variable keeps later commands copy-paste friendly.

**The story:** Baseline screenshot prevents “it was always like that” arguments.

**Expected output:**

```text
ens160           UP ...
public
  interfaces: ens160
IFACE=ens160
```

**Switches**

| Token | Meaning |
|---|---|
| `get-active-zones` | Shows current membership |
| `TARGET=dmz` | Editable lab knob |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| empty IFACE | Bring interface up; verify link |

---

### Task 2 — Core A: verify `ssh` (and peers) exist in `$TARGET` before move

**Purpose:** Lockout prevention gate identical to real change windows.

```bash
firewall-cmd --info-zone="$TARGET" | grep -E 'services:|ports:|target:'
```

**Human-Readable Breakdown:** Confirm `ssh` appears — `dmz` stock profile includes `ssh` on RHEL 9 class images.

**Reading it left to right:** If `ssh` missing, **stop** — add service (Lab 58) or use console.

**The story:** Five seconds of `grep` beats a forty-minute KVM session.

**Expected output:**

```text
  target: default
  services: ssh
  ports:
```

**Switches**

| Token | Meaning |
|---|---|
| `--info-zone` | Rich view |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| no ssh | Choose `internal` or add service |

---

### Task 3 — Core B: runtime `--change-interface` into `$TARGET`

**Purpose:** Immediate move — no permanent yet (split pedagogy).

```bash
firewall-cmd --get-active-zones
firewall-cmd --zone="$TARGET" --change-interface="$IFACE"
firewall-cmd --get-active-zones
firewall-cmd --list-all --zone="$TARGET" | head -n 25
```

**Human-Readable Breakdown:** Middle command mutates; following commands prove effect.

**Reading it left to right:** Interface line should disappear from `public` and appear under `dmz`.

**The story:** Runtime-only moves are great for maintenance — still vanish on certain reload patterns if not mirrored.

**Expected output:**

```text
dmz
  interfaces: ens160
```

**Switches**

| Token | Meaning |
|---|---|
| `--change-interface` | Move binding |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| ALREADY_SET | Usually harmless — verify map |
| WRONG_IFACE | Re-derive `$IFACE` |

---

### Task 4 — Permanent mirror + `reload` to prove boot-time story

**Purpose:** Wire the move into `/etc/firewalld` namespace and reconcile.

```bash
firewall-cmd --permanent --zone="$TARGET" --change-interface="$IFACE"
firewall-cmd --reload
firewall-cmd --get-active-zones
firewall-cmd --permanent --get-active-zones 2>/dev/null || true
```

**Human-Readable Breakdown:** Permanent flag writes config; `reload` pushes authoritative permanent state into runtime.

**Reading it left to right:** Third command should still show `$IFACE` under `$TARGET`.

**The story:** This is the **exam-grade** sequence: mutate permanent → reload → verify.

**Expected output:**

```text
success
success
dmz
  interfaces: ens160
```

**Switches**

| Token | Meaning |
|---|---|
| `--permanent` | Durable |
| `--reload` | Apply |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| NIC jumps back | Conflicting NM zone settings — align NetworkManager `connection.zone` if used |

---

### Task 5 — Edge case: compare runtime vs permanent `list-all` for `$TARGET`

**Purpose:** Catch partial writes or drift immediately after reload.

```bash
echo RUNTIME; firewall-cmd --list-all --zone="$TARGET" | egrep 'interfaces|services|ports'
echo PERMANENT; firewall-cmd --permanent --list-all --zone="$TARGET" | egrep 'interfaces|services|ports'
```

**Human-Readable Breakdown:** After healthy reload, pairs should match on these lines.

**Reading it left to right:** Mismatch means troubleshooting before you declare victory.

**The story:** This is the edge case that saves Sunday patching.

**Expected output:**

```text
RUNTIME
  interfaces: ens160
  services: ssh
  ports:
PERMANENT
  interfaces: ens160
  services: ssh
  ports:
```

**Switches**

| Token | Meaning |
|---|---|
| `egrep` | High-signal diff |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| interfaces missing in permanent | Re-run permanent change-interface |

---

### Task 6 — Capstone narration + full revert to `public` (runtime + permanent)

**Purpose:** Demonstrate complete story and **restore teaching baseline**.

```bash
{
  date
  echo "ACTIVE"; firewall-cmd --get-active-zones
} | tee /tmp/iface-zone-lab.txt
```

**Human-Readable Breakdown:** Evidence file shows final bound state before rollback.

**The story:** graders want **return to known-good** — practice it muscled, not hoped.

**Expected output:**

```text
ACTIVE
dmz
  interfaces: ens160
```

**Switches**

| Token | Meaning |
|---|---|
| `tee` | `/tmp` evidence |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| permission | root shell |

**Cleanup**

```bash
firewall-cmd --permanent --zone=public --change-interface="$IFACE"
firewall-cmd --reload
firewall-cmd --get-active-zones
rm -f /tmp/iface-zone-lab.txt
```

**Human note:** If `public` already contained the iface in permanent config historically, this returns to stock; if your image differed, verify `public` shows `$IFACE` and services look classroom-normal.

---

## 🔍 Interface Reassignment Decision Guide

```
Need to move a NIC?
  │
  ├── "Does target zone allow my admin path?"
  │       └── grep ssh in --info-zone=TARGET
  │
  ├── "Immediate test only?"
  │       └── runtime --change-interface
  │
  ├── "Must survive reboot?"
  │       └── permanent --change-interface → reload → verify drift
  │
  ├── "Did NetworkManager also set a zone?"
  │       └── nmcli con show ... connection.zone
  │
  └── "Audit current wiring"
          └── Lab 60 active-firewall-zones
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Capture `ip -br link`, `get-active-zones`, set `$IFACE`/`$TARGET`
- [ ] 02 Verify `ssh` exists in `$TARGET` zone profile
- [ ] 03 Runtime `--change-interface` and re-read active map + `list-all`
- [ ] 04 `--permanent --change-interface` + `reload` + verify active map
- [ ] 05 Compare runtime vs permanent high-signal fields for `$TARGET`
- [ ] 06 Save `/tmp` evidence, then Cleanup move iface back to `public` + reload + delete file

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Move before ssh check | Lockout | Console recover |
| Runtime only | Reboot reverts oddly | Mirror permanent |
| Skip reload after permanent | Drift confusion | Always reload |
| NM zone fights firewalld | Flip-flop bindings | Align NM `zone` property |
| Wrong `$IFACE` | Nothing seems to change | Re-derive from `ip -br` |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Memorize **verify services → runtime move → permanent mirror → reload → triple verify → rollback**.

**RHCE candidate**
- Explain notify ordering: interface task before service tasks or vice versa depending idempotency.

**SRE / Platform interview**
- Discuss dual-homed isolation: never put replication NIC in `trusted` without explicit approval.

**DevOps**
- Parameterize `iface` and `target_zone` as Ansible facts — no literals.

**AI / MLOps**
- Document which NIC carries object storage traffic vs control plane — moves follow architecture.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| [active-firewall-zones](https://github.com/kelvintechnical/active-firewall-zones) | Pre-flight audit |
| [default-firewall-zone](https://github.com/kelvintechnical/default-firewall-zone) | Default zone interactions |
| [firewalld-add-services](https://github.com/kelvintechnical/firewalld-add-services) | Fix missing services before moves |
| [firewalld-zones](https://github.com/kelvintechnical/firewalld-zones) | Trust bundle vocabulary |

---

## 🎓 After the Lab — 60-Second Oral Exam

Answer out loud without scrolling:

- What is the full persistence trio for interface moves?
- Why is `--reload` safer than rebooting to “test” permanent changes during an exam?
- What happens if NetworkManager’s `connection.zone` disagrees with `firewalld`?
- Which command proves the NIC landed under the intended zone header?
- What is the first rollback motion if SSH stops responding mid-lab?
- Why might `--permanent --get-active-zones` be unreliable across versions?
- What NM field sometimes fights a pure `firewalld` binding story?

If any answer wobbles, redo Tasks 2–4 slowly — rehearse the rollback path as often as the forward path.

```text
Persistence spine:  runtime move → permanent mirror → reload → get-active-zones
Rollback spine:     public + change-interface + reload → ssh still answers
```

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
