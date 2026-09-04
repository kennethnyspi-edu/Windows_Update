# windows-updates

Selective Windows Update playbook for `ansible.windows.win_updates`. Pick
**Security only**, **Critical only**, **Security + Critical**, **all
categories**, or **specific KB numbers** at run time.

## Layout
```
.
├── windows-updates.yml          # the playbook
├── requirements.yml              # collection dependency (ansible.windows)
├── ansible.cfg                   # points at inventory/hosts.ini by default
├── awx-survey-spec.json          # importable AWX Job Template survey
├── inventory/
│   └── hosts.example.ini         # copy to hosts.ini and edit
└── group_vars/
    └── windows.yml.example       # copy to windows.yml and edit
```
`inventory/hosts.ini` and `group_vars/windows.yml` are gitignored — copy the
`.example` files and fill in your own values locally; they won't get pushed.

## Setup
```bash
git clone <this-repo-url>
cd windows-updates
cp inventory/hosts.example.ini inventory/hosts.ini      # edit with real hostnames
cp group_vars/windows.yml.example group_vars/windows.yml # edit defaults if needed
ansible-galaxy collection install -r requirements.yml
```

## Requirements
- Collection: `ansible.windows` >= 1.7.0 (installed via `requirements.yml`)
- Targets reachable over WinRM; connecting account is a local Administrator
- `win_updates` elevates to SYSTEM itself via `runas` — no `become:` needed,
  and this avoids the WinRM double-hop problem
- Secondary Logon service (`seclogon`) running on targets for that elevation
- No PSWindowsUpdate module required — this uses the native Windows Update
  Agent COM API directly

## Variables

| Variable          | Default             | Notes |
|-------------------|----------------------|-------|
| `target_hosts`    | `windows`            | Inventory host/group/pattern |
| `update_mode`     | `security_critical`  | `security` \| `critical` \| `security_critical` \| `all` \| `kb` |
| `kb_list`         | `[]`                 | YAML list, used when `update_mode: kb`, e.g. `['KB5034441','KB5034439']` |
| `kb_csv`          | `""`                 | Alternative to `kb_list` as a comma-separated string (used by the AWX survey) |
| `reject_list` / `reject_csv` | `[]` / `""` | Always-skip list, any mode — list or CSV form |
| `dry_run`         | `false`              | `true` = search/report only, installs nothing |
| `reboot_updates`  | `true`               | Auto-reboot mid-play if a required update needs it, then resume |
| `reboot_timeout`  | `1800`               | Seconds to wait for host to come back after reboot |

## Usage
```bash
# Security + Critical (default), auto-reboot
ansible-playbook windows-updates.yml

# Security only, no reboot
ansible-playbook windows-updates.yml -e update_mode=security -e reboot_updates=false

# Only two specific KBs
ansible-playbook windows-updates.yml -e update_mode=kb -e '{"kb_list": ["KB5034441","KB5034439"]}'

# Report only, nothing installed
ansible-playbook windows-updates.yml -e dry_run=true
```

## AWX / Job Template setup
1. Create a Job Template pointing at this repo (Project) and
   `windows-updates.yml`, with your Windows machine credential + inventory.
2. Enable **Survey**, then import `awx-survey-spec.json`:
   ```bash
   awx job_templates survey_spec --pk <TEMPLATE_ID> --input awx-survey-spec.json
   # or
   curl -sk -u admin:PASSWORD -X POST \
     https://<awx-host>/api/v2/job_templates/<TEMPLATE_ID>/survey_spec/ \
     -H 'Content-Type: application/json' -d @awx-survey-spec.json
   ```
3. The survey exposes `update_mode` (dropdown), `kb_csv`, `reject_csv`,
   `dry_run`, `reboot_updates`, `target_hosts` — the playbook converts the CSV
   fields into lists internally.

## Notes
Run once with `dry_run: true` (or `state: searched`) to see what a host
reports before committing to `all`. WSUS-managed hosts sometimes expose extra
categories (e.g. `UpdateRollups`) worth splitting out in the `_category_map`
in `windows-updates.yml` if you need finer control.
