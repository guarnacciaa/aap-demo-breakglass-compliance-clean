Breakglass Compliance Role
==========================

This role performs compliance checks for a break-glass emergency account.

Compliance Check
-----------

Checks:
- User existence
- UID range
- Shell
- Home directory
- Group membership
- Password expiration
- SSH authorized keys

Outputs:
- JSON compliance report
- Structured compliance data for AAP / EDA

This role is **read-only** and performs no remediation.


Remediation
-----------

This role also provides optional remediation capabilities via `tasks/remediate.yml`.

Remediation is **NOT executed automatically** during compliance checks.
It must be explicitly invoked.

Example:

- Compliance check only:
  ```yaml
  roles:
    - breakglass_compliance
