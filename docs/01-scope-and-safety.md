# Scope and Safety Policy

## Environment

The lab uses QEMU/KVM with the libvirt network `ADCS-LAB`, backed by the host bridge `virbr10`.

The intended subnet is `10.10.10.0/24`. The network must be isolated with no `<forward>` element or `<forward mode='none'/>`; lab machines must not use NAT, bridged networking, macvtap, or a physical interface. The domain is `esi-sba-meev.local` and must never be connected to a real network.

## Network Restrictions

* Do not bridge intentionally vulnerable machines onto a physical network.
* Do not use libvirt NAT or expose lab services through router rules or port forwarding.
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

Create a clean snapshot after each verified configuration milestone. Keep the clean baseline (`ESI-01-BASELINE`) separate from the intentionally vulnerable state (`ESI-02-ATTACK-PATH`). Record the snapshot's purpose and the corresponding documentation commit.

## Stop Conditions

Stop testing if the lab becomes reachable from an unintended network, if real credentials enter the environment, or if the scope becomes unclear. Restore the last known-good state before proceeding.

