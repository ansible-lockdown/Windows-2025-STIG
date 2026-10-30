# Windows 2025 DISA STIG

## Configure a Windows 2025 system to be [DISA STIG](https://public.cyber.mil/stigs/downloads/) compliant

### Based on [Windows DISA STIG Version 1, Rel 2 released on July 1st, 2026](https://dl.dod.cyber.mil/wp-content/uploads/stigs/zip/U_MS_Windows_Server_2025_V1R2_STIG.zip)

---

![Org Stars](https://img.shields.io/github/stars/ansible-lockdown?label=Org%20Stars&style=social)
![Stars](https://img.shields.io/github/stars/ansible-lockdown/Windows-2025-STIG?label=Repo%20Stars&style=social)
![Forks](https://img.shields.io/github/forks/ansible-lockdown/Windows-2025-STIG?style=social)
![Followers](https://img.shields.io/github/followers/ansible-lockdown?style=social)
[![X URL](https://img.shields.io/twitter/url/https/x.com/AnsibleLockdown.svg?style=social&label=Follow%20%40AnsibleLockdown)](https://x.com/AnsibleLockdown)

![Discord Badge](https://img.shields.io/discord/925818806838919229?logo=discord)

![Release Branch](https://img.shields.io/badge/Release%20Branch-Main-brightgreen)
![Release Tag](https://img.shields.io/github/v/tag/ansible-lockdown/Windows-2025-STIG?label=Release%20Tag&color=success)
![Main Release Date](https://img.shields.io/github/release-date/ansible-lockdown/Windows-2025-STIG?label=Release%20Date)

![Devel Commits](https://img.shields.io/github/commit-activity/m/ansible-lockdown/Windows-2025-STIG/devel?color=dark%20green&label=Devel%20Branch%20Commits)

![Issues Open](https://img.shields.io/github/issues-raw/ansible-lockdown/Windows-2025-STIG?label=Open%20Issues)
![Issues Closed](https://img.shields.io/github/issues-closed-raw/ansible-lockdown/Windows-2025-STIG?label=Closed%20Issues&color=success)
![Pull Requests](https://img.shields.io/github/issues-pr/ansible-lockdown/Windows-2025-STIG?label=Pull%20Requests)

![License](https://img.shields.io/github/license/ansible-lockdown/Windows-2025-STIG?label=License)

---

## Looking For Support?

[Lockdown Enterprise](https://www.lockdownenterprise.com#GH_AL_WINDOWS_2025_stig)

[Ansible support](https://www.mindpointgroup.com/cybersecurity-products/ansible-counselor#GH_AL_WINDOWS_2025_stig)

### Community

Join us on our [Discord Server](https://www.lockdownenterprise.com/discord) to ask questions, discuss features, or just chat with other Ansible-Lockdown users.

### Contributing

Bug reports and feature requests are welcome from everyone, please raise an issue.

Pull requests are accepted from approved contributors only. To be onboarded, join the [Discord Server](https://www.lockdownenterprise.com/discord) and request contributor access. See [CONTRIBUTING.md](CONTRIBUTING.md) for the full process.

---

## Caution(s)

This role **will make changes to the system** which may have unintended consequences. This is not an auditing tool but a remediation tool to be used after an audit.

Check Mode is not supported! The role will complete in check mode without errors, but it is not supported and should be used with caution.

This role was developed against a clean install of the Windows 2025 operating system. If you are implementing an existing system please review this role for any site-specific changes that are needed.

To use the release version please point to the main branch and relevant release for the STIG benchmark you wish to work with.

---

## Domain Members

Account policy is domain scoped. On a domain joined host the Default Domain Policy owns
`[System Access]`, so the 12 secedit backed controls in this role cannot hold there. `prelim.yml`
detects domain membership, skips those controls and warns once, rather than writing settings the
domain will not keep. Set them in the Default Domain Policy instead.

Everything outside `[System Access]` applies normally on a domain member.

This behavior was verified on a domain joined workstation during Windows Fleet testing, where a
complete hardening run left all `[System Access]` values byte-identical to the pre-run baseline. It
has not been separately measured on Windows Server 2025.

---

## Matching A Security Level For STIG

It is possible to only run controls that are based on a particular security level for STIG.
This is managed using tags:

- CAT1
- CAT2
- CAT3

The control found in the defaults main also needs to reflect true so as this will allow the controls to run when the playbook is launched.

## Coming From A Previous Release

STIG releases always contain changes, so it is highly recommended to review the new references and available variables. This has changed significantly since the initial release of ansible-lockdown.

Further details can be seen in the [Changelog](./CHANGELOG.md)

## Auditing (new)

The role can scan the host with the DISA SCAP Compliance Checker before and after remediation, so
the difference between the two runs shows what the role actually changed. SCC is downloaded to the
target, scoped to the Windows Server 2025 benchmark and run. A summary of each scan and a zip of
its full output are kept in `audit_log_dir`, which only SYSTEM and Administrators can read.

Every audit variable defaults to off, so an ordinary hardening run is unchanged. Enable it with:

```bash
# scan, remediate, scan again
ansible-playbook -i inventory site.yml -e setup_audit=true -e run_audit=true

# scan and stop, changing nothing on the host
ansible-playbook -i inventory site.yml -e setup_audit=true -e audit_only=true
```

Settings live in `defaults/main/audit.yml`. See [docs/README_SCC_QuickStart.md](docs/README_SCC_QuickStart.md)
for the operator guide: variables, profiles, SCC command line behavior and how to read the output.

A first `audit_only` run against a standalone Windows Server 2025 member server scored **pass 95,
fail 119, not checked 26, not applicable 51, 44.4% compliance** on an unremediated host, against
SCC 5.15's bundled content.

## Compliance facts

With `create_benchmark_facts` enabled (the default), the role writes a record of what it applied to:

```
C:\ProgramData\ansible\facts.d\compliance_facts.json
```

It captures the benchmark release, the run date and which CAT levels were enabled.

Windows has no default local-facts directory, so unlike the Linux roles this file is **not**
collected automatically. Ask for it explicitly:

```yaml
- name: Read the compliance facts
  ansible.windows.setup:
    fact_path: 'C:\ProgramData\ansible\facts.d'
```

It then appears as `ansible_compliance_facts` - not under `ansible_local`, which is the Linux
convention. Set `ansible_facts_path` to relocate the directory, or `create_benchmark_facts: false`
to skip writing it.

## Documentation

- [Read The Docs](https://ansible-lockdown.readthedocs.io/en/latest/)
- [Getting Started](https://www.lockdownenterprise.com/docs/getting-started-with-lockdown#GH_AL_WINDOWS_2025_stig)
- [Customizing Roles](https://www.lockdownenterprise.com/docs/customizing-lockdown-enterprise#GH_AL_WINDOWS_2025_stig)
- [Per-Host Configuration](https://www.lockdownenterprise.com/docs/per-host-lockdown-enterprise-configuration#GH_AL_WINDOWS_2025_stig)
- [Getting the Most Out of the Role](https://www.lockdownenterprise.com/docs/get-the-most-out-of-lockdown-enterprise#GH_AL_WINDOWS_2025_stig)

Role-specific guides in this repository:

- [OpenSSH (WN25-SH-*) QuickStart](docs/README_QuickStart_sshd.md) - the OpenSSH band is new in
  the Windows Server 2025 STIG; covers the opt-in install, why the hardening controls gate on
  `sshd_config` existing rather than on the service, and the permission controls.
- [WinRM QuickStart](docs/README_QuickStart_winrm.md) - connecting over HTTP vs HTTPS, and the
  five WinRM controls that can end the session you are running over.

## Requirements

**General:**

- Basic knowledge of Ansible, below are some links to the Ansible documentation to help get started if you are unfamiliar with Ansible

  - [Main Ansible documentation page](https://docs.ansible.com)
  - [Ansible Getting Started](https://docs.ansible.com/ansible/latest/user_guide/intro_getting_started.html)
  - [Tower User Guide](https://docs.ansible.com/ansible-tower/latest/html/userguide/index.html)
  - [Ansible Community Info](https://docs.ansible.com/ansible/latest/community/index.html)
- Functioning Ansible and/or Tower Installed, configured, and running. This includes all of the base Ansible/Tower configurations, needed packages installed, and infrastructure setup.
- Please read through the tasks in this role to gain an understanding of what each control is doing. Some of the tasks are disruptive and can have unintended consequences in a live production system. Also, familiarize yourself with the variables in the defaults/main/main.yml file.

**Technical Dependencies:**

- Windows 2025 - Other versions are not supported
- Running Ansible/Tower setup. This role requires ansible-core 2.16.1 or newer; the role asserts this at run time.
- Python3 Ansible run environment
- pywinrm

`pywinrm` is required on the controller host that executes Ansible; it is the connection library Ansible uses to reach a Windows target.

## Role Variables

### OpenSSH controls (new in the Windows Server 2025 STIG)

The `WN25-SH-*` band hardens Windows OpenSSH and has no counterpart in earlier Windows STIGs.

`WN25-SH-000010` installs OpenSSH Server and enables `sshd`. That adds a listening network
service, so it is gated on `win25stig_install_openssh`, which defaults to `false`. Set it to
`true` where the requirement applies to your estate.

The other ten controls harden an OpenSSH that is already present. They are gated on
`prelim_openssh_configured`, detected in `prelim.yml`, so on a host where OpenSSH has never
started - and therefore has no `sshd_config` - they are skipped rather than failing. Configuration changes notify the `Restart_sshd` handler; set
`win25stig_skip_sshd_restart` to `true` if the remediation runs over the SSH connection it would
otherwise bounce.

### Per-user controls and the registry hives

The benchmark's per-user rules are evaluated against `HKEY_CURRENT_USER`, which `win_regedit`
resolves to the hive of the account Ansible authenticated as. Writing there would put the setting
in the automation account's profile, reaching no real user while still reporting changed.

So `prelim.yml` mounts the Default profile and the hive of every user profile on the host, local
and domain accounts alike, under `HKEY_USERS`, and the per-user controls loop over those. This is
governed by `win25stig_load_user_hives`, which defaults to `true`.

The hives the role mounts are unloaded again at the end of the run, including when the run fails
or ends early. User hives that were already mounted when the role started are left alone, so a
signed-in user does not lose their profile mid-run. The Default profile hive is the exception: no
one signs in with it, and the role always releases it, which also clears one left mounted by an
interrupted earlier run. When WN25-UC-000010 runs, it reports a profile that exists on disk but
is not loaded as a warning rather than silently skipping it.

Setting `win25stig_load_user_hives` to `false` mounts nothing. The per-user controls then reach
only the hives Windows already has loaded, such as those of signed-in users, plus the Default
profile, which WN25-UC-000010 edits through its file directly. They address hives by their
`HKEY_USERS` path and never through the `HKEY_CURRENT_USER` alias.

This role is designed so that the end user should not have to edit the tasks themselves. All customizing should be done via the defaults/main/main.yml file or with extra vars within the project, job, workflow, etc. Non-disruptive CAT I, CAT II, and CAT III findings will be corrected by default. Disruptive finding remediation can be enabled by setting `win25stig_disruption_high` to `true`.

## Tags

Below is an example of the tag section from control within this role. Using this example if you set your run to skip all controls with the tag CCI-000366, this task will be skipped. The opposite can also happen where you run only controls tagged with CCI-000366.

```sh
tags:
      - WN25-00-000010
      - CAT2
      - CCI-000366
      - SRG-OS-000480-GPOS-00227
      - SV-277985r1182240_rule
      - V-277985
      - audit
      - NIST800-53_CM-6_b
      - NIST800-53_A_CM-6_1_iv
      - NIST800-53_4_CM-6_b
```

## Community Contribution

Pull requests are accepted from approved contributors only, and issues are welcome from everyone.
See [CONTRIBUTING.md](CONTRIBUTING.md) for the onboarding process, the rules, and the commit signing
requirements (GPG signature and Signed-off-by on every commit).

## Pipeline Testing

uses:

- ansible-core 2.16.1 or newer, from the pinned virtualenv on the runner
- runs the role against a Windows target provisioned on Azure with OpenTofu, which is torn down when
  the run ends
- self-hosted runners
- pull requests into `devel` or a `benchmark*` branch run the devel pipeline; pull requests into
  `main` or `latest` run the main pipeline
- the job runs only for pull requests raised from a branch in this repository, because it carries the
  cloud credentials

## Local Testing

- Ansible
  - ansible-core 2.16.1 or newer, with Python 3
- `pywinrm` on the controller, which is the connection library Ansible uses to reach a Windows target

## Credits and Thanks

Massive thanks to the fantastic community and all its members.

This includes a huge thanks and credit to the original authors and maintainers.
