# QuickStart: WinRM and this role

WinRM is how Ansible talks to Windows, and it is also something this role **hardens**. Those two
facts interact: some controls can lock you out of the connection you are running over. This page
covers connecting, the controls that matter, and the failure modes worth recognizing.

## Connecting

There is no `ansible_connection` default that works for Windows - you must set it. A minimal
inventory:

```yaml
all:
  hosts:
    <inventory_hostname>:
      ansible_host: <ip-or-fqdn>
  vars:
    ansible_connection: winrm
    ansible_user: <ansible-user>
    ansible_password: <password>      # use a vault or a lookup, not plain text
    ansible_winrm_server_cert_validation: ignore
    ansible_winrm_operation_timeout_sec: 120
    ansible_winrm_read_timeout_sec: 180
```

Requires `pywinrm` in the **same Python that runs Ansible**:

```bash
python3 -c "import winrm; print('ok')"     # must be ansible's interpreter
head -1 "$(which ansible-playbook)"        # tells you which interpreter that is
```

Checking with a different `python3` than the one in ansible's shebang will mislead you.

## HTTP vs HTTPS - the most common connection failure

WinRM listens on **5985 for HTTP** and **5986 for HTTPS**. Ansible's winrm plugin defaults to the
HTTPS side. If your host only has the HTTP listener, the default configuration times out on 5986
even though the host is perfectly reachable:

```
ssl: HTTPSConnectionPool(host='<ip-or-fqdn>', port=5986):
Max retries exceeded ... ConnectTimeoutError
```

Diagnose by testing the ports directly:

```bash
nc -z -w 5 <host> 5985 && echo "HTTP  listener present"
nc -z -w 5 <host> 5986 && echo "HTTPS listener present"
```

If only 5985 answers, tell Ansible so:

```yaml
ansible_port: 5985
ansible_winrm_scheme: http
ansible_winrm_transport: ntlm
```

Or without editing inventory:

```bash
ansible-playbook -i inv site.yml \
  -e ansible_port=5985 -e ansible_winrm_scheme=http -e ansible_winrm_transport=ntlm
```

Use `ntlm`, not `basic`, over HTTP. NTLM encrypts the message payload inside the HTTP channel, so
credentials are not sent in the clear; `basic` over HTTP requires `AllowUnencrypted` on the
listener, which the STIG controls below explicitly forbid.

To add an HTTPS listener instead:

```powershell
$c = New-SelfSignedCertificate -DnsName $env:COMPUTERNAME -CertStoreLocation Cert:\LocalMachine\My
New-Item -Path WSMan:\localhost\Listener -Transport HTTPS -Address * -CertificateThumbPrint $c.Thumbprint -Force
New-NetFirewallRule -DisplayName "WinRM HTTPS" -Direction Inbound -LocalPort 5986 -Protocol TCP -Action Allow
```

## Is the host reachable at all?

A failed ping proves nothing - Windows Firewall blocks inbound ICMP by default. The quickest
reliable check on the same subnet is ARP:

```bash
arp -n <ip>
```

`(incomplete)` means nothing answered at that address, so the machine is off or absent. An entry
with a MAC means the host is up and you have a service or firewall problem instead.

## The WinRM controls, and why they can lock you out

Five controls harden WinRM itself:

| Control | CAT | Requirement |
| --- | --- | --- |
| `WN25-CC-000470` | **I** | WinRM **client** must not use Basic authentication |
| `WN25-CC-000500` | **I** | WinRM **service** must not use Basic authentication |
| `WN25-CC-000480` | II | WinRM client must not allow unencrypted traffic |
| `WN25-CC-000510` | II | WinRM service must not allow unencrypted traffic |
| `WN25-CC-000520` | II | WinRM service must not store RunAs credentials |

If you are connected over **HTTP with basic auth**, applying these ends your own session. That is
not a bug in the role - it is the control working.

`win_skip_for_test` exists for exactly this case:

```yaml
win_skip_for_test: false   # defaults/main/main.yml - apply the WinRM controls
```

Set it to `true` in inventory to skip all five while you are testing:

```yaml
win_skip_for_test: true
```

**This is a testing convenience, not a production setting.** Leaving it `true` means five
controls, two of them CAT I, are never applied. Set it back to `false` for a real run.

Note the name reads backwards: `win_skip_for_test: true` **skips** the controls. It is unprefixed
rather than `win25stig_`-prefixed, matching the rest of the Windows fleet.

## The safe way to apply them

Connect over **HTTPS (5986) with NTLM or Kerberos** before setting `win_skip_for_test: false`.
Then the controls tighten settings your session does not depend on, and nothing drops.

If you must apply them over HTTP, expect to lose the connection and plan out-of-band recovery -
console, or a second listener you are not using.

## Timeouts

The two timeout values in the sample inventory are not decorative. Several controls run
`Get-WindowsFeature`, `secedit /export` or drive-wide `win_find` calls that take a while, and the
defaults are short enough to cut them off:

```yaml
ansible_winrm_operation_timeout_sec: 120
ansible_winrm_read_timeout_sec: 180
```

`read_timeout_sec` must be **greater than** `operation_timeout_sec`, or pywinrm raises at
connection setup.

## Facts and privilege

Some Windows facts are gathered only when the connecting account is an administrator -
`ansible_facts['distribution']` among them. The role's OS gate reads it:

```yaml
- "'2025' in ansible_facts['distribution'] | default('', true)"
```

If you connect as a non-administrator, `distribution` is absent, the gate fails and the role
refuses to run. Connect as an account in the local Administrators group.

A limited `gather_subset` produces the same symptom for a different reason. This returns `None`:

```bash
ansible -i inv all -m ansible.windows.setup -a 'gather_subset=!all,!min,virtual'
```

while a full `ansible.windows.setup` populates it. If you are checking facts by hand, gather them
fully before concluding anything.

## Verifying the WinRM configuration

```powershell
winrm enumerate winrm/config/listener
winrm get winrm/config/service/auth
winrm get winrm/config/client/auth
winrm get winrm/config/service      # AllowUnencrypted should be false
```

Post-remediation, `Basic` should read `false` on both service and client, and `AllowUnencrypted`
should be `false`.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Timeout on port 5986 | host has only the HTTP listener | set `ansible_port: 5985` + `ansible_winrm_scheme: http` |
| `ModuleNotFoundError: winrm` | `pywinrm` missing from ansible's interpreter | install into the interpreter in `ansible-playbook`'s shebang |
| Connection dies mid-run | the WinRM controls applied to your own transport | `win_skip_for_test: true`, or connect over HTTPS first |
| OS gate fails, distribution empty | non-admin account, or a limited `gather_subset` | connect as an administrator; gather full facts |
| `read_timeout_sec` error at startup | read timeout not greater than operation timeout | raise `ansible_winrm_read_timeout_sec` |
| Ping fails but the host is up | Windows Firewall drops inbound ICMP | use `arp -n <ip>` or a port check instead |

## Related

- `docs/README_QuickStart_sshd.md` - the `WN25-SH-*` OpenSSH controls. Note that OpenSSH is a
  *separate* remote access path from WinRM; hardening one does not affect the other, and this role
  connects over WinRM regardless of what the SSH controls do.
