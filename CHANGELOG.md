# Changes to WIN25STIG

## Based on STIG v1.2.0 - ALD Windows Alignment Updates

### Connection severing controls guarded - carried across from the Windows Fleet

Benchmark: **Windows Server 2025 STIG v1.2.0**.

A Windows Fleet role was run against a live domain joined host in September 2026 and lost the
host twice to controls that remove a local account's network access. Both have direct equivalents
here, and neither was guarded. Neither is reachable by `ansible-lint`, `yamllint` or
`--syntax-check`; both only bite at runtime, against a domain joined member server.

**These guards are ported and statically verified. They have not been run against a Windows Server 2025
host.** Each was host proven on the fleet test host.

- FIXED: **WN25-MS-000020 would sever the control connection.** It writes
  `LocalAccountTokenFilterPolicy=0`, filtering the privileged token of local accounts on network
  logon, so a run connected over WinRM or psrp as a local administrator loses the host as it
  applies. It presents as `the specified credentials were rejected by the server` rather than a
  dropped connection, because the port keeps answering and the service keeps running.

- FIXED: **WN25-MS-000080 would sever both transports at once.** Its member server branch denies
  network logon to `Guests`, `Enterprise Admins`, `Domain Admins`, `Local account` and
  `Local account and member of Administrators group`. That removes the network logon right from
  every local account on the host, including the one the run is connected as. Unlike WN25-MS-000020
  it takes SSH with it, because Windows OpenSSH password authentication uses the same logon path.
  On the fleet test host, after the equivalent control applied, every remote transport failed - WinRM
  password as both a local and a domain admin principal, SSH password, SSH public key, and WinRM
  certificate authentication - while 5985, 5986 and 22 all still accepted TCP. The console was the
  only way back.

  Note the deny list also names the two privileged domain groups, so **a Domain Admin does not
  survive it either**. The principal that does is a domain account that is neither a Domain Admin
  nor an Enterprise Admin, and that is a member of the local Administrators group on the target.

  Both controls now carry `not win_skip_for_test` (this role's default remains `false`) and are
  listed with the other connection severing controls in `defaults/main/main.yml`. Both are **also**
  skipped automatically when the run connects as a local account, whatever `win_skip_for_test` is
  set to, from two new facts in `prelim.yml`. The operator is warned once per control and each
  warning is counted through `warning_facts.yml`.

  The facts are deliberately separate. `prelim_control_account_is_local` is about the principal and
  applies on any transport, and guards WN25-MS-000080. `prelim_local_account_network_logon` adds the
  transport test and guards WN25-MS-000020. Collapsing them would either leave WN25-MS-000080 able to
  sever an SSH run, or stop WN25-MS-000020 applying over SSH where it is safe.

  The guard sits on the **member server branch only**. The non-domain branch of WN25-MS-000080, which
  sets the right to `Guests` alone and cannot sever anything, is untouched and still applies.

- NOTE: the local account detection builds its backslash from a YAML single quoted variable rather
  than a Jinja literal. In a Jinja expression neither a single nor a doubled backslash literal
  matches one backslash in this position - the first is a syntax error, the second tests for two -
  so a `DOMAIN\user` principal went undetected. The YAML form takes the character literally and
  never reaches Jinja's string parser.

- CHANGED: the compliance facts file is now JSON. `templates/compliance_facts.ps1.j2` is replaced by
  `templates/compliance_facts.json.j2`, and the role writes
  `C:\ProgramData\ansible\facts.d\compliance_facts.json` instead of `...\compliance_facts.ps1`.
  This aligns the ALD Windows roles on a single facts format. The
  file is now parsed rather than executed to produce the fact.

  **The fact name and its keys are unchanged.** `ansible.windows.setup` builds the key from the
  file's `BaseName`, so it is still `ansible_compliance_facts`, and the five existing keys plus the
  conditional `Cat_N_tag_run` entries keep their names and meanings. Anything already reading this
  fact continues to work.

  Every value goes through `| to_json`, so the CAT toggles are now real JSON booleans rather than
  PowerShell `$true` / `$false`. The conditional tag entries use a leading comma, because a trailing
  comma after `cat_3_hardening_enabled` would emit invalid JSON on the default untagged run.

- ADDED: a task removing a superseded `compliance_facts.ps1` before the new file is written. Both
  extensions are collected and both produce the same fact key, so on a host hardened by an earlier
  release the stale file would otherwise be a second source for `ansible_compliance_facts` with no
  guaranteed precedence.

- ADDED: a `managed_by` key. The PowerShell file opened with a "managed by ansible" comment banner
  and JSON cannot hold comments, so that provenance is recorded as a field instead of being lost. Its
  value is derived from `file_managed_by_ansible`, so the wording stays in one place and the variable
  remains in use.

- `win_template` now sets `newline_sequence: "\r\n"`, so the file is written with CRLF line endings.

  **Not executed.** No Windows inventory was available, so the file is never actually collected by
  `ansible.windows.setup` here. The reasoning about fact naming comes from reading the module source.
  The rendered output was validated offline against all eight CAT tag combinations.

- `README.md`: the `Community Contribution` section still described the old open-contribution model -
  "We encourage you (the community) to contribute to this role" and "All community Pull Requests are
  pulled into the devel branch". That contradicted both `CONTRIBUTING.md`, which states pull requests
  come from approved contributors, and this README's own `Contributing` section a few screens above
  it. Replaced with the wording the Windows Fleet already carried: pull requests from
  approved contributors, issues welcome from everyone, and a pointer to `CONTRIBUTING.md` for
  onboarding and the commit signing requirements. The Windows Fleet now carries an identical
  section.

- `README.md`: removed four controller-side dependencies the role does not use. It declared
  `passlib`, `python-lxml`, `python-xmltodict` and `python-jmespath`, and a paragraph describing an
  OpenSCAP tool installation. This role calls only `ansible.windows`, `community.windows` and
  `ansible.builtin` modules: there is no `password_hash` filter, no `xml` module, no `json_query`
  filter and no OpenSCAP task anywhere in it. `pywinrm` is retained and now carries the note, since
  it is the controller-side connection library. The only XML in the role is PowerShell
  `Get-AppLockerPolicy -Effective -XML` running on the target, which implies nothing about
  controller packages.

- FIXED: `.yamllint` now ignores `.ansible/`, matching the Windows Fleet. The QA gate runs
  the checker twice, and the first run's internal `ansible-lint` installs
  `collections/requirements.yml` into `.ansible/collections/`. `yamllint` then walked that tree on
  the second run and linted several hundred vendored `ansible.windows` and `community.windows` files
  against this role's style rules, so the second run failed while the first passed. The gate is now
  order-independent. Only dependency code is excluded: role YAML is still linted.

### SCAP Compliance Checker auditing

- ADDED: **the role can measure the host it hardens.** With `setup_audit` and `run_audit` it stages
  DISA SCC 5.15 on the target, scopes it to `MS_Windows_Server_2025_STIG`, and scans before and
  after remediation, so the difference between the two scans is the role's own effect rather than
  an assertion about it. `audit_only` scans and ends the run for that host without changing
  anything, and `fetch_audit_output` collects the results to the controller. Every audit variable
  defaults to off, so an ordinary hardening run is unchanged. Each scan leaves a summary and a zip
  of its full output in `audit_log_dir`. That directory and the scanner's working directory stop
  inheriting the `C:\ProgramData` ACL, which lets every local user read, and are restricted to
  SYSTEM and Administrators, because the output names local and domain accounts and includes the
  All Settings full configuration dump. The scanner bundle is checked against DISA's published
  SHA-256, pinned in `defaults/main/audit.yml`, before it is unpacked. Settings are documented
  there and in `docs/README_SCC_QuickStart.md`. A first `audit_only` run against an unremediated
  standalone member server scored pass 95, fail 119, not checked 26, not applicable 51.

- ADDED: **prelim mounts the Default profile and every user profile hive.** The per-user controls
  read `HKEY_USERS`, and before this the role mounted nothing, so they reached only profiles that
  happened to be loaded already. Profiles are enumerated from `ProfileList`, so domain accounts
  with a profile on the host are included. `win25stig_load_user_hives` governs it and defaults to
  `true`. The hives the role mounts are unloaded in an `always` section, so a failing control or
  scan does not leave them mounted. User hives already mounted before the run are left alone; the
  Default profile is always released. With the variable set to `false`, nothing is mounted and the
  per-user controls reach only the hives Windows already has loaded, plus the Default profile
  through its file.

- CHANGED: **WN25-UC-000010 edits the Default profile file only when the hives are not mounted.**
  Its Default profile branch loads `C:\Users\Default\NTUSER.DAT` through `win_regedit`'s `hive:`
  option, which Windows refuses while prelim has the same file mounted as `HKU\DEFAULT`. When the
  hives are mounted, the loop over `HKEY_USERS` already covers the Default profile.

- FIXED: **WN25-PK-000010 warned about a root certificate the benchmark does not require.** It
  checked for DoD Root CA 4 alongside 3, 5 and 6, and warned when it was absent. The V1R2 check text
  requires only 3, 5 and 6, so a compliant host was told it was missing a certificate. The check,
  its warning and its place in the warning count are removed.

### Account policy scope on domain joined hosts

- ADDED: **the 12 secedit backed `[System Access]` controls are skipped on a domain joined host.**
  On such a host the Default Domain Policy owns that section and overwrites it at every policy
  refresh, so a local write reverts within roughly two hours while the run still reports success.
  An audit taken straight afterwards therefore reports a compliance that does not last. A new
  `discovered_account_policy_is_domain_scoped` fact gates the account lockout trio and the password
  policy controls, and the operator is warned once with a pointer to set them in the Default Domain
  Policy instead.

  This is the failure class behind the `The key 'LockoutDuration' in section 'System Access' is not
  a valid key` reports from domain joined hosts. Not reachable by `ansible-lint`, `yamllint` or
  `--syntax-check`: the controls are individually valid and only fail against a joined host.

- ADDED: `prelim.yml` gathers the `windows_domain` fact subset explicitly when
  `ansible_facts['windows_domain_member']` is not already defined, rather than assuming a prior
  gather supplied it. `discovered_domain_joined` is seeded `false` in `vars/main.yml`, so a host
  whose membership cannot be determined still applies the controls rather than silently skipping
  them.

- REMOVED: `tasks/cat2_cloud_lockout_order.yml`, the conditional import that loaded it, and the
  `win25stig_cloud_based_system` and `win25stig_cloud_vendors` variables. The three account lockout
  controls were implemented twice, here and in that file, selected by a cloud detection fact.
  Both copies had converged on the same keys in the same order, so the duplicate carried
  nothing the standard path did not.
  Cloud detection was also unreliable by construction: Azure and on-prem Hyper-V both report
  `Microsoft Corporation` with model `Virtual Machine`, so the two could not be told apart by the
  facts the detection used. Any inventory setting either variable can drop it; neither is read any
  more.


- `README.md`: added a `Domain Members` section documenting that account policy is domain scoped,
  which controls are skipped on a domain joined host, and the instruction to set them in the
  Default Domain Policy instead.

### Repository hygiene

- FIXED: **text that was wrong for the role as it ships.** The README `Pipeline Testing` section
  now names the branches each validation workflow runs on, the pinned runner virtualenv, the
  Azure target that is torn down after the run, and the fork guard. Comments in the pipeline
  workflows, `repo_qa.yml` and `.qa_config.yml` no longer describe an AWS runner, a push trigger,
  a `benchmark_v2r*` branch family, removed pipelines or a `wn25_` register prefix. British
  spellings in the QuickStart guides, the changelog and `templates/sshd_config.j2` are now
  American English; the template change alters one comment line in the rendered `sshd_config`.

- FIXED: **pipeline credentials were appended to disk and never removed.** Both validation
  workflows wrote `sensitive_info.json` with `>>`, so a second run in the same workspace appended
  a second JSON object to the first and produced a file OpenTofu cannot parse, and no step ever
  deleted it. On a persistent self-hosted runner the plaintext Windows credentials stayed in the
  workspace until some later checkout happened to clean it. The redirect now truncates, and a
  removal step runs under `if: always()`, so it fires on failure and when `ENABLE_DEBUG` keeps
  the instance alive.

- FIXED: **neither validation pipeline declared a `concurrency` group.** Two pull requests opened
  close together each reached `tofu apply` and raced over OpenTofu state and the cloud resources
  it tracks. A new run now cancels an earlier run of the same pull request, and the Azure job
  holds a repository-wide lock that queues rather than cancels, so a run is never stopped
  mid-apply with an instance left behind. This matches the rest of the Windows Fleet.

- ADDED: **the validation pipelines scan the test host with SCC.** The Azure run now passes
  `setup_audit`, `run_audit` and `fetch_audit_output`, and uploads the fetched result sets as a
  workflow artifact kept for 30 days instead of leaving them on the persistent runner. The sets
  name local and domain accounts and carry their SIDs.

- ADDED: **`results/` is in `.gitignore`.** `audit_output_destination` defaults to `./results/`,
  which resolves inside the checkout under the invocation the QuickStart documents, and the fetched
  set includes the SCC All Settings report, a full host configuration dump.

- CHANGED: **`site.yml` names its play.** The play was anonymous and carried
  `# noqa: name[play]` to silence the lint rule rather than satisfy it. It is now
  `Run Windows Server 2025 STIG Benchmark` and the suppression is gone, matching the play shape
  used elsewhere in the fleet.

- FIXED: **Tofu Destroy did not run when `ENABLE_DEBUG` was unset.** The teardown step was gated on
  `env.ENABLE_DEBUG == 'false'`, which is false for an unset or empty variable, so the Azure test
  instance was left running. Now gated on `!= 'true'`, which tears down by default and keeps the
  instance only when debugging is explicitly requested.
- FIXED: **the IAC_BRANCH test was a shell syntax error when the variable was unset.**
  `if [ ${{ vars.IAC_BRANCH }} != '' ]` expands to `if [ != '' ]` with no value. Replaced with
  `if [ -n "${{ vars.IAC_BRANCH }}" ]`.
- FIXED: **the debug step echoed an undefined variable.** `$benchmark_type` is never set; the
  environment carries `TF_VAR_benchmark_type`. Corrected, so `DEBUG - Show IaC files` reports the
  benchmark type instead of an empty string.
- ADDED: explicit least-privilege `permissions:` blocks on both pipeline jobs, rather than inheriting
  the default token scope, and `workflow_dispatch` so either pipeline can be run manually.
- ADDED: `issue_message` to the pinned `actions/first-interaction@v3.1.0` step. The v3.1.0 runtime
  calls `getInput` for it with `required: true` even though its own `action.yml` does not mark it
  required, so the welcome job failed without it.
- `README.md`: removed the two pipeline-status badges and rewrote the `Pipeline Testing` section.
  The previous text described an audit-on-devel pipeline this role does not run. The section now
  records the ansible-core floor, how collections are resolved, the Azure target, the branches the
  automated test gates on, and the self-hosted OpenTofu runners. `Local Testing` is unchanged and
  still accurate.

- replaced `CONTRIBUTING.rst` with `CONTRIBUTING.md`, carrying the current Ansible-Lockdown
  contributing guide. The Windows Fleet now ships a byte-identical file
- `README.md`: added a Contributing section pointing at `CONTRIBUTING.md`, normalized the social
  badge to the `X URL` form on `x.com`, and pointed the Discord link at
  `https://www.lockdownenterprise.com/discord`
- `README.md`: aligned the shared heading text and the benchmark banner format with the rest of the
  Windows fleet. Role-specific sections are unchanged
- `defaults/main.yml` moved to `defaults/main/main.yml`, matching the directory layout the Linux
  roles have adopted. Ansible loads role defaults from the directory, so no task or template change
  was needed and no variable name or value changed. Verified by loading the defaults tree through
  Ansible's role-defaults loader
- CHANGED: `benchmark_version` is now expressed numerically, `v1r2` becomes `v1.2.0`. This is the
  value `templates/compliance_facts.json.j2` renders into the local fact `Benchmark_release`, so that
  fact changes from `STIG-v1r2` to `STIG-v1.2.0` on the next run
- the benchmark content is unchanged and remains V1R2

- CHANGED: `repo_qa.yml` now runs on the self-hosted runner rather than a hosted one. The
  linter install is isolated in a virtualenv, which a hosted runner would not need: this
  runner persists between jobs, so installing into its interpreter would leak the pinned
  `yamllint` and `ansible-lint` versions into every other workflow sharing it, and a bare
  `pip install` into a system interpreter is refused on current Debian and Ubuntu
  (externally-managed-environment). The venv is prepended to `PATH` so the checker's
  `shutil.which` lookup still finds both linters.
- CHANGED: `repo_qa.yml` runs on pull requests only. The `push` trigger and the weekly
  `schedule` cron are removed: a push to a release branch arrives through a pull
  request anyway, and the cron reported drift on a build nobody was watching.
  `workflow_dispatch` is retained for checking drift deliberately.

- FIXED: `repo_qa.yml` pinned the QA checker at `2.8.3`, which resolves a role's defaults as the
  single file `defaults/main.yml` and aborts before running any check when it is absent. This
  role now keeps its defaults in a `defaults/main/` directory, so the pinned checker could not
  run against it at all and the gate would have failed on its first pull request. Pinned to
  `2.8.4`, which accepts the directory and also understands the `Cat<N>` task directories and
  the Windows toggle shape.
- CHANGED: the checker run now names the variable prefix with `-b`. Left to auto-detect it reads
  the role's benchmark type incorrectly, reports Rule Coverage as a green PASS having compared
  nothing, and keys Unused Variables on whichever prefix it guessed. `.qa_baseline.json` is
  regenerated with the same flag so the two stay in step.

- CHANGED: `repo_qa.yml` no longer depends on `actions/setup-python` succeeding. It prefers the
  runner's own `python3` when that is 3.10 or newer, falls back to `actions/setup-python` only
  when it is not, and then asserts the floor before installing anything, so a runner without a
  suitable interpreter fails with a message naming the version it found rather than with a pip
  resolution error several steps later. 3.10 is the real floor: `yamllint` and `ansible-lint`
  both declare `requires-python >= 3.10`, while the checker itself is standard library only and
  runs on 3.8. Nothing is installed system-wide and no `sudo` is needed on either path, which
  matters on a runner shared with other workflows.

- FIXED: the QA gate reported `Rule Coverage` as SKIP, having compared nothing. This role uses
  two variable prefixes - `win25stig_*` for tunables and `wn25_<family>_<id>` for rule toggles -
  and the checker keys every check off one of them, so no single invocation is honest. Naming
  `-b win25stig` lets Unused Variables see the tunables, but supplying `-b` at all bypasses the
  shared detector for one that cannot return `stig_win`, so Rule
  Coverage found no toggles. Letting it auto-detect fixes Rule Coverage but leaves Unused
  Variables blind to an undefined `win25stig_*` tunable, which is the class that hid an undefined
  variable in this fleet's firewall remediation. The gate now runs both ways and requires both
  to pass. The root cause is in the checker: `_detect_benchmark_type` cannot return `stig_win`.

- FIXED: **WN25-DC-000060 warned the operator about the wrong control.** The manual remediation
  message opened with the title of WN25-DC-000050, Kerberos policy user ticket renewal maximum
  lifetime, while the control is computer clock synchronization tolerance. The task name and the
  policy path beneath it were already correct, so only the headline read as the wrong finding.

- FIXED: **WN25-00-000390 was named as the SMB client control.** The benchmark titles it as the
  SMB server control, and WN25-00-000400 is the client one, so two consecutive controls carried
  the same name and the server control could not be found by name. The remediation was already
  correct, writing `LanmanServer\Parameters\SMB1` against `mrxsmb10\Start` on its neighbor.
  Corrected across the parent task and its four sub-tasks.

- FIXED: **a WN25-00-000200 sub-task had lost its name.** It read `Warn Count."`, with the
  severity, benchmark ID, phase and title stripped and a stray quotation mark left behind. It was
  the only task in the role carrying no benchmark ID in its name.

- FIXED: **the README tag example documented a control identifier from another benchmark.** It
  gave `SV-254238r848530_rule` and `V-254238` for WN25-00-000010; neither appears anywhere in this
  role or in the Server 2025 benchmark. Replaced with `SV-277985r1182240_rule` and `V-277985`, and
  the four NIST and audit tags the real task carries are now shown alongside them.

- `README.md`: removed the claim that the role is compatible with python3 when it is the default
  interpreter and comes with prerequisites that configure the system accordingly. The role sets no
  interpreter and carries no such prerequisites; the sentence was Linux boilerplate. The Python 3
  run environment requirement is stated elsewhere and is unchanged.

- FIXED: **seven NIST tags were misspelled, so selecting by them matched nothing.** A stray `R` in
  `NIST800-53R_4_AC-3` (twice), a trailing `s` on `NIST800-53_4_AC-3s` and
  `NIST800-53A_4_IA-5_1_ds`, a truncated `NIST800-53_4_CM-6_`, a hyphen for an underscore in
  `NIST800-53_4-AC-7_b`, a missing `-53` in `NIST800_AC-8_c`, and an uppercase suffix in
  `NIST800-53_SI-11_B`. Each was a single occurrence against a well populated correct form, and
  each is now that form. The same seven exist in the one Windows sibling that shares this role's
  tag vocabulary, and are corrected there separately. The remaining three siblings use a different
  vocabulary (`NIST800-53R4_` and `NIST800-53A_`, the latter with dot separators, and one of them
  uniformly `NIST800-53R4_` with no dots) and carry none of these.

- FIXED: **17 tags used `NIST800-53-` where 408 use `NIST800-53_`.** The two forms are distinct tag
  strings, so `--tags` selection silently missed the minority spelling. Normalized to the
  underscore separator. Nine duplicate tag entries within a single `tags:` list were removed at the
  same time, eight of them pre-existing and one produced by the corrections above resolving to a
  tag the task already carried. Control identifier tags are untouched: `SV-`, `V-`, `CCI-` and
  `WN25-` counts are identical before and after, and coverage stays at 292 of 292.

## 2026 August - Contributing guide and README refresh

- `README.md`: removed the decorative emoji from headings, and switched the social badge from
  `twitter.com` to `x.com`

## Based on STIG v1.2.0 - Release 1.2.0 - August 2026


Initial release of the Windows Server 2025 STIG remediation role, built against DISA STIG
Windows Server 2025 Version 1, Release 2 (benchmark date 01 July 2026), sourced from
`U_MS_Windows_Server_2025_STIG_V1R2_Manual-xccdf.xml`. Rule coverage is 292 of 292.

- Structure follows the modernized Windows fleet layout: `tasks/Cat1`, `tasks/Cat2` and
  `tasks/Cat3`, one file per control ID band, imported statically so `--tags` selects real tasks.
- Role behavior variables and security tunables use the `win25stig_` prefix. Rule toggles use
  the `wn25_<control id>` form, for example `wn25_au_000010`.
- The role is remediation only. There is no GPO authoring path.
- Fleet-common variables live unprefixed in `vars/main.yml`, matching the Linux roles:
  `min_ansible_version`, `change_requires_reboot`, `company_title` and `file_managed_by_ansible`.
- `skip_os_check` defaults to false. Setting it true bypasses the OS version and family assert for
  hosts the check misidentifies.
- `create_benchmark_facts` defaults to true. The role writes `compliance_facts.ps1` under
  `ansible_facts_path` (`C:\ProgramData\ansible\facts.d`), recording the benchmark release, the run
  date and which CAT levels were enabled. Unlike Linux, Windows has no default local-facts
  directory, so the file is collected only when `ansible.windows.setup` is given a matching
  `fact_path`; it then surfaces as `ansible_compliance_facts`.

- BREAKING: two tunables were misspelled against the Windows rights they set and are renamed.
  `win25stig_sebackuprivilege` becomes `win25stig_sebackupprivilege` (`SeBackupPrivilege`) and
  `win25stig_selockmemorprivilege` becomes `win25stig_selockmemoryprivilege`
  (`SeLockMemoryPrivilege`). Rename them if you override either.
- BREAKING: `win25stig_machineaccountpsswd_max_age` is renamed to
  `win25stig_machineaccountpassword_max_age`. Same misspelling class as the two renames above.
  Rename it if you override it.
- FIXED: two warning messages named the wrong unit. `win25stig_lockoutbadcount` is a count of failed
  logon attempts and `win25stig_resetlockoutcount` is in minutes; both were described as days.
- CHANGED: runtime-discovered values now use the fleet `discovered_` prefix instead of
  `wn25_`, matching the Linux roles and the Windows Fleet. 100 registers and
  set_fact values were renamed, for example `wn25_00_000020_audit_dc` becomes
  `discovered_00_000020_audit_dc`. Facts gathered once in `prelim.yml` keep the `prelim_`
  prefix, which is the same split used across the Windows Fleet. Rule toggles are untouched and keep the
  `wn25_<control id>` form.
  This removes a namespace collision: `wn25_00_000020` (a user toggle) and
  `wn25_00_000020_audit_dc` (a runtime register) previously differed only by suffix, so any
  sweep over toggles had to separate them by regex. These names are internal to the role,
  so overriding a toggle is unaffected.
- FIXED: every operational switch used in a `when:` **or in a template** is now filtered through `| bool`. Passed as an
  extra var (`-e skip_reboot=false`) a switch arrives as the **string** `"false"`, which is truthy,
  so a negated conditional evaluated to the opposite of what was asked. `-e skip_os_check=false`,
  meaning "do run the OS check", silently skipped it; `-e skip_reboot=false`, meaning "do reboot",
  silently suppressed the reboot. Non-negated switches failed outright on ansible-core 2.19+ with
  "Conditionals must have a boolean result". Affects `skip_os_check`, `skip_reboot`,
  `create_benchmark_facts`, `win_skip_for_test`, `change_requires_reboot`, the three
  `win25stig_cat*_controls` switches, `win25stig_cloud_based_system`, `win25stig_disruption_high`,
  `win25stig_complexity_high`, `win25stig_lengthy_search` and `win25stig_install_openssh`.
  The same defect existed in the templates: `compliance_facts.ps1.j2` would have recorded
  `cat_N_hardening_enabled = $true` for a CAT level disabled via `-e`, so the compliance record
  would have misreported what the role applied, and `sshd_config.j2` would have written a directive whose control was disabled via `-e`.
- CHANGED: the OpenSSH configuration controls now render `templates/sshd_config.j2` over the whole
  file instead of editing it directive by directive. The STIG check for each of them greps
  `sshd_config` itself, so a drop-in would not satisfy the documented check, and a whole-file
  render cannot edit the wrong occurrence of a directive that also appears inside a `Match` block.
  **The role now owns this file** - hand edits are replaced on the next run. Four directives
  Windows ships but the benchmark does not mandate are reproduced as tunables so they survive:
  `win25stig_sshd_authorizedkeysfile`, `win25stig_sshd_sftp_subsystem`,
  `win25stig_sshd_allowgroups` and `win25stig_sshd_admin_authorizedkeysfile`, the last of these
  being the `Match Group administrators` block that administrator public key authentication needs.
  Each STIG directive remains gated on its own control toggle.
- FIXED: WN25-UC-000010 wrote `SaveZoneInformation` to `HKLM`, but the benchmark reads
  `HKEY_CURRENT_USER` and the fix text points at a User Configuration policy, so the remediation
  could not affect the state being checked. It now clears the value in every loaded user hive and
  in the Default profile, which covers all logged-on users and all future profiles, and warns for
  any profile whose hive is not loaded rather than skipping it silently.
- FIXED: WN25-MS-000100 was gated to member servers only. V1R2 widened it to "member servers and
  stand-alone or nondomain-joined systems", where the right must instead be empty.
- FIXED: WN25-MS-000140 wrote `RequirePlatformSecurityFeatures`, which belongs to WN25-CC-000110.
  MS-000140 set it to 1 in CAT I and CC-000110 then set it from `win25stig_dma_protection`
  (default 3) in CAT II, so on every run of a domain-joined member server both tasks reported
  changed and the tunable was silently overridden. The control's check text requires only
  `LsaCfgFlags`, so the colliding value has been removed from its loop.
- FIXED: five controls carried no domain-role gate despite the benchmark scoping them.
  WN25-DC-000160 applies to domain controllers only but ran everywhere; WN25-CC-000140,
  WN25-SO-000030, WN25-SO-000040 and WN25-SO-000100 are marked not applicable to domain
  controllers but applied to them.
- CHANGED: the six controls that manage user rights now use `ansible.windows.win_user_right`
  instead of `community.windows.win_security_policy` with `section: Privilege Rights`. Ansible
  warns that the latter is error-prone for rights and privileges, and the split across two modules
  is what allowed WN25-MS-000130 to silently undo WN25-DC-000420 without any same-module conflict
  check noticing. Affected: WN25-UR-000010, WN25-UR-000060, WN25-UR-000080, WN25-UR-000160,
  WN25-DC-000390 and WN25-MS-000130.
- BREAKING: `win25stig_selockmemoryprivilege` is now a **list**, defaulting to `[]`, because
  `win_user_right` takes a list of accounts. Override it with a list, not a string.

- FIXED: the three account lockout controls are applied in the order Windows accepts:
  `WN25-AC-000020`, then `WN25-AC-000010`, then `WN25-AC-000030`. Applying the reset counter before
  the duration made secedit reject the import.
- FIXED: the feature removal controls (`WN25-00-000320` to `WN25-00-000380` and `WN25-00-000410`)
  run only when the feature exists. Windows Server 2025 no longer ships some of them, and
  `win_feature` aborted the play on an unknown name.
- FIXED: `WN25-00-000090` reads TPM status through CIM. `wmic` is not present on Windows Server 2025.
- CHANGED: the OpenSSH banner is composed from `win25stig_legalnoticecaption` and
  `win25stig_legalnoticetext`, the same two variables as the interactive logon notice.
- ADDED: `docs/README_QuickStart_sshd.md` and `docs/README_QuickStart_winrm.md`, operator guides for
  connecting over OpenSSH and WinRM after hardening.

### New Benchmark

- ADDED: the `WN25-SH-*` band, 11 OpenSSH controls, new in this benchmark.
  `WN25-SH-000010` installs OpenSSH Server and enables `sshd`. Because that adds a listening
  network service, it is gated on `win25stig_install_openssh`, which defaults to **false**. The
  other ten harden an existing installation and are gated on `prelim_openssh_configured`, so
  they no-op rather than fail on a host where OpenSSH has not generated its configuration.
  Configuration changes notify the `Restart_sshd` handler, which can be suppressed with
  `win25stig_skip_sshd_restart` when the remediation runs over the SSH connection it would
  otherwise bounce.
- ADDED: `WN25-00-000001` and `WN25-00-000002`, manual documentation warnings for patch
  management and unauthorized hardware.
- ADDED: `WN25-00-000332` (Wi-Fi) as a manual warning, since removing a physical adapter is not
  safely automatable, and `WN25-00-000333` (Bluetooth), which disables the Bluetooth Support
  Service where it exists.
- ADDED: `WN25-AU-000150`, `WN25-AU-000281` and `WN25-DC-000270`, three advanced audit policy
  subcategory settings.
- NOT PRESENT: five controls carried by earlier Windows STIGs are absent from V1R2 -
  `00-000100`, `AC-000070`, `AU-000290`, `CC-000020` and `SO-000300`. V1R2 has no password
  length control, so there is no `win25stig_minimumpasswordlength` tunable.
- `WN25-CC-000250`: `AllowTelemetry` accepts 0 or 1 only; 3 is a finding.
  Tunable `win25stig_allow_telemetry`.
