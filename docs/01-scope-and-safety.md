# Scope and Safety Policy

## Environment

The lab uses VirtualBox with a dedicated Internal Network named `ADCS-LAB`.

The intended subnet is `10.10.10.0/24`. Lab machines must not use bridged networking.

## Network Restrictions

* Do not bridge intentionally vulnerable machines onto a physical network.
* Do not expose lab services through router rules or port forwarding.
* Do not add a second network adapter to a vulnerable VM without documenting and reviewing the isolation implications.
* Disable shared clipboard, drag-and-drop, and shared folders for lab guests.
* Do not use real enterprise credentials or identities.

## Data Handling

* Do not publish passwords, password hashes that have not been reviewed, private keys, PFX/P12 files, access tokens, or recovery material.
* Do not commit VM disks, snapshots, memory dumps, or unreviewed log bundles.
* Review screenshots for personal information, hostnames, addresses, and accidental secrets.
* Treat evidence files as sensitive until reviewed.
* Remember that Git history can retain material removed from the latest version.

## Testing Boundaries

All testing is restricted to the lab systems and identities defined in the project scope.

Unexpected connectivity outside the lab is a reason to stop and investigate before continuing.

## Evidence Integrity

Screenshots and logs should represent actual observations. Do not fabricate, alter, or combine evidence in a way that misrepresents a result. Sanitized copies may be published when the original evidence is preserved privately.

## Recovery

Create a clean snapshot after each verified configuration milestone. Record the snapshot's purpose and the corresponding documentation commit.

## Stop Conditions

Stop testing if the lab becomes reachable from an unintended network, if real credentials enter the environment, or if the scope becomes unclear. Restore the last known-good state before proceeding.

