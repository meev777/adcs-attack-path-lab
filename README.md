# AD CS Attack-Path Lab

An isolated Active Directory security laboratory for studying identity weaknesses, Active Directory Certificate Services (AD CS), privilege escalation paths, defensive monitoring, and remediation.

## Objectives

* Build a reproducible Windows domain environment.
* Understand the security boundaries between domain identities, authentication, and certificate services.
* Investigate relevant misconfigurations and validate their security impact in an authorized lab.
* Document observations with reproducible evidence.
* Evaluate detection opportunities and verify applicable mitigations.

## Lab Architecture

The initial environment consists of:

* **DC01:** Windows Server domain controller and DNS server.
* **CA01:** Windows Server running an Enterprise Certification Authority.
* **WS01:** Windows workstation joined to the lab domain.
* **Kali Linux:** Testing and analysis environment.

See [`diagrams/architecture.md`](diagrams/architecture.md) for the planned architecture.

## Project Status

**Phase 0 — Planning and repository setup**

The architecture, isolation requirements, evidence strategy, and build plan are being established. No attack-path result is considered verified until it has been reproduced and documented.

## Methodology

Each milestone follows this process:

1. Define the objective and prerequisites.
2. Implement the configuration.
3. Verify the result independently.
4. Capture sanitized evidence.
5. Document observations, limitations, and lessons learned.
6. Commit the reviewed changes.

## Safety and Scope

This project is intended for an isolated, authorized laboratory. Intentionally vulnerable configurations must not be exposed to production networks or public services.

Only fictional lab identities and credentials will be used. Sensitive information, private keys, passwords, VM disks, and unreviewed raw evidence must not be published.

## References and Attribution

The original project was inspired in part by external Active Directory lab material. See [`references.md`](references.md) for attribution and the distinction between reproduced concepts and independently validated work.

## Limitations

Results depend on the Windows versions, domain configuration, certificate template settings, and security controls used during testing. Findings should not be generalized to every Active Directory environment.

## Roadmap

* [ ] Phase 0: Project specification and repository setup
* [ ] Phase 1: Virtual network and domain infrastructure
* [ ] Phase 2: Enterprise CA and certificate-template configuration
* [ ] Phase 3: Controlled attack-path validation
* [ ] Phase 4: Detection, remediation, and retesting
* [ ] Phase 5: Documentation review and portfolio release

