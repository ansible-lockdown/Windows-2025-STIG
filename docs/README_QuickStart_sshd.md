# QuickStart: OpenSSH (`WN25-SH-*`) on Windows Server 2025

The `SH` band is new in the Windows Server 2025 STIG. Earlier Windows STIGs had no OpenSSH
controls at all, so nothing in the Ansible Lockdown Windows fleet did this before. This page
explains how the role handles it and what to expect on a real host.

## The short version

- OpenSSH on Windows lives in `C:\ProgramData\ssh`.
- **`sshd_config` and the host keys do not exist until sshd starts for the first time.** Having
  the OpenSSH capability installed is not the same as having it configured.
- The role therefore keeps **two** facts, and the hardening controls gate on the second.
- Nothing installs OpenSSH unless you opt in.

## The 11 controls

| Control | CAT | What it does | Module |
| --- | --- | --- | --- |
| `WN25-SH-000010` | II | Install OpenSSH Server, set `sshd` Automatic and start it | `win_shell` + `win_service` |
| `WN25-SH-000020` | II | Deploy the DOD banner and set the `Banner` directive | `win_template` x2 |
| `WN25-SH-000030` | II | `PubkeyAuthentication yes` | `win_template` |
| `WN25-SH-000040` | **I** | `PermitEmptyPasswords no` | `win_template` |
| `WN25-SH-000050` | II | `RekeyLimit` | `win_template` |
| `WN25-SH-000060` | II | `ClientAliveCountMax` | `win_template` |
| `WN25-SH-000070` | II | `ClientAliveInterval` | `win_template` |
| `WN25-SH-000080` | II | `sshd_config` permissions | `win_acl_inheritance` + `win_acl` |
| `WN25-SH-000090` | II | Private host key permissions | `win_find` + `win_acl` |
| `WN25-SH-000100` | II | Public host key permissions | `win_find` + `win_acl` |
| `WN25-SH-000110` | II | `GSSAPIAuthentication no` | `win_template` |

`SH-000040` is the only CAT I control in the band, so it lives in `tasks/Cat1/`. The other ten
are in `tasks/Cat2/`.

## Installation is opt-in

`WN25-SH-000010` installs OpenSSH Server and starts `sshd`. That adds a **listening network
service** to the host, so it does not run by default:

```yaml
win25stig_install_openssh: false   # defaults/main/main.yml
```

Set it to `true` where the requirement genuinely applies. Leaving it `false` does not disable the
other ten controls - they still harden an OpenSSH that is already present.

## The two facts, and why there are two

`tasks/prelim.yml` sets both:

| Fact | Means | Set from |
| --- | --- | --- |
| `prelim_openssh_installed` | the `sshd` service is registered | `win_service_info` |
| `prelim_openssh_configured` | `sshd_config` exists on disk | `win_stat` |

**The hardening controls gate on `prelim_openssh_configured`, not on `prelim_openssh_installed`.**

This distinction is not academic. A host can have the OpenSSH capability installed, the `sshd`
service registered as `Stopped / Manual`, and `C:\ProgramData\ssh` present but **completely
empty**. Windows generates `sshd_config` and the host keys when sshd *first starts*. Gating on
the service alone means every configuration control fails with:

```
Path C:\ProgramData\ssh\sshd_config does not exist !
```

When OpenSSH is installed but unconfigured, prelim emits a warning rather than failing silently:

```
Warning!! The sshd service is present but C:\ProgramData\ssh\sshd_config does not exist,
so OpenSSH has never been started and has generated no configuration or host keys.
The WN25-SH-* hardening controls will be skipped. Start sshd once, or set
win25stig_install_openssh to true, then run the role again to harden it.
```

To get the controls to run on such a host, start sshd once:

```powershell
Set-Service -Name sshd -StartupType Automatic
Start-Service sshd
```

That produces `sshd_config`, `sshd.pid`, `logs\`, and three key pairs - `ssh_host_rsa_key`,
`ssh_host_ecdsa_key`, `ssh_host_ed25519_key` and their `.pub` counterparts.

## Ordering caveat

`tasks/Cat1` runs before `tasks/Cat2`. `SH-000040` is CAT I and therefore in Cat1, while
`SH-000010` - the control that may install OpenSSH - is in Cat2.

So on a host where the **same run** installs OpenSSH for the first time, `sshd_config` does not
yet exist when `SH-000040` is evaluated, and it skips. `SH-000010` refreshes both facts after
starting sshd, so the remaining Cat2 controls do apply in that run. **Re-run the role** to pick up
`SH-000040`.

On a host that already has OpenSSH configured, everything applies in a single run.

## Restarting sshd

Windows OpenSSH reads `sshd_config` only at start, and has no reload verb. Every control that
edits the file notifies the `Restart_sshd` handler.

If your remediation runs **over the SSH connection sshd is serving**, that restart will drop you.
Suppress it and restart out of band:

```yaml
win25stig_skip_sshd_restart: true
```

The role's own connection is WinRM, not SSH, so this only matters if you have repointed Ansible
at the SSH transport.

## Tunables

```yaml
win25stig_install_openssh: false                       # SH-000010 gate
win25stig_sshd_config_path: 'C:\ProgramData\ssh\sshd_config'
win25stig_ssh_data_dir: 'C:\ProgramData\ssh'
win25stig_ssh_banner_path: 'C:\ProgramData\ssh\Banner.txt'
win25stig_ssh_rekeylimit: '1G 1h'                      # SH-000050
win25stig_ssh_clientalivecountmax: 1                   # SH-000060
win25stig_ssh_clientaliveinterval: 600                 # SH-000070
win25stig_skip_sshd_restart: false

# Windows OpenSSH defaults reproduced by the template - not mandated by the benchmark
win25stig_sshd_authorizedkeysfile: '.ssh/authorized_keys'
win25stig_sshd_sftp_subsystem: 'sftp-server.exe'
win25stig_sshd_allowgroups: 'administrators "openssh users"'
win25stig_sshd_admin_authorizedkeysfile: '__PROGRAMDATA__/ssh/administrators_authorized_keys'
```

## The role owns sshd_config

`sshd_config` is rendered whole from `templates/sshd_config.j2`, not edited directive by
directive. **Anything you add to that file by hand is replaced on the next run.**

Two reasons it works this way:

- The STIG check for every configuration control greps the base file
  (`Get-Content "$env:ProgramData\ssh\sshd_config" | Select-String ...`). A `sshd_config.d`
  drop-in - which is how the Linux roles do it - would be honored by sshd but would **not**
  satisfy the documented check procedure.
- A whole-file render cannot edit the wrong occurrence of a directive. Line-based editing changes
  the first match, which is wrong when the same directive also appears inside a `Match` block.

Because the file is rendered whole, the template also reproduces four directives Windows ships
that the benchmark does not mandate. They are variables, not literals, so override them rather
than editing the template:

| Variable | Why it matters |
| --- | --- |
| `win25stig_sshd_admin_authorizedkeysfile` | the `Match Group administrators` block - losing it breaks administrator public key authentication |
| `win25stig_sshd_allowgroups` | controls who may log in at all |
| `win25stig_sshd_sftp_subsystem` | losing it breaks sftp |
| `win25stig_sshd_authorizedkeysfile` | per-user key location |

Each STIG directive is gated on its own control toggle, so setting `wn25_sh_000110: false` omits
`GSSAPIAuthentication` from the rendered file rather than pinning it to a value.

One consequence of one file, one write: the CAT II deployment task carries **six** control
identifiers rather than the role's usual one. `SH-000040` is CAT I, so it deploys the same
template from `tasks/Cat1/` - that way a `--tags CAT1` run still hardens the file. Both renders
produce identical output.

The banner is rendered from `templates/ssh_banner.txt.j2`, which composes the **same two
variables the interactive Windows logon notice uses**, so the SSH banner and the console banner
cannot drift apart:

| Variable | Role in the banner | Also used by |
| --- | --- | --- |
| `win25stig_legalnoticecaption` | the title line | `WN25-SO-000140` (`LegalNoticeCaption`) |
| `win25stig_legalnoticetext` | the body | `LegalNoticeText` |

The template carries nothing else - no provenance header, no attribution. This matches the Linux
roles, whose SSH banner templates (`templates/etc/issue.net.j2`) likewise render only the banner
variable. The banner is displayed *before* authentication, so anything added to it is visible to
unauthenticated clients, and the DOD notice is mandated text.

Override either one to rebrand both banners at once. The title must remain one of the values
`WN25-SO-000140` accepts - "DOD Notice and Consent Banner", "US Department of Defense Warning
Statement", or an organization-defined equivalent - because that control checks the title text.

## A note on the permission controls

`SH-000080`, `SH-000090` and `SH-000100` disable ACL inheritance before applying the required
ACEs, using:

```yaml
ansible.windows.win_acl_inheritance:
  state: absent
  reorganize: false
```

`reorganize: false` is deliberate and load-bearing. Per the module documentation, for
`state=absent` the flag "indicates if the inherited ACEs should be **copied from the parent**".
With `reorganize: true`, an inherited `BUILTIN\Users:(RX)` would be converted into a *permanent
explicit* ACE - which is the opposite of what these controls require, and the role itself would
be what made the over-permissive entry stick. `false` discards the inherited ACEs so only the
STIG-required ones remain.

On a freshly generated OpenSSH install, `sshd_config` typically carries three **inherited** ACEs
(SYSTEM, Administrators, Authenticated Users), while the host keys already carry **explicit**
`SYSTEM:F` and `Administrators:F`. So `SH-000080` usually changes something and `SH-000090`
usually does not.

Required end state:

| File | Principals |
| --- | --- |
| `sshd_config` | `NT AUTHORITY\SYSTEM` F, `BUILTIN\Administrators` F, `NT AUTHORITY\Authenticated Users` RX |
| private host keys | `BUILTIN\Administrators` F, `NT AUTHORITY\SYSTEM` F |
| public host keys | `BUILTIN\Administrators` F, `NT AUTHORITY\SYSTEM` F |

The host-key controls enumerate keys with `win_find` and loop, so they cover every algorithm
present and no-op cleanly when there are none.

## Running just the SSH controls

Every control carries an `ssh` tag as well as its rule ID, so:

```bash
# the whole band
ansible-playbook -i <inventory> site.yml --tags ssh

# one control
ansible-playbook -i <inventory> site.yml --tags WN25-SH-000040

# everything except SSH
ansible-playbook -i <inventory> site.yml --skip-tags ssh
```

Tag selection works because the role uses static `import_tasks` throughout.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| All SH controls skip, prelim warns about `sshd_config` | OpenSSH installed but never started | start sshd once, then re-run |
| `Path ...sshd_config does not exist !` | running an older copy of this role that gated on the service | update; hardening gates on `prelim_openssh_configured` |
| `SH-000040` skipped but the Cat2 controls applied | OpenSSH was installed by this same run | re-run the role |
| Connection drops mid-run | `Restart_sshd` fired on the connection in use | set `win25stig_skip_sshd_restart: true` |
| No host keys found by `SH-000090` / `SH-000100` | sshd has not generated them yet | start sshd once |

## Verifying by hand

```powershell
Get-Service sshd | Select-Object Status, StartType
Test-Path C:\ProgramData\ssh\sshd_config
Select-String -Path C:\ProgramData\ssh\sshd_config `
  -Pattern '^\s*(PermitEmptyPasswords|PubkeyAuthentication|GSSAPIAuthentication|RekeyLimit|ClientAliveInterval|ClientAliveCountMax|Banner)\s'
(Get-Acl C:\ProgramData\ssh\sshd_config).Access |
  Format-Table IdentityReference, FileSystemRights, IsInherited
```

A compliant `sshd_config` has these directives **uncommented**. Stock Windows ships them all
commented out, so an untouched install shows `#PermitEmptyPasswords no`, `#ClientAliveInterval 0`
and so on - present as defaults but not set, and `ClientAliveInterval 0` / `ClientAliveCountMax 3`
are non-compliant against the required 600 and 1.
