# Evidence Register

Screenshots must be genuine, readable, and tied to a specific verification step.

| ID  | Evidence                        | Purpose                                              | Status  |
| --- | ------------------------------- | ---------------------------------------------------- | ------- |
| E01 | VirtualBox VM inventory         | Show the planned lab components                      | Pending |
| E02 | Internal Network configuration  | Demonstrate the intended network isolation           | Pending |
| E03 | DC01 network settings           | Record static IP and DNS configuration               | Pending |
| E04 | Domain verification             | Demonstrate that AD DS and DNS are operational       | Pending |
| E05 | CA01 domain membership          | Verify CA host membership in the domain              | Pending |
| E06 | Enterprise CA configuration     | Verify the CA type and configuration                 | Pending |
| E07 | Certificate template properties | Record relevant template security settings           | Pending |
| E08 | Attack-path validation          | Support a specific, controlled finding               | Pending |
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

Original sensitive evidence, if any, remains local and outside the public repository.

