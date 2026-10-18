---
status: draft
---

# Design review: inconsistencies and gaps in cynapse's design

Companion to [`spec.md`](./spec.md). Found by reading cynapse's ADRs 0001–0014, its code and open
issues against cyberlegion, cyberfleet, civitas (`spikyuni.github.io` decisions), SDD, cyber-mux,
cyber-truss, DNA and uni-agent-kit.

Labels are stable. `I` is an **inconsistency** (two sources say different things). `M` is
**missing** (a neighbour needs it, or the design promises it, and nothing defines or builds it).
Each entry gives the evidence and a recommendation. Severity: **high** blocks a neighbour or is
expensive to undo; **med** causes drift; **low** is bookkeeping.

Paths are relative to `~/code/cyberuni/` unless they start with `cynapse/`.

## Summary

| Label | Severity | One line |
| --- | --- | --- |
| [I1](#i1) | high | Who owns presence: cynapse, or the runtime |
| [I2](#i2) | high | "Lease" means four different things; "claim" is a fifth word |
| [I3](#i3) | high | Project address key: git common dir, forge node id, or cyberfleet's own |
| [I4](#i4) | med | `wake` trait: "not built", "advice", and a CLI flag that does nothing |
| [I5](#i5) | med | Who owns waiting: cyberlegion `mail await` or `cynapse entry wait` |
| [I6](#i6) | high | `0.1.0` is declared a frozen contract while three breaking changes are queued |
| [I7](#i7) | low | `cynapse.awaiting-reply` is a listed contract type that no code defines |
| [I8](#i8) | high | `parent`, `subject` and `kind` each name two things |
| [I9](#i9) | med | Org and repo drift: `spikyuni/cynapse` versus `cyberuni/cynapse` |
| [I10](#i10) | low | Status text is stale in four places |
| [I11](#i11) | med | "Ledger", "combat log" and "mail" mean different things in different repos |
| [I12](#i12) | med | cyberlegion `ack` is per message; cynapse read state is a forward-only cursor |
| [I13](#i13) | high | Anyone can delete, yet cyberfleet puts authority records in cynapse as audit |
| [I14](#i14) | med | Boundary pressure: state records used for facts another store could hold |
| [I15](#i15) | med | Who registers roles and humans, and as what kind |
| [I16](#i16) | low | ADR supersession is not marked on the superseded ADR |
| [I17](#i17) | high | `CYNAPSE_PARTICIPANT` is cynapse's identity channel; no runtime sets it |
| [I18](#i18) | low | "Hub" names two different things |
| [I19](#i19) | low | cynapse's own fixtures model other projects' domains |
| [I20](#i20) | med | Two channel-identity schemes coexist without a rule for which applies |
| [M1](#m1) | high | No wait across channels |
| [M2](#m2) | med | No way to remove a member or add one after create |
| [M3](#m3) | high | No strict identity or sender attribution |
| [M4](#m4) | high | Leases and claims: no TTL, exclusivity, fencing or compare-and-set |
| [M5](#m5) | high | No recipe for a project address, a subject-less work channel, or a ship queue |
| [M6](#m6) | med | No registry, validation or thread type for consumer-defined types |
| [M7](#m7) | med | Guide, compose and `cynapse.published` are unbuilt |
| [M8](#m8) | med | No versioning policy; error codes incomplete |
| [M9](#m9) | med | No maintenance: backup, integrity check, retention, durability setting |
| [M10](#m10) | low | `search` is not in the CLI; no edit-an-entry |
| [M11](#m11) | high | Several sessions acting as one handle are undefined |
| [M12](#m12) | med | No design for the hub, offline writers or access control |
| [M13](#m13) | med | No mention concept, though cyberfleet relies on `@handle` |
| [M14](#m14) | med | The plugin ships no skills; `init-cynapse` is unbuilt |
| [M15](#m15) | med | No behavioral spec, control-flow graph or scenario suite exists |
| [M16](#m16) | med | No path to import existing cyberlegion mail |
| [M17](#m17) | high | No confidentiality model for sensitive channels |
| [M18](#m18) | med | No way to ask "which runtimes should hear about this channel" |

---

## Inconsistencies

### <a id="i1"></a>I1. Who owns presence (high)

- `cynapse/AGENTS.md` settled decisions: "cynapse owns participant addressing and identity: addresses, standing/owner identity, **and presence**."
- ADR-0002 and ADR-0011 list "leases and presence" as what cynapse stores.
- ADR-0013 need 8: "Session liveness is asserted by the runtime, never measured by cynapse"; cynapse has "no presence leases or heartbeats."
- Civitas 0002: "**Presence lives in the runtime.** cynapse may display it when the runtime publishes it." cyberlegion `unit/participants`: "cynapse never measures whether a session is alive."
- cyberlegion also uses "presence" for a different thing: the stand-in session bound to a standing owner (`unit claim`).

**Recommend.** Treat ADR-0013 and civitas as the decision. cynapse stores a registration status
(`live` or `retired`) that the runtime asserts. Remove "presence" from AGENTS.md, ADR-0002 and
ADR-0011's lists (by a superseding note, see I16) and rename the status to "registration status" so it is not mistaken for presence.

### <a id="i2"></a>I2. "Lease" means four things; "claim" is a fifth word (high)

1. cynapse ADR-0006: a lease is a state record with TTL, exclusive flag, path patterns, `released_ts`, orphan repair, modelled on mcp_agent_mail. Not built (#80 "leases with read-time expiry").
2. Civitas 0002: "Coordination claims live in cynapse. **Ownership leases live in the runtime.**" and "cynapse's planned lease … is built as a claim." cynapse's docs were not updated to say "claim".
3. cyberlegion ADR-0033 `service/lease`: a fenced reservation with `generation`, expiry and holder, kept in the hub.
4. `uni-agent-kit/packages/agent-harness/src/worktrees/lease.ts`: an exclusive-created git `locked` file holding `{library, leaseId, holder}`.
5. `state set --kind lease` today is a free string with no behavior.

Nothing says which of these cynapse is meant to provide, and (3) needs a **generation (fencing) check** that ADR-0006's design lacks.

**Recommend.** Rename cynapse's concept "claim" everywhere, as civitas already does. Decide in a new ADR whether cyberlegion's fenced service lease stays in the runtime (civitas says yes) or moves. Add a fencing token to the claim design only if a consumer outside cyberlegion needs it. See M4.

### <a id="i3"></a>I3. Project address key (high)

- `cynapse/AGENTS.md:62`: "Project addressing lives here, including the derivation from the git common directory" (cites civitas 0004).
- ADR-0013 §9: a repository's address channel is keyed by the forge's **native id** (`gh repo view --json id`). "The root commit SHA is not used."
- Civitas 0004 says two clones resolve to two addresses.
- cyberfleet ADR-0030 supersedes civitas 0004: the key is cyberfleet's, via a ship registry; `src/project-address.ts` is to be replaced.
- cyberfleet test fixtures use `cynapse:acme/widget` with `keyScheme: 'cynapse'`; the code uses `KEY_SCHEME = 'cyberlegion'`.
- cyberlegion's lease and endpoint specs key by project, so two projects with the same display name collide.
- A repository with no hosted remote has no native id; ADR-0013 says cynapse registers an address, but gives no derivation.

**Recommend.** Delete the git-common-dir line from AGENTS.md. State once, in the spec, that cynapse builds keys from `(store, nativeId)` and registers its own id for things with no store. Settle the no-remote case (see M5).

### <a id="i4"></a>I4. The `wake` trait (med)

- ADR-0005 lists "whether members are woken" as a trait. ADR-0013 keeps `ChannelTraits.wake` as "advice to the runtime."
- Civitas 0001: "under this decision that trait is not built."
- Code: `channel create --wake` stores it and nothing reads it (`types.ts:81`). cyberfleet's plan says "stored but not acted on."
- cyberlegion's `unit/participants` lists "waking from cynapse's change token" as a non-goal.

**Recommend.** Keep it, but document it as advice only, and have the CLI help say so. If no runtime reads it within a release, remove it before it becomes a contract.

### <a id="i5"></a>I5. Who owns waiting (med)

- cyberlegion #153: `mail await` "stays in cyberlegion, built on `changes(since)` and entry reads."
- cyberfleet ADR-0024 §5 uses `cynapse entry wait` on its address channel and asks for a cross-channel wait (G6).
- Both are polling loops over the same `entries` and `changes` calls.

**Recommend.** One of them is redundant. If cyberfleet calls cynapse directly (its stated design), cynapse needs the cross-channel wait (M1) and cyberlegion's await becomes a thin caller or is dropped.

### <a id="i6"></a>I6. A frozen contract with breaking changes queued (high)

- ADR-0013 item 6: from `0.1.0`, the `Store` interface, `--json` shapes and `cynapse.*` types are public, and changing them "is a breaking change."
- Queued: async `Store` (#71), renamed terms (#72), regrouped verbs (#75), stable error codes (#77), strict `--as` (#81), moving dev/CLI internals out of main exports (#78).
- cyberlegion calls the `Store` **synchronously** and pins `^0.1.0`; cyberfleet's plan mitigates with one adapter module and `^0.2.0`.
- For `0.x` caret ranges, `^0.1.0` already excludes `0.2.0`, so each break strands cyberlegion until it upgrades.

**Recommend.** Land #71, #72 and #77 in one release, announce it in the changelog as the last pre-1.0 break, and publish a deprecation window for the old names. See M8.

### <a id="i7"></a>I7. `cynapse.awaiting-reply` (low)

ADR-0013 item 6 lists `awaiting-reply` among "the `cynapse.*` entry types this ADR adds" and part of the public contract. It is a **state record kind** (`kind: cynapse.awaiting-reply`), not an entry type, and no source file defines it (state `kind` is a free string). The ADR's "Built" list omits it.

**Recommend.** Either export a constant and reserve the kind (like `HANDLED_TAG`), or remove it from the contract list.

### <a id="i8"></a>I8. Words that name two things (high)

| Word | First meaning | Second meaning |
| --- | --- | --- |
| `parent` | The entry a reply answers (ADR-0003) | The anchor entry a child channel branches from (ADR-0004) |
| `subject` | What a channel is about (ADR-0010, `SubjectId`) | The participant a state record waits on (`--subject`); also cyberlegion's mail subject line |
| `kind` | Participant kind (`agent`, `human`, `service`) | State record kind; channel kind (address, work) |
| `owner` | An address channel's owner | cyberlegion's "standing owner" and "owner mailbox" |

Issue #72 plans `subject`→`participant`, `kind`→`type`, `parent`→`anchor`, but cyberfleet's checkpoint says to write nothing against current flags until it lands.

**Recommend.** Do #72 before more consumers appear. In the same pass fix the `owner` clash by naming cyberlegion's concept "standing role".

### <a id="i9"></a>I9. Org and repo drift (med)

- The repository is `spikyuni/cynapse` (HEAD `68b25c7`, `package.json` `repository`).
- Still pointing at `cyberuni/cynapse`: `readme.md`, `knip.json`, `AGENTS.md`, `channel-key.ts`, `cli-error.ts`, `packages/cynapse-gui/{README,CHANGELOG}.md`, and ADR links.
- `cynapse gui` says it "needs `@cyberuni/cynapse-gui`" while the CLI package is unscoped `cynapse`.
- Civitas links use `cyber-civitas.github.io`; the content now lives under `spikyuni.github.io`.
- cyberlegion and cyberfleet link `cyberuni/cynapse` issues (#30, #60, #75).

**Recommend.** Decide the final org and package scope, then fix links in one pass. GitHub redirects keep old issue links working after a transfer, so only code and docs need the edit.

### <a id="i10"></a>I10. Stale status text (low)

- ADR-0008 "Not built yet"; ADR-0010 and ADR-0011 "Not built yet" (true), but ADR-0013 Context still says "npm has only `0.0.0`" and "no `participant` CLI command."
- ADR-0009 Consequences: "the schema has no version" (migrations now exist).
- Civitas `components.md`: "published 0.1"; cyberfleet ADR-0023:167: "cynapse has no release yet."
- cyberlegion `apps/web/.../concepts/mail-model.md`: cynapse "at the scaffold stage."
- Issue #60 is closed and ADR-0014 is built, yet cyberlegion's participants README still says stage 3 is "blocked on #60."

**Recommend.** Accepted ADRs are not rewritten, so correct the *neighbours* and keep ADR Status lines current only for the "Built:" paragraph.

### <a id="i11"></a>I11. "Ledger", "combat log", "mail" (med)

- ADR-0002: cynapse owns "ledgers of what happened." SDD's ledger and combat log are **git-tracked, sharded JSONL that must survive branches** and meet a safe-to-publish floor. ADR-0007 says raw conversation never goes into a repository, and calls SDD's combat log "a candidate to move onto a cynapse stream."
- These requirements conflict. A cynapse channel is outside the repo and has no branch semantics; SDD's value is that the record travels with the branch. cyberfleet's checkpoint (R2) says the mission ledger moves; ADR-0029 says SDD is retiring the graph and its move is "a separate question." Hypothesis HY01 is untested.
- "Mail": ADR-0002 says "mail is not a separate concept"; cyberlegion has a "mail plane" and commands.

**Recommend.** Write down which SDD records move and which stay. Until then, drop the combat-log sentence from ADR-0007's consequences. Use "message" in cynapse prose and keep "mail" as a cyberlegion CLI term.

### <a id="i12"></a>I12. Ack versus cursor (med)

cyberlegion: `ack` is the **only** command that changes read state, per message; two concurrent acks give exactly one success; `delete` works on read and unread mail. cynapse: read state is a per-reader cursor that only moves forward; `handled` is a tag that only the **address channel owner** may set (ADR-0013 §5).

Gaps: reading message 7 without having read message 5; acking out of order; a non-owner recipient marking something handled (for example a role's inbox read by a stand-in session); concurrent handlers.

**Recommend.** Say plainly in the spec that cynapse has no per-message read state, only "up to here" plus owner-set `handled`. Check that `appendUnless` (compare-and-set on append) is enough for the "exactly one handler" case or add compare-and-set to tags.

### <a id="i13"></a>I13. Anyone can delete, but authority lives here (high)

- ADR-0014: "There is no permission check" on deleting entries or channels; `--as` is a claim.
- Cyberfleet `design/authority.md:56`: "Authority records live in cynapse, in the channel and its metadata." Its model says "the channel holds the audit record, never the authority."
- Only `cynapse.*` entries are undeletable, and consumer decision entries (`sdd.decision`, `delegation.record`) are not.
- cyberlegion #154: `mail send` stamped another live unit as the sender. cyberlegion #153 dropped authenticated `--as`; cynapse #81 asks for a strict mode.

So the audit trail for a decision can be tombstoned by anyone willing to claim an identity, and the tombstone logs the claimed name.

**Recommend.** Decide whether the tombstone log is enough audit (ADR-0014 argues yes) and say so in cyberfleet's authority doc. Offer a per-type protection (an entry type can be marked non-deletable by its channel's owner) if cyberfleet needs more. See M3, M17.

### <a id="i14"></a>I14. Boundary pressure on state records (med)

ADR-0011: use cynapse's store "only where no external store fits." cyberfleet ADR-0029 puts blocking links and cross-repo ordering in state records "where the tracker cannot" and the Operator's graph writes there too. That is allowed, but ADR-0010 also says relations are metadata on both ends in their own stores, and DNA says "write through, keep no copy."

**Recommend.** Add a test to the spec for any new state kind: "which store would hold this if cynapse did not, and why can it not?" Record the answer in the ADR that adds the kind.

### <a id="i15"></a>I15. Who registers roles and humans (med)

- ADR-0013: a unit registers itself only by omitting `registeredBy`, and then must be a `service`; `registeredBy` names a `service`.
- cyberlegion registers units as `agent`, with the hub (a `service`) as registrar; it explicitly does **not** mirror standing owners or service endpoints (`participants.feature`).
- cyberfleet needs roles (`<ship>/captain`), a human (`COUNCIL`) and ship handles as participants (ADR-0030: "cynapse holds the handles as participants"). Nothing registers them, and the key shape for roles differs (`cyberlegion:role/reviewer` in ADR-0013 versus a ship-derived handle).
- ADR-0013 need 7 says several humans "fit as-is," but `addParticipant` is implicit, and `#ensureParticipant` still creates unknown `--as` values as agents.

**Recommend.** State in the spec who may register which kind, and what key namespace each registrar uses. Make `--as` strict by default (#81) once the registrars exist.

### <a id="i16"></a>I16. Supersession bookkeeping (low)

ADR-0011 "supersedes the Decision section of ADR-0002," but ADR-0002's Status still reads "Accepted." ADR-0012 amends ADR-0004, ADR-0010 amends ADR-0005, ADR-0014 supersedes the "seq ranges" sketch in ADR-0006, and none of those ADRs carries a pointer forward. `docs/adr/README.md` has no status or supersession column. Readers of ADR-0002 see the list that ADR-0011 corrected (change feeds, presence).

**Recommend.** Add "Superseded in part by ADR-00NN" lines to the older ADRs' Status (the one permitted edit) and a status column to the index.

### <a id="i17"></a>I17. The identity channel nobody sets (high)

- cynapse reads the acting participant from `--as` or `$CYNAPSE_PARTICIPANT`.
- cyberlegion identifies a unit by `CYBERLEGION_AGENT_ID`, `CYBERLEGION_ROOT` and pane env. `CYNAPSE_PARTICIPANT` appears in none of cyberlegion, cyberfleet, SDD or their specs.
- ADR-0013 item 7 requires passing `$CYNAPSE_HOME` to launched agents but says nothing of `$CYNAPSE_PARTICIPANT`.
- A launched agent that calls `cynapse` without `--as` therefore has no identity, or is created as a new agent named after its id.

**Recommend.** Add the environment contract to the spec: both variables are set by whatever launches an agent, and the participant id (not the name) is the value. Add a cyberlegion scenario that checks the spawn environment.

### <a id="i18"></a>I18. "Hub" (low)

ADR-0003 and ADR-0007 use "hub" for the future sync owner of `seq`. cyberlegion's hub is a directory (`~/.agents/cyberlegion`) with a lock and a registry, and the "hub" is also a `service` participant that registers units. Two hubs sharing a machine will be confusing.

**Recommend.** Call cynapse's future one the "sync hub" or rename cyberlegion's.

### <a id="i19"></a>I19. Fixtures model other projects (low)

`cynapse-gui/src/core/fixture.ts` and `dev seed` model an "SDD mission graph," an `sdd.mission-ledger` convention, an "epic" and a "Fleet operator" (cyberfleet checkpoint R4). That ties examples to terms cyberfleet is dropping, and can mislead readers about what cynapse defines (ADR-0005: it defines none).

**Recommend.** Rename the seeded types to a neutral namespace (`example.*`).

### <a id="i20"></a>I20. Two channel-identity schemes (med)

ADR-0012 says every channel is keyed by its subject. The code still derives a branched channel's id as `UUIDv5(anchor entry id)` (`sqlite.ts:419`) and a keyed channel's as `UUIDv5(key)` (`:421`). So a thread or arbitration channel has no subject key, `channel resolve` cannot find it, and cyberfleet's thread type (R9) has nowhere to live except on the anchor entry.

**Recommend.** State the rule: subject-keyed channels for things that exist elsewhere, anchor-derived for conversation branches, and nothing else. Say whether a branched channel may also gain a subject key.

---

## Missing

### <a id="m1"></a>M1. Wait across channels (high)

`entry wait` covers one thread. cyberfleet G6 asks for `unread --wait` or `changes --wait` to block until anything arrives for a participant. Without it each runtime writes its own poll loop (I5).

### <a id="m2"></a>M2. Members after create (med)

`addMember` exists; there is no remove and no CLI verb for add outside `channel create`. cyberfleet G7 cites #75, which is now "regroup CLI verbs," so the request is not tracked. Needed: `member add|remove|list`, with role changes written as entries.

### <a id="m3"></a>M3. Strict identity and sender attribution (high)

#81 (strict `--as`), cyberlegion #154 (spoofed sender), cyberfleet G8. No design says what "strict" means: registered names only? only ids? a runtime-signed token? ADR-0013 defers real checks to the hub.

### <a id="m4"></a>M4. Claims (high)

Designed in ADR-0006, unbuilt (#80). Missing from the design: how expiry works with no daemon ("read-time expiry" in #80), a fencing token, compare-and-set on `setState` (#79), an owner for orphan repair, and how it differs from cyberlegion's service lease (I2).

### <a id="m5"></a>M5. Addresses with no obvious key (high)

- A repository with no hosted remote, and a folder (ADR-0012 says cynapse "registers an address"; `registerAddress` is not idempotent per cyberfleet's plan).
- A work channel with no external subject (a prompt with no issue), cyberfleet ADR-0023 open dependency.
- A ship's queue channel versus the repository's address channel (ADR-0030).
- Two clones of one repository (I3).

Each needs a defined key recipe and a test that two callers derive the same channel.

### <a id="m6"></a>M6. Consumer types (med)

ADR-0005 makes types consumer-defined, so cynapse cannot validate payloads. Missing: how a consumer declares its types (cyberfleet F3 needs `delegation.record`, `pod.report`, `pr.ready`, `trunk.moved`, `pr.escalation`, `fleet.dispute`), a collision check between namespaces, and a **thread type** carried on the anchor entry (cyberfleet R9, `design/lifecycle.md:78`). Also #76: opening a subject's channel with another type silently keeps the first.

### <a id="m7"></a>M7. Guide, compose, `cynapse.published` (med)

ADR-0010 and ADR-0011 are accepted and unbuilt: no composition API, no change-feed merge, no `cynapse.published` shape. cyberfleet ADR-0023 §3 already depends on `cynapse.published`; its shape becomes a contract the day it ships.

### <a id="m8"></a>M8. Versioning and errors (med)

No policy says what a minor version may break while `0.x`. #77 (stable codes for every error) is open; only exit codes 3, 4 and 5 are defined besides generic failure and usage. A runtime branching on a code needs the whole list.

### <a id="m9"></a>M9. Operations (med)

Backup, integrity check, vacuum, export (#86); a `synchronous = FULL` setting (#88); the `retention` trait is stored but unused (#87); indexes and batching for large stores (#83). ADR-0014 requires erased bytes to leave the WAL (#67 closed), but there is no test for backups taken before a delete.

### <a id="m10"></a>M10. Search and edit (low)

`search` is in `Store` but not in the CLI (#85); editing an entry as a folded append is requested (#62). Output budget flags (#84) and participant names in text output (#82) are also open.

### <a id="m11"></a>M11. Many sessions, one handle (high)

cyberfleet ADR-0030: "Any number of sessions may act as one handle," and a mention wakes every live session acting as it. cyberlegion fails loud on a shared handle, ADR-0033 allows one lease holder per key, and cynapse `resolveAddress` returns one participant or fails. ADR-0013 puts "which session acts as the role" in the runtime and says the cursor belongs to the participant. Not defined: two sessions reading the same role's cursor at once (each marks the other's reads), and whether `handled` needs a holder.

### <a id="m12"></a>M12. Hub, offline writers, access control (med)

ADR-0003: entries from an offline writer "wait without a `seq`"; "that flow isn't built." ADR-0007: a hub beyond one machine, NATS or Dolt "not adopted." ADR-0014: deletes must travel as log entries. Without it, the "channel is the unit of access control" claim has no mechanism (I13, M17).

### <a id="m13"></a>M13. Mentions (med)

cyberfleet relies on `@<ship>/captain` mentions and waking every live session behind a handle. cynapse has no mention concept (not in ADR-0005 or ADR-0013, nor in the code). Decide: an entry field (`mentions: [participantId]`) so `unread` can surface it, or a convention the runtime parses.

### <a id="m14"></a>M14. The plugin ships nothing (med)

`plugin.json` and the vendor manifests exist; `skills/` does not, and `init-cynapse` (ADR-0008) is unbuilt. `cynapse setup status|route|materialize|agents-md`, `.agents/cynapse.json`, and stamps are described and not coded. No SDD role-map or hook anywhere expects cynapse (the only install paths are cyberlegion's optional peer dependency and `npm i`).

### <a id="m15"></a>M15. No behavioral specs (med)

This is the first spec directory. There is no control-flow graph, no scenario map and no `.feature` suite for any capability; the oracle today is ADR prose and `*.test.ts`. Draw the CFGs for the four flows listed in `spec.md` first.

### <a id="m16"></a>M16. Moving existing mail (med)

cyberlegion stage 2 and 3 move traffic and retire the hub mailboxes. Nothing defines whether existing mail is imported, left to expire, or abandoned, how a `<epochMs>-<hex>` id maps to a UUIDv7 (and to a `seq`), or what happens to an owner mailbox's unread set. ADR-0013 leaves the plan to cyberlegion.

### <a id="m17"></a>M17. Confidential channels (high)

cyberfleet puts authority records and escalations in cynapse "never on a public tracker." The database is a single local file readable by every process of the user, with no per-channel access control. Nothing says whether that is acceptable (local trust), whether channels can be encrypted, or what happens when the hub arrives (M12).

### <a id="m18"></a>M18. From channel to runtimes (med)

cyberlegion #7 is described three ways (observer isolation, ringing the runtimes of a channel's members, routing mail to the owner). A runtime that wants to wake the right sessions for a channel needs: the channel's members, each member's registered participant, and who registered it (`participants({registeredBy})`). The calls exist separately; no recipe or single call gives a runtime "who to ring for this entry."

---

## Suggested order

1. **I6, I8, M8**: one breaking release (async, renames, error codes) so neighbours adapt once.
2. **I1, I2, I3, I9**: one docs pass fixing AGENTS.md, ADR status lines and links.
3. **I17, I15, M3**: the identity contract (environment, registrars, strict `--as`).
4. **M1, M2, M13, M18**: the pieces cyberfleet needs to call cynapse directly.
5. **M4, M11, I13, M17**: claims and the confidentiality decision; each needs an ADR before code.
6. **M15**: write the behavioral nodes as each of the above lands.
