# Project Charter

## 1. Purpose

This project investigates how Active Directory identity configuration and certificate-service weaknesses can contribute to privilege escalation within a Windows domain.

The goal is to understand the underlying security mechanisms, reproduce selected scenarios in a controlled environment, and document practical detection and remediation approaches.

## 2. Research Question

How can weaknesses in Active Directory authentication, account configuration, and certificate templates combine into an attack path, and what controls can prevent or detect that path?

## 3. Objectives

* Establish a reproducible domain and certificate-services environment.
* Understand the role of AD DS, DNS, Kerberos, and AD CS.
* Investigate the prerequisites and security implications of selected attack techniques.
* Verify each finding instead of assuming that a published walkthrough will work unchanged.
* Identify relevant telemetry and evaluate mitigation strategies.
* Produce documentation suitable for technical review and internship applications.

## 4. Scope

### In scope

* A private VirtualBox laboratory.
* Fictional users, groups, hosts, and credentials.
* Active Directory configuration and certificate-template security.
* Authorized, controlled attack simulation.
* Relevant event logs and defensive controls.
* Reproducible build notes, diagrams, and sanitized evidence.

### Out of scope

* Production or third-party systems.
* Unauthorized access to real accounts or networks.
* Publishing reusable credentials, private keys, or sensitive raw evidence.
* Claims of successful exploitation without supporting evidence.

## 5. Success Criteria

The project is complete when:

1. The environment can be rebuilt from documented steps.
2. The architecture and trust relationships are explained accurately.
3. Each reported finding has documented prerequisites and validation evidence.
4. Detection and remediation claims are supported by tests or explicitly labeled as proposed.
5. The repository passes a security and attribution review.
6. The author can explain the design decisions, failures, and limitations independently.

## 6. Learning and Attribution

External tutorials may be used as learning references. Their ideas and any directly reproduced configurations will be credited. New analysis, design changes, independent validation, and defensive tests will be distinguished from reproduced material.

## 7. Current Status

Planning only. The environment and attack path have not yet been validated as part of this rebuild.

