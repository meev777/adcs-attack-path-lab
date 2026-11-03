# Evidence Register

Screenshots must be genuine, readable, and tied to a specific verification step.

| ID  | Evidence                        | Purpose                                              | Status  |
| --- | ------------------------------- | ---------------------------------------------------- | ------- |
| E01 | QEMU/KVM VM inventory           | Show MEEV-DC01, CA01, MS01, and the libvirt network  | Pending |
| E02 | libvirt isolated network        | Demonstrate no NAT/route forward and no DHCP         | Pending |
| E03 | MEEV-DC01 network settings      | Record static IP and DNS configuration               | Pending |
| E04 | Domain verification             | Demonstrate that AD DS and DNS are operational       | Pending |
| E05 | CA01 domain membership          | Verify CA host membership in the domain              | Pending |
| E06 | Enterprise CA configuration     | Verify the CA type and configuration                 | Pending |
| E07 | Certificate template properties | Record relevant template security settings           | Pending |
| E08 | Attack-path validation          | Support the controlled path build                    | Pending |
| E09 | Detection evidence              | Show relevant logs or alert behavior                 | Pending |
| E10 | Remediation retest              | Demonstrate the observed effect of a security change | Pending |

## Capture Procedure

For every evidence item:

1. Confirm the system and configuration being examined.
2. Perform the relevant verification.
3. Capture the screen without hiding relevant context.
4. Record the date, VM, objective, and observation in the associated report.
5. Review the image for sensitive information.
6. Publish only the sanitized version.

Do not mark an item complete merely because a configuration wizard finished. Verify the resulting configuration first.

## Current Notes

- `screenshots/DC01/s1-domaininfos.png` shows `MEEV-DC01`, `esi-sba-meev.local`, `10.10.10.10/24`, gateway `10.10.10.1`, and DNS `127.0.0.1`.
- `screenshots/DC01/s3-test-result.png` shows a DNS `Forw` failure with `8.8.8.8` and IPv6 root hints. Remove the public forwarder, set DNS to `10.10.10.10`, rerun `dcdiag /test:dns /v`, and recapture before marking E03/E04 complete.
- `screenshots/E01-host-only-network.png` is an obsolete VirtualBox capture; replace it with QEMU/KVM and libvirt evidence.
- The CA screenshots document an earlier role-install attempt and are not proof of an Enterprise CA.
- The current libvirt network is active and isolated: no `<forward>` element, no DHCP, and `virbr10` has `10.10.10.1/24`. The bridge shows `DOWN` while all guests are shut down; that is normal.
- The current libvirt domain labels are `ESCLAB-DC0`, `ESCLAB-CA0`, and `ESCLAB-PC)`. The Windows names must remain `MEEV-DC01`, `CA01`, and `MS01`; rename the libvirt labels only while the domains are shut off. The existing `DC01-02-ADDS-VERIFIED` snapshot is a running checkpoint, not the clean `ESI-01-BASELINE`.

Original sensitive evidence, if any, remains local and outside the public repository.

