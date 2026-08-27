# Argus

**A reusable, wallet-owned KYC attestation registry on Arkiv — verify once, and every DAO’s hundred eyes see the same tamper-proof, time-scoped proof of personhood before a vote counts.**

> Submitted to the [Arkiv Ideathon #4 — “What can YOU [ARKIV]?”](https://ideathon.arkiv.network) — Other track.
> This is an idea + data model submission — no code deployed, per the ideathon’s rules.

-----

## The problem

DAOs need real humans behind votes to resist sybil attacks — one person spinning up dozens of wallets to swing governance. Today every DAO either:

- skips verification entirely and eats the sybil risk, or
- bolts on its own KYC flow, creating friction, cost, and a new centralized data silo per DAO.

There’s no shared, portable layer that says *“this wallet passed KYC, here’s who attested it, here’s when it expires”* — something multiple DAOs can independently check without trusting each other’s databases or re-verifying the same human over and over.

Named after **Argus Panoptes**, the hundred-eyed giant of Greek myth whose eyes never all slept at once — nothing could slip past him unseen.

## Scope — the first slice

A weekend build:

- One attestor role writes an `ArgusAttestation` entity per wallet.
- One DAO integration: a query that gates vote-casting on `isVerified(wallet)`.

That’s the whole slice — no attestor marketplace, no cross-DAO reputation, no dispute flow yet.

**Honest risks / open problems (intentionally out of scope for this slice):**

- Attestor trust is centralized to whoever gets whitelisted first — Argus doesn’t yet solve *“who watches the watchers.”*
- A liveness check can’t stop someone from getting verified once and renting/selling that wallet’s access to others (sybil-by-proxy rather than sybil-by-volume).

## Entity & data model

Argus writes a single entity type: **`ArgusAttestation`**.

**Indexed attributes** (queryable):

|Attribute          |Type     |Description                                       |
|-------------------|---------|--------------------------------------------------|
|`walletAddress`    |address  |The subject being verified                        |
|`attestorId`       |string   |Which accredited KYC provider issued this         |
|`verificationLevel`|enum     |`"liveness-only"` | `"liveness+id"` | `"full-kyc"`|
|`issuedAt`         |timestamp|When the attestation was written                  |

**Payload attributes** (not indexed):

|Attribute        |Type              |Description                           |
|-----------------|------------------|--------------------------------------|
|`expiresIn`      |int (seconds)     |Default `7,776,000` (90 days)         |
|`riskFlags`      |string[], optional|e.g. `"duplicate-biometric-suspected"`|
|`attestationHash`|bytes32           |Hash of the off-chain provider result |

No PII, biometric data, or ID documents ever enter the entity. Argus stores the **claim**, never the **evidence**.

## Queries a DAO relies on

```
// Core eligibility check before a vote counts
isVerified(wallet):
  where(eq("walletAddress", W)).where(gt("expiresAt", now))

// Stricter DAOs — trusted attestors / verification tier only
where(eq("attestorId", X)).where(eq("verificationLevel", "full-kyc"))

// Audit / dispute — full attestation history for a wallet
where(eq("walletAddress", W)).orderBy("issuedAt", desc)

// Governance analytics — verified-voter count at a past snapshot
count(
  where(eq("verificationLevel", X))
    .where(gt("expiresAt", snapshotTime))
)
```

## Expiry, extension & ownership

- **Expiry is the trust-decay mechanism.** Attestations default to 90 days via `expiresIn` — a “verified” wallet automatically stops being verifiable once the underlying check goes stale. No DAO has to run a cron job to purge old claims.
- **Extension means re-verification, not renewal.** An attestor writes a fresh entity rather than editing the old one, so history stays intact for audits.
- **Ownership is dual.** Only the attestor’s wallet can create an attestation, and only that same wallet can revoke or update it — so verification status can’t be forged or silently altered by a DAO or third party.

## Why Arkiv, not a plain database?

A plain database makes one party the source of truth for “who’s verified” — they can revoke, alter, or lose that data, and every DAO has to trust their uptime and honesty blindly.

Arkiv makes each attestation independently checkable against Ethereum (creator, block, tx), wallet-owned so no company can rewrite it, and queryable across DAOs without a shared backend or API key. Expiry as a built-in primitive is the real unlock: “verified” is a claim that should decay, and Arkiv makes that decay automatic and structural instead of something every DAO reimplements badly on its own.

## What stays off Arkiv

- The actual KYC process — camera capture, liveness challenge, ID-document OCR, face-match scoring — runs off-chain with the attestor’s own provider.
- Raw biometrics, ID scans, and any PII never touch Arkiv; only the `attestationHash` and metadata do.
- Vote execution and DAO governance logic stay off Arkiv — Argus is only the eligibility check a DAO’s contract or frontend calls before a vote counts.

-----

*Built for [Arkiv Ideathon #4](https://ideathon.arkiv.network) · [docs.arkiv.network](https://docs.arkiv.network)*