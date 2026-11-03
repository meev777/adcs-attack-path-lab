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

* **MEEV-DC01:** Windows Server domain controller and DNS server for `esi-sba-meev.local`.
* **CA01:** Windows Server running an Enterprise Certification Authority (`MEEV-ROOT-CA`).
* **MS01:** Windows workstation joined to the lab domain.
* **Kali Linux:** Testing and analysis environment on the QEMU/KVM host.

The VMs use the isolated libvirt network `ADCS-LAB` on `10.10.10.0/24` with the host bridge `virbr10`. See [`diagrams/architecture.md`](diagrams/architecture.md) for the architecture and [`planing/attack-path.png`](planing/attack-path.png) for the target path. The manual build is in [`planing/00-build-runbook.md`](planing/00-build-runbook.md); the controlled vulnerability build is in [`planing/01-attack-path-build.md`](planing/01-attack-path-build.md).

## Project Status

**Phase 1 — Domain and network infrastructure (in progress)**

The QEMU/KVM lab network, domain, and DC are built. The next step is creating the attack-path identities and groups, followed by the CA and workstation. No attack-path result is considered verified until it has been reproduced and documented.

## Methodology

Each milestone follows this process:

1. Define the objective and prerequisites.
2. Implement the configuration.
3. Verify the result independently.
4. Capture sanitized evidence.
5. Document observations, limitations, and lessons learned.
6. Commit the reviewed changes.

## Safety and Scope

This project is intended for an isolated, authorized laboratory. Intentionally vulnerable configurations must not be exposed to production networks or public services. The `esi-sba-meev.local` domain exists only on the isolated libvirt network; it must never be connected to a real network.

Only fictional lab identities and credentials will be used. Sensitive information, private keys, passwords, VM disks, and unreviewed raw evidence must not be published.

## References and Attribution

The original project was inspired in part by external Active Directory lab material. See [`references.md`](references.md) for attribution and the distinction between reproduced concepts and independently validated work.

## Limitations

Results depend on the Windows versions, domain configuration, certificate template settings, and security controls used during testing. Findings should not be generalized to every Active Directory environment.

## Roadmap

* [x] Phase 0: Project specification and repository setup
* [ ] Phase 1: QEMU/KVM network and domain infrastructure (in progress)
* [ ] Phase 2: Enterprise CA and certificate-template configuration
* [ ] Phase 3: Controlled attack-path validation
* [ ] Phase 4: Detection, remediation, and retesting
* [ ] Phase 5: Documentation review and portfolio release

