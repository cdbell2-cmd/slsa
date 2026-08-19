# SLSA Explained Simply

> **Feynman approach:** If you can't explain it simply, you don't understand it well enough.  
> This document explains SLSA as if you've never heard of it — then builds to what it means for your pipelines.

---

## Start here: the problem we're solving

Imagine you ordered medicine from an online pharmacy. The bottle arrives. It looks right. But how do you know:

- It came from the pharmacy you ordered from, not a warehouse in between?
- Nobody swapped the pills while the box was in transit?
- The pharmacy actually made it the way they're supposed to — not some rogue employee cutting corners?

You can't know — unless the pharmacy gives you a **chain of custody document**: signed at manufacture, signed at packaging, signed at shipping. Each step verified and tamper-evident.

**SLSA is that chain of custody for software.**

Every time your team builds an application, something goes in (source code) and something comes out (an artifact — a container image, a JAR, a binary). SLSA is a framework that guarantees you can prove: _this exact artifact came from this exact source code, built this exact way, on this trusted platform, and nobody touched it in between._

---

## Why this matters right now

In 2020, attackers compromised SolarWinds' **build system** — not the source code. They injected malicious code _during the build_ and the resulting software looked perfectly legitimate. It was signed, it passed tests, it shipped to 18,000 customers.

The source code was fine. The build was poisoned.

SLSA exists so that a scenario like that leaves fingerprints — or better, is structurally impossible.

---

## The simplest possible explanation of each level

Think of it like a restaurant health inspection rating.

### Level 0 — No rating

The kitchen has no rules. The chef might wash their hands. They might not. You have no way to know. This is where most software builds are today — they probably do the right thing, but there's no proof.

### Level 1 — Posted procedures

The kitchen has a written recipe for every dish. The recipe is stored in a binder on the shelf. Anyone can check what *should* happen. There's a basic log of what was cooked and when.

**In software terms:**
- The build is defined in a Jenkinsfile stored in your code repository
- When a build runs, it produces a document (provenance) that says: "I was built at 3pm, from commit `a1b2c3`, on this CI server"
- That document is available to anyone who wants to check it

**What it stops:** accidental or ad-hoc changes. If someone builds from their laptop instead of CI, there's no provenance. The difference is visible.

**What it doesn't stop:** a malicious CI admin could still write a fake provenance document.

### Level 2 — Inspected kitchen with a sealed receipt

The health inspector is now involved. Every dish that leaves the kitchen has a receipt that's **stamped and signed by the inspector** — not just written by the chef. The receipt is cryptographically sealed so you'd know if anyone changed it.

**In software terms:**
- The build still runs from a Jenkinsfile in source control (same as L1)
- But now the provenance document is **cryptographically signed** by the build platform itself
- Anyone can verify: "this signature could only have been produced by the CloudBees CI server, and the document says it built artifact X from commit Y"

**What it stops:** a bad actor can't just write a fake provenance document. The signature would fail verification. Even if someone compromised the artifact, verifiers would see the signature doesn't match.

**What it doesn't stop:** a compromised CI admin who can steal the signing key. That's L3's problem.

### Level 3 — Sealed kitchen, observed chef

Now the kitchen is physically secured. The inspector watches the whole process. The recipe is locked — the chef can't deviate even if they wanted to. The signing key is held by the inspector, not the chef.

**In software terms:** the build environment is hardened; the signing key is in hardware (HSM); the build system cannot be modified by the team running it. This is where organizations with very high security requirements go.

### Level 4 — Two chefs, always

Two inspectors watch every cook. Every recipe change requires two independent approvals. Fully hermetic builds with no internet access.

---

## The three pillars (what SLSA actually checks)

SLSA doesn't just look at the build. It looks at three things:

```
SOURCE ──────► BUILD ──────► ARTIFACT
  │              │               │
  ▼              ▼               ▼
Version       Scripted &      Signed
controlled    reproducible    provenance
```

### 1. Source

**Simple version:** the recipe is written down and stored safely.

- Code lives in a version control system (Git)
- Every change is tracked with author, timestamp, and message
- At L2+: changes are reviewed before they're accepted

**Why it matters:** if the source can be changed without a trace, there's nothing to verify against. SLSA needs an immutable reference point.

### 2. Build

**Simple version:** the recipe is followed exactly, by a trusted kitchen, every time.

- The build is fully automated — no human runs manual steps
- The build runs on a trusted, hosted platform (CloudBees CI on EKS)
- Ephemeral build agents: each build gets a clean workspace, discarded when done

**Why it matters:** a human running `mvn package` on their laptop is not verifiable. A Jenkinsfile running on a managed Kubernetes pod is.

### 3. Provenance

**Simple version:** the sealed receipt that ties together what went in, what came out, and who witnessed it.

A provenance document contains:
- The artifact identity (container image digest — a unique fingerprint)
- The source it was built from (git commit SHA)
- The builder that ran the build (CloudBees CI URL)
- The time it ran
- The steps that were followed (Jenkinsfile reference)

At L2, this document is signed with a cryptographic key held by the CI platform.

---

## What "signed" actually means (the lock analogy)

Imagine you have a padlock with a unique key. You write a note, put it in a transparent lockbox, and lock it with your padlock. Anyone can read the note through the glass — but they can't change it without breaking the lock. And because your padlock is unique, anyone who knows your lock can confirm _you_ locked it.

That's a digital signature. The "padlock" is a private key (secret, held by the CI system). The "key to check if it's locked" is a public key (shared with everyone who wants to verify).

When you run `cosign verify-attestation`, you're checking: does this padlock match the public key I trust?

If yes: the provenance is authentic and untampered.  
If no: something is wrong — reject the artifact.

---

## How CloudBees CI fits in

CloudBees CI on EKS is already a hosted, managed build platform — which means the structural requirements for L2 are largely in place:

| SLSA L2 requirement | CloudBees CI reality |
|---|---|
| Hosted build service | CloudBees CI Modern on EKS = hosted ✓ |
| Version-controlled build scripts | Jenkinsfiles in Git = version controlled ✓ |
| Ephemeral build environment | Kubernetes pod agents = clean per build ✓ |
| Signed provenance | **This is the gap to close** — add cosign to the pipeline |
| RBAC on build controls | Configurable via OC RBAC — needs verification |

The L1/L2 work is mostly about adding the provenance generation and signing step to pipelines, then enforcing it at scale via CasC.

---

## Three things to remember

1. **SLSA is about the build, not just the code.** Your source code could be perfect and an attacker could still inject malware during the build. SLSA closes that gap.

2. **L2 = signed receipt from a trusted platform.** Anyone downstream can verify: this artifact came from this CloudBees CI server, from this commit, and hasn't been changed since.

3. **cosign is the tool; SLSA is the standard.** cosign is what signs and verifies. SLSA is the framework that says _what_ to sign, _what_ to include, and _what level of trust_ that signature represents.

---

## Analogy map

| Real world | SLSA concept |
|---|---|
| Restaurant inspection rating | SLSA level (1-4) |
| Written recipe stored in a binder | Jenkinsfile in SCM |
| Inspector's signed receipt | Signed provenance document |
| Tamper-evident seal on a bottle | cosign signature |
| Unique padlock only you own | Private signing key |
| Key to check the padlock | Public key |
| Container image's SHA256 | Immutable artifact identity (digest) |
| "Built in our factory, not a garage" | Hosted build platform (CloudBees CI) |
| Chain of custody document | SLSA provenance attestation |
| Ephemeral kitchen (cleaned between orders) | Kubernetes pod agent (fresh per build) |

---

## What this is NOT

- **Not a test framework.** SLSA doesn't tell you if your software works. It tells you the _build process_ can be trusted.
- **Not encryption.** The artifact isn't secret — the provenance just proves its origin.
- **Not a one-time setup.** Every build must produce provenance. It's a continuous process, not a checkbox.
- **Not just for containers.** SLSA works for JARs, binaries, npm packages — anything produced by a build. This training focuses on containers because that's Proofpoint's primary artifact type.
