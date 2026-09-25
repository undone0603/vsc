# Design Note: Adversarial Conformance Testing and Fixture Structure for Offline Verification

**Author:** Zachary Kietzman (AuthiChain)
**Status:** Draft / Proposed Contribution to W3C VSC CG Repository
**Target:** Open standard conformance, test suites, and interoperability criteria

## 1. Overview and Motivation

Verifiable supply chains and Digital Product Passports (DPPs) often rely on assumptions of continuous network connectivity or centralized trust registries. However, real-world supply chain verification—particularly at border checkpoints, recycling facilities, and disconnected retail nodes—frequently occurs completely offline.

Furthermore, traditional conformance suites test only the happy path, allowing subtly broken or non-compliant implementations to pass silently if they avoid throwing unhandled exceptions.

This design note proposes an adversarial conformance model built on:

- Deliberately broken and malformed implementation fixtures to test parser and validator resilience.
- Strict reason checking (machine-readable failure codes rather than generic errors).
- Explicit ternary verdict semantics with zero numeric scoring.
- Strict offline verification guaranteeing zero server dependencies.

## 2. Core Principles

**Ternary Verdicts:** Verifiers must return one of three definitive states:

- `VALID`: All cryptographic proofs, schema requirements, and supply-chain continuity checks succeed.
- `INVALID`: A definitive cryptographic, structural, or policy violation occurred.
- `INDETERMINATE`: Insufficient verifiable context available locally (e.g., missing offline trust anchor root), preventing a definitive pass/fail.

Numeric scores (e.g., 85/100) are explicitly prohibited as they obscure actionable compliance failures.

**Strict Reason Codes:** Every non-VALID outcome must be accompanied by a standardized, machine-readable reason code (e.g., `ERR_REVOCATION_STATUS_UNKNOWN`, `ERR_SIGNATURE_EXPIRED`, `ERR_SCHEMA_VIOLATION_FIELD_X`).

**Adversarial Test Fixtures:** Conformance test runners validate verifier implementations against standardized JSON/JSON-LD bundles containing edge-case corruption (e.g., expired timestamps, mismatched supply-chain predecessor links, altered payloads, and truncated chains).

## 3. Fixture Schema Specification

A conformance fixture is packaged as a JSON document containing the test metadata, the input verifiable credential / supply-chain packet, and the expected verifier outcome.

Example Fixture Structure (`fixture-vsc-001.json`):

> **Proposed placeholders:** `https://w3id.org/vsc/conformance/v1/fixture.schema.json` and `https://w3id.org/vsc/v1` are proposed placeholder URLs. They are not registered and do not resolve yet.

```json
{
  "$schema": "https://w3id.org/vsc/conformance/v1/fixture.schema.json",
  "testId": "VSC-ADV-042",
  "description": "Reject credential with an expired validity window under strict offline mode",
  "profile": "offline-strict",
  "input": {
    "credential": {
      "@context": [
        "https://www.w3.org/2018/credentials/v1",
        "https://w3id.org/vsc/v1"
      ],
      "id": "urn:uuid:6ba7b810-9dad-11d1-80b4-0c0411d3047a",
      "type": ["VerifiableCredential", "DigitalProductPassportCredential"],
      "issuer": "did:key:z6MksH1...",
      "issuanceDate": "2024-01-01T00:00:00Z",
      "expirationDate": "2025-01-01T00:00:00Z",
      "credentialSubject": {
        "id": "urn:epcid:sgtin:0614141.107346.2017",
        "batchNumber": "B-99887",
        "predecessorHash": "sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
      },
      "proof": {
        "type": "Ed25519Signature2020",
        "created": "2024-01-01T00:05:00Z",
        "proofPurpose": "assertionMethod",
        "verificationMethod": "did:key:z6MksH1...#z6MksH1...",
        "proofValue": "z3u7...abc"
      }
    },
    "environment": {
      "currentTimestamp": "2026-09-25T00:00:00Z",
      "allowNetworkAccess": false
    }
  },
  "expected": {
    "verdict": "INVALID",
    "reasonCode": "ERR_CREDENTIAL_EXPIRED",
    "strictReasonMatch": true
  }
}
```

## 4. Conformance Test Suite Workflow

1. **Test Runner Initialization:** The conformance test suite loads the library of adversarial fixtures (`/fixtures/*.json`).
2. **Execution:** The target verifier implementation processes each fixture under the specified environmental constraints (`allowNetworkAccess: false`).
3. **Assertion:** The test runner evaluates:
   - Did the verdict match the expected ternary state (`VALID`, `INVALID`, `INDETERMINATE`)?
   - Does the returned `reasonCode` match the expected standardized string code?
4. **Reporting:** Generates an interoperability matrix showing adherence to the VSC baseline specification.

## 5. Next Steps for the CG

- Review and adopt the JSON schema for test fixtures in the `w3c-cg/vsc` repository.
- Populate initial test vectors covering common supply chain failure modes (revoked roots, broken chain hashes, clock skew).
- Integrate into the `tools/test-suites/` workstream.
