# Jivora — security platform test bed · Veracode

> ⚠️ **Intentionally vulnerable.** This repository exists solely to evaluate
> security scanning platforms. Do not deploy it, do not reuse its code, and do
> not treat any credential in it as real.

**Platform under test:** [Veracode](https://www.veracode.com) — Static Analysis
(Pipeline Scan and Policy Scan) plus Software Composition Analysis.

## Enabling it

Veracode is the one platform here with no self-serve free tier; you will need a
trial or an existing subscription.

1. Get API credentials from the Veracode platform
   (*Account → API Credentials*).
2. Add repository secrets:
   - `VERACODE_API_ID` and `VERACODE_API_KEY` for static analysis
   - `SRCCLR_API_TOKEN` for agent-based SCA
3. The workflow in `.github/workflows/veracode.yml` packages the monorepo into
   an archive and runs **Pipeline Scan** on every push, **Policy Scan** on
   `main`, and SCA separately.

Run both scan types. Pipeline Scan is the fast PR-time variant; Policy Scan is
the slow one that produces the compliance artefact — and the difference between
them is most of what Veracode's price buys.

## What to watch for

Veracode is the enterprise/compliance play, so evaluate it on that axis rather
than on raw finding count:

- **Policy gating.** Can you express "no High or above ships" as a policy, have
  it fail the build, and produce evidence of it? Jivora handles PHI under HIPAA;
  this is the capability that matters if you ever sell to a covered entity that
  demands third-party attestation.
- **Compliance reporting.** Does it produce an artefact you could hand to an
  auditor or attach to a BAA negotiation? None of the other four really do.
- **Scan time.** Time the Policy Scan honestly and decide whether it fits a
  normal development loop, or whether it becomes a nightly job people learn to
  ignore.
- **JavaScript/TypeScript depth.** Veracode's heritage is compiled languages.
  This repository is entirely JS/TS across Express and two Next.js apps —
  measure recall against `FINDINGS.md` rather than assuming enterprise pricing
  buys enterprise coverage in this ecosystem.
- **Monorepo handling.** Everything ships as one archive. Watch whether findings
  come back attributed to the right application, or flattened into one blob.
- **Blind spots to confirm:** Veracode has no first-class IaC or container
  posture scanning in the SAST/SCA products. The 27 findings across
  `infra/terraform/`, `infra/k8s/` and `docker-compose.yml` are the test — and
  the 8 committed-secret findings in category H.

## Cost note

Expect a sales conversation, an annual contract and a floor well above the rest
of this field. The question is not whether Veracode is more expensive — it is —
but whether the compliance artefact is worth the difference for a platform of
Jivora's size. Record the answer in `SCORECARD.md §4`.
---

## What this repository is

A deliberately vulnerable, structurally faithful miniature of **Jivora** — a
Lyme-disease wellness platform handling PHI under HIPAA. It exists to be scanned.

The shape is copied from the real system's architecture document (v1.4), so
findings land on the same surfaces they would in production: a monorepo of three
applications over one PostgreSQL database, two API surfaces with different
authentication, PHI column encryption, an AI assistant with a read-only tool
loop, Stripe webhooks, and a Docker Compose deployment.

```
Jivora-backend/     Node 22 · Express 4 · Sequelize · Socket.IO · @anthropic-ai/sdk
jivora-web/         Next.js App Router · member app · port 4000
jivora-admin/       Next.js App Router · admin panel · port 4002 · Yarn only
infra/terraform/    The AWS deployment the platform would move to
infra/k8s/          The Kubernetes topology for the same
docker-compose.yml  The production stack as actually deployed today
.env.prod           Committed on purpose
```

### Architectural fidelity

| Real Jivora | Here |
| --- | --- |
| Two account stores (`user`, `admin_user`), seven roles | Both, with the same role ids |
| `/api/web/*` and `/api/*` registered separately | Both registrars present |
| PHI column encryption + `*_hash` lookup columns | Present — and subtly wrong |
| Entitlements recomputed live, 402 on paywall | `lib/web/entitlements.js` |
| Admin route manifest as single source of truth | `lib/admin/route-manifest.js` + `route-access.js` |
| 13 read-only assistant tools bound to `user.id` | All 13 — one is not bound |
| Stripe webhook signature mandatory | Claimed in docs, not in code |
| In-process scheduler, no separate worker | `lib/web-protocol/reminder-scheduler.js` |

Where the code contradicts the architecture document, that is the point: the
most interesting findings in this repository are the ones where a stated
security control does not survive contact with the implementation.

## Running it

Don't. There is no seed data, and several dependencies are pinned to versions
with known remote-code-execution advisories. If you want a container for
image-layer scanning:

```sh
docker build -t jivora/backend:sectest Jivora-backend
docker build -t jivora/web:sectest     jivora-web
```

`npm install` inside any of the three apps is enough for dependency scanning and
is safe as long as nothing is executed.

## Scoring a platform

1. Connect the platform. Do not read `FINDINGS.md` first.
2. Let it scan everything it is willing to scan.
3. Export its findings.
4. Now open [`FINDINGS.md`](FINDINGS.md) — the answer key: 57 code and config
   issues across 7 categories, plus 25 dependencies chosen for known advisories.
5. Fill in [`SCORECARD.md`](SCORECARD.md).

Four multi-step attack chains are planted deliberately. Whether a platform
surfaces them *as chains* rather than as unrelated findings is the sharpest
differentiator in this repository.

## Sibling repositories

The same codebase, scanned by a different platform in each:

| Repository | Platform |
| --- | --- |
| [`jivora-sectest-github-advanced-security`](https://github.com/beeva/jivora-sectest-github-advanced-security) | GitHub Advanced Security |
| [`jivora-sectest-aikido`](https://github.com/beeva/jivora-sectest-aikido) | Aikido |
| [`jivora-sectest-snyk`](https://github.com/beeva/jivora-sectest-snyk) | Snyk |
| [`jivora-sectest-veracode`](https://github.com/beeva/jivora-sectest-veracode) | Veracode |
| [`jivora-sectest-wiz`](https://github.com/beeva/jivora-sectest-wiz) | Wiz |

Identical content by design — only this README and the workflows in
`.github/workflows/` differ, so the results are directly comparable.

## Safety

Every credential in this repository is synthetic and authenticates against
nothing. `AKIAIOSFODNN7EXAMPLE` is AWS's own published documentation key. There
is no real PHI here and no real patient data; the clinical vocabulary is
structural, and every value is invented.

GitHub push protection is disabled on this repository on purpose — the committed
`.env.prod` is one of the things being tested for.

Do not copy any pattern from this repository into anything real.
