---
title: "Automating OpenShift certificate renewals without breaking production"
date: 2026-02-01
---

Renewing certificates in a Kubernetes/OpenShift environment looks simple until it isn't: secrets are rotated, pods restart, Java keystores complain, and what should have been maintenance becomes an incident.

This post is a template you can adapt. It’s written from an operations perspective: reduce risk, make the change observable, and keep rollback cheap.

## Typical failure modes

1) **Certificate updated, but the workload doesn’t reload**
Some applications only read certs at startup (common with JVM stacks). Updating a secret is not enough; you need a controlled rollout.

2) **Wrong key/cert pairing**
A frequent mistake is shipping a certificate that doesn't match the private key, especially when pipelines mix artifacts.

3) **Intermediate chain missing**
Browsers may still work while internal clients fail in stricter TLS/mTLS paths. Always validate the full chain.

4) **Keystore formats drift**
You receive PEM, the app expects JKS or PKCS12, and conversions happen “somewhere” without traceability.

## A safer approach: treat certificates as assets

Instead of “renew and pray”, model each certificate as an *asset* with explicit metadata:

- namespace
- secret name + key(s)
- cert format (PEM/JKS/PKCS12)
- password reference (if needed)
- owner application
- rollout strategy (restart, hot reload, none)
- validation steps

That single idea turns renewals into an inventory problem — and inventories are automatable.

## Minimal runbook (operator-friendly)

1) **Pre-check**
- Identify impacted routes/ingress + internal clients.
- Confirm if the app needs restart.
- Validate expiry dates and CN/SAN requirements.

2) **Stage artifacts**
- New cert + key + chain in a staging location (git or secure store).
- Verify pair match:
  - modulus match (RSA) or equivalent checks for ECDSA.
- Verify chain completeness.

3) **Update secret**
- Apply secret update in a controlled window.
- Prefer `apply` over ad-hoc edits for traceability.

4) **Rollout**
- If restart is required: rollout deployment with maxUnavailable=0 where possible.
- Watch readiness/liveness, logs, and upstream probes.

5) **Post-check**
- External validation (TLS handshake, expiry, chain).
- Internal validation (mTLS paths if present).
- Document: what changed, where, and what was observed.

6) **Rollback**
- Keep the previous secret material for a defined grace period.
- Rollback should be “swap secret back + rollout”.

## What I automate first

If you build only one automation, make it a validator:

- parse a YAML inventory of cert assets
- check required fields per cert type
- verify secrets exist (or can be created)
- ensure referenced key names exist
- confirm keystore password refs are present when required

Once validation is stable, automation can evolve into:
- rendering secrets from encrypted inputs
- renewing via ACME/internal CA
- controlled rollouts per asset

## Closing thought

Certificates are not “just files”. In production they are *live dependencies*.
Treat them like code: version them, validate them, and deploy them with guardrails.

(Replace this post with your real process and tooling as you evolve your repo.)
