# firewall-config-conversion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a skill that converts a firewall config from one vendor to another via the shared `parsing-*` intermediate schema (any source → any of 4 targets), emitting the target's native config plus a per-section fidelity report.

**Architecture:** A documentation skill (Markdown) under `skills/firewall-config-conversion/`. A lean SKILL.md holds input routing, the conversion workflow, the fidelity-report template, and output conventions; a `feature-mapping.md` carries the cross-vendor equivalence + non-isomorphic catalog; four `emit-<vendor>.md` files carry per-target emission patterns (only the chosen target's file + the mapping load per conversion — progressive disclosure). Input is the intermediate schema; raw config delegates to the matching `parsing-*` skill first.

**Tech Stack:** Markdown + YAML frontmatter. Verification is structural (frontmatter, reference-pointer resolution, section-coverage greps — same as the audit skills) plus a controller-run live vSRX commit-*check* of an emitted SRX config. No pytest.

## Global Constraints

- New skill dir/name `firewall-config-conversion`; frontmatter `name` == dir; description starts "Use when", ≤1024 chars; `author: [fastrevmd-lab, Claude]`; `license: MIT`; `version: 1.0.0`.
- Self-contained: reference only files inside its own `references/`. Progressive disclosure: lean SKILL.md; heavy emission detail in `references/`.
- Input model: operate on the intermediate schema; raw config → run the matching `parsing-*` skill first; never re-implement parsing.
- Full-config scope: emitters cover ALL emittable schema sections — objects/groups (address, service, application), security policies, NAT, zones, interfaces, static/OSPF/BGP routing + virtual-routers, system, admin users, DHCP, HA, VPN tunnels, screens, schedules, security_services.
- Output = target's native CLI/set format (SRX `set`; Cisco ASA CLI; FortiGate `config/edit/set/next/end`; Palo PAN-OS `set`).
- **Secrets are NEVER emitted** — PSKs/keys/passwords become placeholders + a `manual-not-converted` caveat.
- Fidelity report is mandatory: every section tagged `converted` / `converted-with-caveats` / `manual-not-converted` with specifics; inline `# CAVEAT:` comments in the emitted config. Output is a migration DRAFT — never claim production-ready.
- The canonical schema (section names) is `skills/parsing-srx-configs/references/intermediate-schema.md`; the vendor syntax anchors are `skills/parsing-<vendor>/references/config-format.md`.
- Re-sync the new skill into `~/.claude/skills/` at the end.

---

### Task 1: Skill skeleton (SKILL.md)

**Files:**
- Create: `skills/firewall-config-conversion/SKILL.md`

**Interfaces:**
- Produces: body sections `Overview`, `When to Use`, `Input Handling`, `Conversion Workflow`, `Output & Fidelity Report`, `Reference Material`, `Common Pitfalls`, `Verification Checklist`. Names `references/feature-mapping.md` and `references/emit-<srx|palo|fortinet|cisco>.md` and `references/example-conversion.md` (created in Tasks 2–7). Defines the exact fidelity-report + output templates that Tasks 3–7 reuse.

- [ ] **Step 1: Write SKILL.md frontmatter exactly**

```yaml
---
name: firewall-config-conversion
description: Use when converting or migrating a firewall / NGFW configuration from one vendor to another (Cisco ASA/FTD, FortiGate, Palo Alto PAN-OS, Juniper SRX) — translating objects, security policies, NAT, zones, interfaces, routing, system, HA, and VPN from a parsed config into a target vendor's native config. Operates on the parsing-* intermediate schema; for raw config, run the matching parsing-cisco/fortinet/palo/srx skill first. Emits the target's native CLI plus a per-section fidelity report (converted / converted-with-caveats / manual-not-converted) flagging everything that does not translate cleanly. Output is a migration draft requiring review, never production-ready; secrets are never emitted.
version: 1.0.0
author:
  - fastrevmd-lab
  - Claude
license: MIT
metadata:
  hermes:
    tags: [firewall, ngfw, conversion, migration, cross-vendor, cisco, fortigate, panos, srx, intermediate-schema, fidelity-report, vendor-neutral]
    related_skills: [parsing-cisco-configs, parsing-fortinet-configs, parsing-palo-configs, parsing-srx-configs, firewall-config-diff, firewall-best-practices-audit]
---
```

- [ ] **Step 2: Write the body sections** (real prose, no placeholders):
  - `Overview` — schema-pivot conversion (any source → any of 4 targets); output is a migration draft + fidelity report; never claims production-ready; secrets never emitted.
  - `When to Use` — "convert/migrate X config to Y", vendor refresh, M&A consolidation, EOL migration. When NOT: auditing (→ firewall-best-practices-audit), comparing/diffing (→ firewall-config-diff), a target without an emitter.
  - `Input Handling` — intermediate schema → emit directly; raw config → run the matching `parsing-*` skill first; user picks the target vendor; never re-implement parsing.
  - `Conversion Workflow` — 1. confirm schema + source vendor + target. 2. load `references/feature-mapping.md` + the target's `references/emit-<target>.md`. 3. emit each schema section in target syntax, inserting `# CAVEAT:` inline where lossy. 4. classify each section for the fidelity report. 5. placeholder all secrets. 6. output config then the fidelity report.
  - `Reference Material` — point to feature-mapping.md, the 4 emit-*.md, example-conversion.md.
  - `Common Pitfalls` and `Verification Checklist` — from Steps 4–5.

- [ ] **Step 3: Add the exact Output & Fidelity Report templates** (Tasks 3–7 reuse these verbatim):

````markdown
## Output & Fidelity Report

### Conversion output (emit in this order)

```text
# Conversion DRAFT: <source-vendor> -> <target-vendor>
# Review required. Not production-ready. Manual items listed in the fidelity report.
<target-native config, with inline "# CAVEAT: ..." comments where lossy/approximate>
```

### Fidelity report (after the config)

```text
Fidelity report (<source> -> <target>)
Section: address_objects     → <converted|converted-with-caveats|manual-not-converted> (<n/m or note>)
Section: security_policies   → ...
Section: nat_rules           → ...
Section: interfaces          → ...
Section: routing             → ...
Section: vpn_tunnels         → ...
Section: system / admin      → ...
Manual items (must do on target):
  1. <e.g. re-key all VPN PSKs — secrets are not emitted>
  2. ...
```
````

- [ ] **Step 4: Fill Common Pitfalls** (prose bullets):
```
- Claiming the converted config is production-ready — it is a draft; require review + the manual items.
- Emitting secrets — PSKs/keys/passwords are always placeholders + a manual caveat.
- Silently dropping a section with no target equivalent — report it as manual-not-converted.
- Re-implementing parsing instead of delegating to the matching parsing-* skill.
- Forcing a 1:1 mapping where the target model differs (zones vs security-levels, App-ID vs port services, routing-instances vs contexts/vsys) — emit the closest form + a CAVEAT.
- Converting interface/routing/system verbatim — these are platform-bound; remap and flag (naming, protocol semantics).
- Losing rule/NAT order — preserve order in the emitted config.
```

- [ ] **Step 5: Fill Verification Checklist**:
```
- [ ] Confirm input is the intermediate schema (parse raw first) and the target vendor is chosen.
- [ ] Every emittable schema section is either converted or explicitly reported (no silent drops).
- [ ] Inline CAVEAT comments mark every lossy/approximate translation.
- [ ] No secrets in the output (PSKs/keys/passwords placeholdered).
- [ ] Rule and NAT order preserved.
- [ ] Output labeled a migration draft; fidelity report lists all manual items.
```

- [ ] **Step 6: Verify frontmatter loadable + valid**

Run:
```bash
cd /home/mharman/fwskillsshare && python3 - <<'EOF'
import re,yaml
from pathlib import Path
p=Path("skills/firewall-config-conversion/SKILL.md"); t=p.read_text()
assert t.startswith("---"); fm=yaml.safe_load(t[3:t.find("\n---",3)])
assert fm["name"]=="firewall-config-conversion"
d=re.sub(r'\s+',' ',re.search(r'^description:\s*(.*?)(?=^\w[\w-]*:|\Z)',t[3:t.find(chr(10)+"---",3)],re.M|re.S).group(1)).strip()
assert 0<len(d)<=1024, len(d)
assert isinstance(fm["author"],list) and len(fm["author"])==2 and fm["license"]=="MIT" and fm["version"]=="1.0.0"
print("OK frontmatter; desc",len(d),"chars")
EOF
```
Expected: `OK frontmatter; desc <N> chars`, N ≤ 1024.

- [ ] **Step 7: Commit**

```bash
cd /home/mharman/fwskillsshare
git add skills/firewall-config-conversion/SKILL.md
git commit -m "feat: scaffold firewall-config-conversion skill body"
```

---

### Task 2: Cross-vendor feature-mapping reference

**Files:**
- Create: `skills/firewall-config-conversion/references/feature-mapping.md`

**Interfaces:**
- Consumes: schema section names (canonical schema).
- Produces: the equivalence + non-isomorphic knowledge the emit-*.md files cite for caveats/classification.

- [ ] **Step 1: Write `feature-mapping.md`** with a title/blurb and two parts:
  1. **Canonical application → target name table.** Reuse the parsers' canonical app list (read the "Cross-Vendor Application Name Mapping" / predefined-app table in `skills/parsing-srx-configs/references/intermediate-schema.md` and `skills/parsing-srx-configs/SKILL.md`) and give, per canonical app (https, http, ssh, dns, etc.), the target form for each vendor: SRX `junos-*`, Palo App-ID/`service-*`, FortiGate service name, Cisco port-based service (`tcp/N`). Note Cisco/FortiGate are port-based (no App-ID) → application identity is approximated.
  2. **Per-section non-isomorphic catalog.** For each schema concept, record maps-1:1 / maps-with-loss / no-equivalent-manual across the 4 vendors: zones (SRX/Palo native; FortiGate interface-based; ASA nameif/security-level), nat_rules (source/dest/static models per vendor), security profiles (SRX application-services / Palo security-profile-group / FortiGate UTM / ASA none), vpn_tunnels (IKE/IPsec crypto + PSK = manual), interfaces (naming conventions), routing-instances/VRF (vs vsys/contexts), schedules, screens (SRX screen / Palo zone-protection / FortiGate DoS-policy / ASA limited).

- [ ] **Step 2: Verify coverage (apps + the hard non-isomorphic sections named).**

Run:
```bash
cd /home/mharman/fwskillsshare && python3 - <<'EOF'
from pathlib import Path
t=Path("skills/firewall-config-conversion/references/feature-mapping.md").read_text().lower()
for vendor in ["srx","palo","forti","cisco"]:
    assert vendor in t, f"missing vendor {vendor}"
for concept in ["zone","nat","vpn","application","interface","profile","routing"]:
    assert concept in t, f"missing concept {concept}"
assert len(t.split())>=200, "mapping too thin"
print("OK feature-mapping: vendors + concepts covered")
EOF
```
Expected: `OK feature-mapping: vendors + concepts covered`.

- [ ] **Step 3: Commit**

```bash
cd /home/mharman/fwskillsshare
git add skills/firewall-config-conversion/references/feature-mapping.md
git commit -m "feat: add cross-vendor feature-mapping + non-isomorphic catalog"
```

---

### Tasks 3–6: Per-target emission references

Tasks 3, 4, 5, 6 are identical in shape, one per target vendor. They are independent (different files) and may be implemented in parallel. For each, substitute `<TARGET>` and `<config-format-ref>`:

- Task 3: `<TARGET>` = `srx`, anchor = `skills/parsing-srx-configs/references/config-format.md`
- Task 4: `<TARGET>` = `palo`, anchor = `skills/parsing-palo-configs/references/config-format.md`
- Task 5: `<TARGET>` = `fortinet`, anchor = `skills/parsing-fortinet-configs/references/config-format.md`
- Task 6: `<TARGET>` = `cisco`, anchor = `skills/parsing-cisco-configs/references/config-format.md`

**Files:**
- Create: `skills/firewall-config-conversion/references/emit-<TARGET>.md`

**Interfaces:**
- Consumes: the schema section names; `feature-mapping.md` (Task 2); the Output/Fidelity templates (Task 1).

- [ ] **Step 1: Write `emit-<TARGET>.md`** — a title/blurb plus one subsection per emittable schema section, each showing how to render that section in `<TARGET>`'s native syntax, using the anchor `<config-format-ref>` for correct syntax. Cover ALL of: address_objects, address_groups, service_objects, service_groups, applications + application_groups, zones, security_policies (preserve order; map action/log/profiles), nat_rules (source/dest/static), interfaces, static routes + virtual_routers + OSPF + BGP, system + admin_users + DHCP, ha_config, vpn_tunnels, screens, schedules, security_services. For each section note its fidelity classification for this target and the `# CAVEAT:` to emit when lossy. Secrets → placeholders only. Where a section has no `<TARGET>` equivalent, state `manual-not-converted` and why.

- [ ] **Step 2: Verify section coverage + no secrets.**

Run (substitute `<TARGET>`):
```bash
cd /home/mharman/fwskillsshare && python3 - <<'EOF'
from pathlib import Path
T="<TARGET>"
t=Path(f"skills/firewall-config-conversion/references/emit-{T}.md").read_text().lower()
must=["address","service","zone","security_polic","nat","interface","route","bgp","ospf","vpn","system","screen","schedule"]
missing=[m for m in must if m not in t]
assert not missing, f"emit-{T} missing sections: {missing}"
import re
assert not re.search(r'\$9\$|\$6\$|pre-shared-key (ascii|hexadecimal)-text \S', t), "looks like a real secret"
print(f"OK emit-{T}: all sections covered, no secrets")
EOF
```
Expected: `OK emit-<TARGET>: all sections covered, no secrets`.

- [ ] **Step 3: Commit**

```bash
cd /home/mharman/fwskillsshare
git add skills/firewall-config-conversion/references/emit-<TARGET>.md
git commit -m "feat: add <TARGET> emission patterns for config conversion"
```

---

### Task 7: Worked example (Cisco → SRX), validated live

**Files:**
- Create: `skills/firewall-config-conversion/references/example-conversion.md`
- Read (input, do not modify): `skills/parsing-cisco-configs/references/fixture-expected-output.json`

**Interfaces:**
- Consumes: `emit-srx.md` (Task 3), `feature-mapping.md` (Task 2), the Output/Fidelity templates (Task 1).

- [ ] **Step 1: Read the source fixture**

Run:
```bash
cd /home/mharman/fwskillsshare && sed -n '1,200p' skills/parsing-cisco-configs/references/fixture-expected-output.json
```
This Cisco-ASA-parsed schema is the source; the target is SRX.

- [ ] **Step 2: Write `example-conversion.md`** — a worked Cisco→SRX conversion of that fixture: (a) name source/target, (b) the emitted SRX `set` config for the fixture's real objects/zones/policies/NAT/interfaces (use only what the fixture contains; preserve order; inline CAVEAT comments where lossy), and (c) the full fidelity report using the Task 1 template, classifying each section and listing manual items (e.g. ASA `security-level`/`nameif` → SRX zones; any secrets → placeholder). No invented objects; no secrets.

- [ ] **Step 3: Verify the example structure + grounding.**

Run:
```bash
cd /home/mharman/fwskillsshare && python3 - <<'EOF'
from pathlib import Path
ex=Path("skills/firewall-config-conversion/references/example-conversion.md").read_text()
assert "Conversion DRAFT" in ex and "Fidelity report" in ex, "missing output/report templates"
assert "set security" in ex, "no SRX set output"
assert "fixture-expected-output.json" in ex or "Cisco" in ex, "source not named"
for sec in ["address","polic","nat"]:
    assert sec.lower() in ex.lower(), f"fidelity report missing {sec}"
print("OK example: Cisco->SRX draft + fidelity report present")
EOF
```
Expected: `OK example: Cisco->SRX draft + fidelity report present`.

- [ ] **Step 4: Commit**

```bash
cd /home/mharman/fwskillsshare
git add skills/firewall-config-conversion/references/example-conversion.md
git commit -m "feat: add worked Cisco->SRX conversion example with fidelity report"
```

---

### Task 8: Validation + housekeeping

**Files:**
- Modify: `README.md` (skill table + install/uninstall + verify-grep), `TODO.md` (tick `firewall-config-conversion`)
- Modify: the four `skills/parsing-*/SKILL.md` (optional cross-link in `related_skills`)

- [ ] **Step 1: Full repo validation (frontmatter + reference pointers).**

Run:
```bash
cd /home/mharman/fwskillsshare && python3 - <<'EOF'
import re,yaml
from pathlib import Path
fail=0
for p in sorted(Path("skills").glob("*/SKILL.md")):
    d=p.parent.name; t=p.read_text(); e=t.find("\n---",3); yaml.safe_load(t[3:e]); body=t[e+4:]
    for ref in set(re.findall(r'references/([A-Za-z0-9._-]+)', body)):
        if not (p.parent/"references"/ref).exists(): print("dangling",d,ref); fail+=1
print(("OK" if not fail else f"{fail} FAIL"),"—",len(list(Path('skills').glob('*/SKILL.md'))),"skills")
EOF
```
Expected: `OK — 19 skills`.

- [ ] **Step 2: README skill-table row** (after the `firewall-best-practices-audit` row):
```markdown
| [firewall-config-conversion](skills/firewall-config-conversion/) | Cross-vendor (Cisco/FortiGate/Palo/SRX via parsers) | `convert firewall config`, `migrate`, `ASA to SRX`, `FortiGate to Palo`, `cross-vendor`, `fidelity report` |
```

- [ ] **Step 3:** Add Claude + Hermes install examples and uninstall lines for `firewall-config-conversion` mirroring the existing per-skill blocks, and add it to the README hermes verify-grep alternation. Update the "## Skills Included" count line to **19 skills** and the family count (add "1 cross-vendor conversion skill" / fold into the audit-skill family wording).

- [ ] **Step 4: Tick TODO.** In `TODO.md` change `2. [ ] \`firewall-config-conversion\`` to `2. [x]`.

- [ ] **Step 5: Cross-link** — append `, firewall-config-conversion` inside the `related_skills: [...]` list in each of the four `parsing-*/SKILL.md`.

- [ ] **Step 6: Re-sync into the user skills dir + parity.**

Run:
```bash
cd /home/mharman/fwskillsshare
DEST="$HOME/.claude/skills/firewall-config-conversion"
rm -rf "$DEST" && cp -r skills/firewall-config-conversion "$DEST"
diff -rq skills/firewall-config-conversion "$DEST" && echo "PARITY OK"
```
Expected: `PARITY OK`.

- [ ] **Step 7: Commit**

```bash
cd /home/mharman/fwskillsshare
git add README.md TODO.md skills/parsing-*/SKILL.md
git commit -m "docs: register firewall-config-conversion (README, TODO, cross-links)"
```

---

## Controller-run final validation (not a subagent task)

After Task 8, the controller validates the worked example's emitted SRX config **live and non-destructively** on a TEST vSRX (e.g. `vSRX-test1` / `vSRX-CI-tester` — NOT vSRX-Production) via `rust-junosmcp`: load the emitted `set` config into the candidate and run `commit check` (validation only), then `rollback 0` — confirming the emitted SRX syntax actually parses on a real device. Use `junos_config_diff` to show the candidate delta. No `commit`. Record the result (secrets redacted); optionally add a `docs/skill-tests/` note.

## Notes for the executor

- Author Markdown in full prose at execution time; the plan fixes structure, the section coverage list, the exact templates, and the verification gates.
- Do NOT invent schema fields — the section list comes from `skills/parsing-srx-configs/references/intermediate-schema.md`.
- Keep SKILL.md lean; per-vendor emission detail lives in `references/emit-*.md` (loaded one at a time).
- Output is always a migration draft; never claim production-ready; never emit secrets.
