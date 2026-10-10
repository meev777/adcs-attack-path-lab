# Planned Lab Architecture

## Logical Topology

```mermaid
flowchart TB
    subgraph LAB["VirtualBox Internal Network: ADCS-LAB"]
        DC["DC01<br/>Windows Server<br/>AD DS + DNS"]
        CA["CA01<br/>Windows Server<br/>Enterprise CA"]
        WS["WS01<br/>Windows Workstation"]
        KALI["Kali Linux<br/>Testing and analysis"]
        WS --> DC
        CA --> DC
        KALI --> DC
        WS --> CA
    end
    HOST["Kali Host / VirtualBox"]
    HOST -. "VM management<br/>not guest network access" .- LAB
```

## Address Plan

| Host             | Address        | Role                               |
| ---------------- | -------------- | ---------------------------------- |
| DC01             | 10.10.10.10/24 | Domain controller and DNS          |
| CA01             | 10.10.10.20/24 | Enterprise Certification Authority |
| WS01             | 10.10.10.30/24 | Domain workstation                 |
| Kali VM, if used | 10.10.10.40/24 | Testing and analysis               |

Domain name: `lab.example.test`

## Network Notes

The topology is a design, not a verified implementation. Guest adapters will use VirtualBox Internal Network mode. The lab has no default gateway unless a later, explicitly reviewed requirement justifies one.

DNS on domain members should point to DC01. Domain joining and Enterprise CA configuration will be verified during the build.

## Revision History

* Phase 0: Initial proposed topology.
* Later revisions must record actual configuration changes and their rationale.

