# IBM MQ in Containers — OAM Authorization Against AD Accounts

## Problem

`setmqaut` against a domain AD system account fails with a "user not found" error inside an IBM MQ container, even though the same account works with `setmqaut` on a Linux physical/virtual MQ server.

## Root Cause

On physical/virtual Linux servers, the OAM resolves a principal by checking it against the OS (`/etc/passwd`, or AD via SSSD/winbind). The MQ container image has no `/etc/passwd` entry for AD accounts and no SSSD/winbind integration. `setmqaut -p` tries to resolve the principal against the OS, finds nothing, and returns `AMQ7026E: A principal or group name was invalid.`

IBM states directly that **the MQ Operator does not allow the use of operating system users and groups within the container image** — so replicating the Linux/AD OS-account model in containers is not a supported path. Joining the container to AD via SSSD/winbind is also not viable: it requires a custom image, keytab/credential state in an ephemeral pod, and puts the deployment outside IBM's supported container configuration.

## Options for Authorizing MCAUSER IDs in Containers

### 1. `SecurityPolicy=UserExternal` (simplest, pairs directly with MCAUSER)

Introduced in MQ 9.2.1. Behaves exactly like the `User` security policy, except the username does **not** need to be known to the operating system. `setmqaut`, `SET AUTHREC`, and `dmpmqaut` then work against arbitrary strings — including the MCAUSER values already in use on channels.

Configuration (`qm.ini`, delivered via the QueueManager CR's `spec.queueManager.ini` ConfigMap):

```ini
Service:
   Name=AuthorizationService
   EntryPoints=14
   SecurityPolicy=UserExternal
```

Usage:

```
SET AUTHREC OBJTYPE(QMGR) PRINCIPAL('svcpayapp') AUTHADD(CONNECT,INQ)
SET AUTHREC PROFILE('PAY.**') OBJTYPE(QUEUE) PRINCIPAL('svcpayapp') AUTHADD(PUT,GET,BROWSE)
```

**Caveats:**
- User-only — groups are **not** supported. Every identity needs its own authority records.
- User IDs are capped at 12 characters.
- Never define a user ID with the literal value `UNKNOWN` (reserved for oversized user IDs).

### 2. LDAP Authorization Against AD (closest to the current Linux/AD model)

Define an `AUTHINFO` object of `AUTHTYPE(IDPWLDAP)` pointing at AD over LDAPS, e.g.:

```
DEFINE AUTHINFO(QM1.LDAP) AUTHTYPE(IDPWLDAP) +
  CONNAME('ad-server1.corp.com(636)') +
  AUTHORMD(SEARCHUSR) +
  FINDGRP('memberOf') +
  CLASSUSR('user') +
  CLASSGRP('group') +
  SHORTUSR('sAMAccountName') +
  NESTGRP(YES) +
  BASEDNU('ou=People,dc=corp,dc=com') +
  BASEDNG('ou=Groups,dc=corp,dc=com') +
  LDAPUSER('cn=svc_mqbind,ou=Service,dc=corp,dc=com') +
  LDAPPWD('********')
```

With this in place, MQ resolves principals via AD instead of the OS, and `setmqaut -g` against AD groups continues to work the same way it does on the physical/VM estate today.

**Caveats:**
- After running `DEFINE AUTHINFO`, the queue manager **must be restarted**, or `setmqaut` will not return the correct result.
- `SHORTUSR` must map to an AD attribute (typically `sAMAccountName`) of 12 characters or fewer.
- Define the admin group's `AUTHREC` **before** cutting over, or the queue manager can become inaccessible to administrators.

### 3. mTLS → MCAUSER Mapping (fits the CHLAUTH certificate-DN migration already underway)

Since CHLAUTH is being migrated from IP-based rules to certificate-DN rules:

1. Configure `SSLPEERMAP` in CHLAUTH to map the client certificate's Distinguished Name to a short MCAUSER value.
2. Configure the queue manager to accept authority records for a user name not known to the system (via Option 1 or Option 2 below it).
3. Authorize that resulting MCAUSER using `setmqaut`/`SET AUTHREC` as normal.

This is IBM's documented pattern for container authentication and authorization: map the DN to a user name via a channel authentication record, then ensure the queue manager will accept authority records for that unknown user name.

## Recommendation

For a Native HA, GitOps-managed estate:

- Use **Option 3 (mTLS → MCAUSER)** as the identity/authentication mechanism, since it aligns with the CHLAUTH certificate-DN migration already in progress.
- Underneath it, choose:
  - **Option 2 (LDAP/AD)** if AD groups must remain the authorization boundary and governance model.
  - **Option 1 (UserExternal)** if a flat set of per-application service identities, managed as MQSC in Git, is acceptable.

Option 1 is materially simpler to operate and removes an AD runtime dependency from the pod's startup path. Option 2 preserves existing AD group governance at the cost of that dependency (LDAPS connectivity, restart-on-config-change, `SHORTUSR` length constraints).

## What Not to Do

Do not attempt to join the container to AD using SSSD or winbind. It requires a custom (unsupported) image, keytab/credential state that does not survive pod ephemerality cleanly, and takes the deployment outside IBM's supported container configuration — the IBM MQ Operator explicitly disallows OS-level users and groups inside the container image.
