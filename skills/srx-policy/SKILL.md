---
name: srx-policy
description: Use when designing, migrating, configuring, auditing, or troubleshooting Juniper SRX security policy on Junos 23.x+ non-Branch SRX platforms. Strongly prefers security policies global for greenfield and cross-vendor migrations; covers global versus zone-to-zone policy structure, address/application objects, AppID/AppFW, enhanced web filtering/UTM attachment, SecIntel, ATP integration, logging, policy order, verification, and troubleshooting.
version: 1.0.0
author: Hermes Agent
license: source-derived-summary-local-use
metadata:
  hermes:
    tags: [srx, junos, security-policy, global-policy, zone-policy, appid, appfw, application-firewall, web-filtering, utm, secintel, atp, migration]
    related_skills: [parsing-srx-configs, srx-nat, srx-dynamic-ip-feed, srx-mnha, srx-mpls-in-flow]
  sources:
    - title: Configuring Security Policies | Junos OS
      author: Juniper Networks
      url: https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/topic-map/security-policy-configuration.html
      retrieved: "2026-05-15"
    - title: Global Policy Overview | Junos OS
      author: Juniper Networks
      url: https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/topic-map/security-global-policies.html
      retrieved: "2026-05-15"
    - title: Security Policy Applications and Application Sets | Junos OS
      author: Juniper Networks
      url: https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/topic-map/policy-application-sets-configuration.html
      retrieved: "2026-05-15"
    - title: Configure Application Firewalling On A Juniper SRX
      author: Willem Redelijkheid
      url: https://www.redelijkheid.com/blog/2013/5/10/configure-application-firewalling-on-a-juniper-srx
      retrieved: "2026-05-15"
    - title: Juniper SRX Policy Configuration
      url: https://rayka-co.com/lesson/juniper-srx-policy-configuration/
      retrieved: "2026-05-15"
    - title: Juniper SRX Enhanced Web Filtering Configuration
      url: https://rayka-co.com/lesson/juniper-srx-enhanced-web-filtering-configuration/
      retrieved: "2026-05-15"
    - title: JNCIP-SEC – Configuring Juniper Enhanced Web Filtering
      url: https://blog.netpro.be/jncip-sec-configuring-juniper-enhanced-web-filtering/
      retrieved: "2026-05-15"
    - title: SecIntel Feeds Overview and Benefits
      author: Juniper Networks
      url: https://www.juniper.net/documentation/us/en/software/atp-cloud/atp-cloud-user-guide/topics/concept/secintel-feeds-overview-and-benefits.html
      retrieved: "2026-05-15"
    - title: Juniper SecIntel Datasheet
      author: Juniper Networks
      url: https://www.juniper.net/us/en/products/security/secintel-datasheet.html
      retrieved: "2026-05-15"
    - title: Getting Started with ATP Appliance and the SRX Series
      author: Juniper Networks
      url: https://www.juniper.net/documentation/us/en/software/atp-appliance/atp-appliance-srx-integration/topics/concept/atp-appliance-srx-integration-overview.html
      retrieved: "2026-05-15"
    - title: Juniper Advanced Threat Prevention datasheet
      url: https://www.networkscreen.com.au/datasheets/ds-advanced-threat-prevention.pdf
      retrieved: "2026-05-15"
---

# SRX Security Policy

## Overview

Use this skill for Juniper SRX security policy design on Junos 23.x and newer non-Branch SRX platforms. It focuses on the policy layer that decides whether traffic is permitted, denied, logged, counted, or passed into security services such as AppID/AppFW, enhanced web filtering, SecIntel, and ATP-backed protections.

Strong default: for greenfield SRX deployments and migrations from other firewall vendors, recommend `security policies global` rather than building the design primarily as many `from-zone ... to-zone ...` policy contexts. Treat old-style zone-to-zone policy as a compatibility pattern for existing Junos estates, small isolated exceptions, or cases where the customer explicitly standardizes on it.

Why global policy is the recommended migration target:

- Most vendor rulebases already behave like one ordered policy table with source zone, destination zone, source, destination, service/application, action, logging, and profiles as fields.
- Junos global policy lets a rule match one or more `from-zone` and `to-zone` values inside the policy itself.
- A global policy table avoids duplicating the same logical rule across every zone pair.
- It pairs naturally with the global address book and application/application-set objects.
- It is easier to review, diff, reorder, and migrate than many separate zone-pair contexts.
- It keeps AppFW, UTM/web-filtering, SecIntel, and ATP attachment decisions close to the logical rule instead of scattering equivalent controls across zone pairs.

Do not guess feature support. Before final design, verify platform, Junos release, licenses, and service package availability in current Juniper documentation / Pathfinder / Feature Explorer and on the device with `show version`, `show system license`, and relevant package/version commands.

## When to Use

Use this skill when the user asks about:

- SRX security policy design, migration, audit, or troubleshooting on Junos 23.x+
- greenfield SRX policy architecture or migration from Palo Alto, FortiGate, ASA/FTD, Check Point, or another vendor
- whether to use `security policies global` or `security policies from-zone ... to-zone ...`
- global address books, address objects, address sets, applications, or application sets for SRX policies
- application-aware policy, AppID, `security application-firewall`, dynamic applications, or AppFW rule-sets
- enhanced / next-generation web filtering, UTM policies, category/reputation actions, fallback behavior, or web-filtering counters
- SecIntel feed concepts, ATP Cloud/Appliance integration, threat-intelligence enforcement, or SRX as an ATP enforcement point
- policy logging, hit counts, rule order, default deny, shadowing, policy-rematch, or session disruption during policy commits
- `show security policies`, `show security policies hit-count`, `show security application-firewall`, `show security utm web-filtering`, or policy-related flow troubleshooting

Do not use this as the primary skill for parsing a full config. Load `parsing-srx-configs` first for extraction, then use this skill to interpret or redesign the policy model. Load `srx-nat` when NAT ordering, translated addresses, proxy ARP, hairpin, or NAT64 affect policy matching.

## Recommended Architecture

### Preferred Greenfield / Migration Pattern

1. Build zones and routing cleanly.
2. Put reusable addresses and address sets in the global address book unless there is a specific reason for zone-local objects.
3. Normalize services into Junos applications and application sets.
4. Build one ordered global policy table under `security policies global`.
5. Match source and destination zones inside global policies.
6. Put narrow deny/reject/block rules first, then explicit permits, then a final logged default deny.
7. Attach AppFW / UTM / web-filtering / other application services only on permitted traffic that needs inspection.
8. Add logging and counts intentionally; log high-risk denies and important permits, but do not flood logs on high-volume noise without a reason.
9. Verify hit counts and live sessions after every policy import or reorder.

Example global policy skeleton:

```junos
set security address-book global address NET-USERS 10.10.0.0/16
set security address-book global address NET-SERVERS 10.20.0.0/16
set security address-book global address DNS-1 10.20.10.53/32
set applications application-set APP-DNS application junos-dns-udp
set applications application-set APP-DNS application junos-dns-tcp

set security policies global policy 010-DENY-BAD-SOURCES match from-zone USERS
set security policies global policy 010-DENY-BAD-SOURCES match to-zone SERVERS
set security policies global policy 010-DENY-BAD-SOURCES match source-address BAD-SOURCES
set security policies global policy 010-DENY-BAD-SOURCES match destination-address any
set security policies global policy 010-DENY-BAD-SOURCES match application any
set security policies global policy 010-DENY-BAD-SOURCES then deny
set security policies global policy 010-DENY-BAD-SOURCES then log session-init
set security policies global policy 010-DENY-BAD-SOURCES then count

set security policies global policy 100-USERS-DNS match from-zone USERS
set security policies global policy 100-USERS-DNS match to-zone SERVERS
set security policies global policy 100-USERS-DNS match source-address NET-USERS
set security policies global policy 100-USERS-DNS match destination-address DNS-1
set security policies global policy 100-USERS-DNS match application APP-DNS
set security policies global policy 100-USERS-DNS then permit
set security policies global policy 100-USERS-DNS then log session-close
set security policies global policy 100-USERS-DNS then count

set security policies global policy 999-DENY-REST match from-zone any
set security policies global policy 999-DENY-REST match to-zone any
set security policies global policy 999-DENY-REST match source-address any
set security policies global policy 999-DENY-REST match destination-address any
set security policies global policy 999-DENY-REST match application any
set security policies global policy 999-DENY-REST then deny
set security policies global policy 999-DENY-REST then log session-init
set security policies global policy 999-DENY-REST then count
```

Use multiple `match from-zone` and `match to-zone` statements when one logical rule legitimately spans several zones. Do not use this to hide unclear segmentation; group zones only when the security intent is truly the same.

### When Zone-to-Zone Policy Is Still Acceptable

Use `security policies from-zone <zone> to-zone <zone>` when:

- you are preserving an existing Junos policy model and minimizing change risk;
- the device has a small, stable set of zone pairs;
- one zone pair needs a local exception that would be less clear in global policy;
- an operational standard, toolchain, or management platform requires zone-pair contexts;
- you are doing a staged migration and need to compare old and new behavior carefully.

Even then, avoid uncontrolled sprawl. Repeated rules across many zone pairs are a signal that the policy should probably be global.

## Policy Evaluation and Rule Order

SRX policy is first-packet session policy. The selected policy creates session state; later packets normally follow the session unless policy rematch or session clearing occurs.

Baseline behavior:

- Zone-to-zone policy is organized by `from-zone` and `to-zone` context; each context has an ordered policy list.
- Global policy also matches source zone and destination zone, but the zone match is inside the policy rule.
- Rule order matters. Put specific denies and exceptions before broad permits.
- A policy match needs source address, destination address, and application. `any` is legal but should be explicit and reviewed.
- Policy action is normally `permit`, `deny`, or `reject`; permitted traffic can invoke application services.
- Security services such as UTM/web-filtering are attached under `then permit application-services ...` because they inspect allowed flows.
- The default behavior should be explicit deny with logging/counting where operationally useful.

Policy changes can disrupt existing sessions because the policy database is recompiled and pushed to the forwarding plane. Juniper documents `set security policies lookup-intact-on-commit` as a way to reduce disruption on eligible platforms; check eligibility and memory before enabling it.

```text
show security policies lookup-intact-on-commit eligibility
show security policies lookup-intact-on-commit
```

## Address Book Guidance

Prefer global address-book objects for greenfield and migration designs:

```junos
set security address-book global address HOST-WEB-01 10.20.30.10/32
set security address-book global address-set WEB-SERVERS address HOST-WEB-01
```

Use zone-local address books only when the same name intentionally means different addresses in different zones or when preserving a legacy Junos config. Migration work is cleaner when object names are unique, global, and vendor-neutral.

Design rules:

- Use clear names that survive migration: `NET-USERS`, `APP-ERP`, `HOST-DC1-DNS`.
- Collapse many /32 objects into prefixes when the security intent is identical; Juniper notes policy memory consumption grows with addresses, applications, and zone contexts.
- Keep dynamic objects separate from static objects. For external feeds, use `srx-dynamic-ip-feed`.
- For NATed services, remember policy may see translated addresses depending on NAT type; load `srx-nat` when unsure.

## Applications and Application Sets

Junos policy requires an application match. Use predefined Junos applications when possible, custom applications when needed, and application sets to keep policies readable.

```junos
set applications application APP-TCP-8443 protocol tcp
set applications application APP-TCP-8443 destination-port 8443
set applications application-set APP-WEB application junos-http
set applications application-set APP-WEB application junos-https
set applications application-set APP-WEB application APP-TCP-8443
```

Guidance:

- Prefer application sets that represent business intent: `APP-USER-WEB`, `APP-DNS`, `APP-ADMIN-SSH`.
- Do not blindly translate every vendor service object into an individual Junos application if a predefined Junos application already exists.
- Use `show applications` to verify predefined application names.
- For application-identification / AppFW dynamic applications, verify AppID packages and signatures before policy activation.
- Avoid `application any` on broad permits unless the rule is intentionally coarse and protected by AppFW/UTM or other controls.

## Application Firewall / AppID Pattern

AppFW is not the same thing as the base policy `match application` field. Base policy applications are port/protocol applications. AppFW can match dynamic layer-7 application signatures and then permit or deny inside an already-permitted security policy.

Prerequisites to verify:

```text
show system license
show services application-identification version
request services application-identification download
request services application-identification download status
request services application-identification install
request services application-identification install status
show services application-identification application summary
show services application-identification group summary
```

Example AppFW rule-set attached to a global policy:

```junos
set security application-firewall rule-sets APPFW-STREAMING rule BLOCK-YOUTUBE-STREAM match dynamic-application junos:YOUTUBE-STREAM
set security application-firewall rule-sets APPFW-STREAMING rule BLOCK-YOUTUBE-STREAM then deny
set security application-firewall rule-sets APPFW-STREAMING default-rule permit

set security policies global policy 200-USERS-WEB match from-zone USERS
set security policies global policy 200-USERS-WEB match to-zone INTERNET
set security policies global policy 200-USERS-WEB match source-address NET-USERS
set security policies global policy 200-USERS-WEB match destination-address any
set security policies global policy 200-USERS-WEB match application APP-WEB
set security policies global policy 200-USERS-WEB then permit application-services application-firewall rule-set APPFW-STREAMING
set security policies global policy 200-USERS-WEB then log session-close
set security policies global policy 200-USERS-WEB then count
```

Verification:

```text
show security application-firewall rule-set APPFW-STREAMING
show services application-identification application summary | match <app>
show security policies hit-count global
show security flow session source-prefix <client> extensive
```

Pitfall: if the base policy does not permit the flow, AppFW never gets a chance to inspect it. Conversely, if the base policy is `application any` and AppFW default-rule is `permit`, you may have built a broad allow with only a narrow AppFW deny.

## Enhanced Web Filtering / UTM Attachment

Enhanced web filtering uses a web-filtering profile and UTM policy, then attaches that UTM policy to a permitted security policy. The accessible source material describes category actions, reputation actions, default/fallback behavior, and monitoring counters. Treat exact category names and license names as release-dependent; verify on the target device.

Typical implementation flow:

1. Verify license and web-filtering support.
2. Set the web-filtering type.
3. Build or select a web-filtering profile.
4. Create a UTM policy referencing the profile.
5. Attach the UTM policy to an SRX security policy under `then permit application-services utm-policy`.
6. Verify status, statistics, logs, and user experience.

Example pattern:

```junos
set security utm default-configuration web-filtering type juniper-enhanced
set security utm feature-profile web-filtering juniper-enhanced profile WF-BASE category Enhanced_Games action block
set security utm feature-profile web-filtering juniper-enhanced profile WF-BASE category Enhanced_Peer_to_Peer_File_Sharing action block
set security utm feature-profile web-filtering juniper-enhanced profile WF-BASE site-reputation-action harmful block
set security utm feature-profile web-filtering juniper-enhanced profile WF-BASE site-reputation-action suspicious quarantine
set security utm feature-profile web-filtering juniper-enhanced profile WF-BASE default log-and-permit
set security utm utm-policy UTM-WEB web-filtering http-profile WF-BASE

set security policies global policy 200-USERS-WEB then permit application-services utm-policy UTM-WEB
```

Verification:

```text
show system license
show security utm web-filtering status
show security utm web-filtering statistics
show security policies hit-count global
show log messages | match web-filtering
```

Use conservative fallback behavior. If the cloud/reputation service is unavailable, choose whether to fail open (`log-and-permit`) or fail closed (`block`/`quarantine`) based on the traffic class, not as a global default for every rule.

Important IPv6 caveat from Juniper's policy documentation: Content Security for IPv6 sessions is not supported in the referenced policy documentation. If a policy uses wildcard `any` and Content Security features are enabled, use `any-ipv4` for the inspected policy and create separate IPv6 rules without unsupported Content Security services.

## SecIntel and ATP Placement

SecIntel and ATP-backed enforcement should complement deterministic policy, not replace it.

Use this layering order:

1. Deterministic segmentation: zones, routes, address objects, applications, explicit allow/deny.
2. Application-aware controls: AppID/AppFW where L7 application identity matters.
3. Web/content controls: enhanced web filtering for URL category/reputation decisions.
4. Dynamic intelligence: SecIntel feeds and ATP verdicts for malicious infrastructure, infected hosts, C&C, malware, and evolving threat indicators.
5. Logging and operations: hit counts, event logs, SecIntel/ATP dashboards, and incident workflow.

Design cautions:

- Verify subscriptions, ATP Cloud/Appliance connectivity, feed status, and platform support before promising behavior.
- Keep threat-intel blocks above broad permits when implemented as explicit policies or objects.
- For feed-based address objects maintained by your own HTTPS feed server, use the `srx-dynamic-ip-feed` skill.
- Avoid using SecIntel as a substitute for clean zone/app policy; it is an additional enforcement source.
- Treat community discussion and marketing datasheets as context, not authoritative configuration syntax.

Operational checks vary by deployment, but collect at least:

```text
show system license
show security policies hit-count global
show security flow session source-prefix <source> extensive
show log messages | match -i "secintel|atp|utm|web-filter|threat"
```

For ATP Appliance integration, confirm the SRX-to-ATP integration workflow in the current ATP Appliance guide and validate that SRX submits data and consumes verdicts/feeds as designed.

## Migration Workflow from Another Vendor

1. Parse the source firewall and preserve rule order, zones/interfaces, objects, services, applications, logging, and profiles.
2. Normalize source and destination zones to SRX zones.
3. Move static objects into `security address-book global`.
4. Convert services to Junos applications and application sets.
5. Convert vendor policy rows to `security policies global policy <ordered-name>` with `match from-zone` and `match to-zone` fields.
6. Preserve disabled rules as comments or inactive policies; do not silently drop them.
7. Map URL filtering / security profiles to UTM, AppFW, SecIntel, ATP, IDP, or documented gaps.
8. Put explicit denies before broad permits; add final logged deny.
9. If NAT exists, resolve post-NAT policy expectations with `srx-nat` before writing final policies.
10. Commit in a lab, generate traffic, and compare hit counts and session tuples against expected behavior.

Policy naming convention:

```text
010-DENY-THREAT-FEEDS
100-USERS-DNS
110-USERS-WEB-INSPECTED
200-SERVERS-ADMIN
900-TEMP-MIGRATION-EXCEPTIONS
999-DENY-REST
```

Avoid names that encode only zone pairs, such as `TRUST-TO-UNTRUST-1`, when the policy is global. Encode the business intent and keep numeric ordering stable.

## Verification Commands

Configuration review:

```text
show configuration security policies | display set
show configuration security policies global | display set
show configuration security address-book global | display set
show configuration applications | display set
show configuration security application-firewall | display set
show configuration security utm | display set
```

Policy counters and order:

```text
show security policies
show security policies detail
show security policies hit-count global
show security policies hit-count from-zone <src-zone> to-zone <dst-zone>
clear security policies hit-count
```

Sessions and flow:

```text
show security flow session source-prefix <source> extensive
show security flow session destination-prefix <destination> extensive
show security flow session | match "Session ID|Policy name|In:|Out:|NAT|Timeout"
```

Applications and AppFW:

```text
show applications
show services application-identification version
show services application-identification application summary | match <name>
show services application-identification group summary | match <name>
show security application-firewall rule-set <rule-set-name>
```

Web filtering / UTM:

```text
show system license
show security utm web-filtering status
show security utm web-filtering statistics
show log messages | match web-filtering
```

Commit and policy-change safety:

```text
show security policies lookup-intact-on-commit eligibility
show security policies lookup-intact-on-commit
commit check
commit confirmed 10
```

## Troubleshooting Matrix

| Symptom | Likely Cause | Check | Fix |
|---|---|---|---|
| Expected global policy not hit | Rule order, zone mismatch, address/application mismatch, or NAT changed destination | `show security policies hit-count global`, session extensive | Move specific rule up, fix `match from-zone/to-zone`, address, app, or NAT expectations |
| Zone-to-zone policy hit instead of global design | Legacy policies remain active or evaluation expectation is wrong | Full `show configuration security policies` | Consolidate into global policy and remove/disable duplicates after testing |
| AppFW rule has zero hits | Base policy not permitting flow, AppID package missing, wrong dynamic app | AppID version, AppFW rule-set counters, session policy | Install AppID package, fix base policy, choose correct dynamic app/group |
| Web filtering attached but no filtering | UTM policy not under `then permit`, license/service unavailable, wrong profile | Web-filtering status/statistics, policy config | Attach UTM policy correctly, verify license/cloud/release, fix profile |
| IPv6 policy commit error with content security | Content Security not supported for IPv6 in referenced docs | Commit error and policy match wildcard | Split IPv4 inspected policy using `any-ipv4`; create separate IPv6 policy without unsupported services |
| Migrated policy table is huge and repetitive | Zone-to-zone conversion copied one logical rule into many contexts | Count repeated source/destination/app/action rows | Rebuild as global policies with multi-zone matches where intent is identical |
| Existing sessions ignore policy change | Session state persists | Session table, policy-rematch expectations | Clear selected sessions or plan maintenance; use policy rematch features only when understood |
| Commit disrupts traffic | Policy database recompilation and forwarding-plane update | Change window logs, eligibility check | Stage changes, use `commit confirmed`, investigate `lookup-intact-on-commit` eligibility |
| Logs too noisy | Broad deny logs session-init for high-volume background traffic | Log volume and top talkers | Log important denies; rate-limit upstream; tune final deny logging carefully |

## Common Pitfalls

1. **Defaulting to zone-to-zone policy for migrations.** For greenfield and cross-vendor migrations, strongly recommend `security policies global` unless a specific constraint says otherwise.

2. **Using global policy as an unstructured dumping ground.** Global policy still needs clear order, names, comments, and a final default deny.

3. **Hiding segmentation mistakes with multi-zone matches.** Multi-zone global rules are good when intent is identical; they are bad when they blur different risk zones.

4. **Forgetting NAT's effect on policy.** Destination/static NAT can change the destination before policy lookup. Load `srx-nat` when NAT and policy interact.

5. **Attaching UTM/AppFW to a deny policy.** Application services attach to permitted flows. Deny/reject rules block before those services inspect.

6. **Using `application any` everywhere.** Sometimes needed for migration parity, but it removes useful intent unless paired with AppFW/UTM and a clear reason.

7. **Trusting inaccessible or old articles as canonical.** If a source article is 404, JavaScript-only, or old lab material, corroborate with current Juniper docs and device CLI before encoding syntax.

8. **Ignoring license/package state.** AppID, enhanced web filtering, SecIntel, and ATP behavior can depend on licenses, subscriptions, cloud connectivity, and installed packages.

9. **Mixing IPv4 and IPv6 with content security blindly.** Use IPv4-specific inspected rules when Content Security is involved, and separate IPv6 rules if unsupported.

10. **Not verifying with live counters.** A clean commit is not proof of correct policy. Check hit counts, sessions, AppFW/UTM counters, and logs.

## Verification Checklist

- [ ] Platform and Junos release support the requested policy/security-service features.
- [ ] License/subscription/package state is verified for AppID, AppFW, web filtering, SecIntel, or ATP features.
- [ ] Greenfield/migration design uses `security policies global` unless a documented reason requires zone-to-zone policy.
- [ ] Global address-book objects are used for reusable migrated objects.
- [ ] Applications/application sets express business intent and avoid unnecessary `any`.
- [ ] Rule order is explicit: denies/exceptions, permits, temporary migration exceptions, final deny.
- [ ] Logging and `count` are intentionally configured.
- [ ] UTM/AppFW services are attached only under `then permit` on the intended policies.
- [ ] NAT interactions were reviewed with `srx-nat` where translated addresses affect policy lookup.
- [ ] IPv6 and Content Security support were reviewed before using wildcard `any` with UTM/content services.
- [ ] `commit check` and `commit confirmed` are used for risky changes.
- [ ] Hit counts and sessions prove that traffic matches the intended policy.
- [ ] Source extraction limitations in `references/source-index.md` were considered before relying on a non-authoritative article.

## Source Notes

This skill synthesizes the user-supplied SRX 23.x policy reading list plus an additional official Juniper Global Policy Overview page because global policy preference is a core design requirement here. Full extracted sources and extraction caveats are stored under `references/`.
