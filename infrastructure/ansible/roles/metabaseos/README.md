# metabaseos

Base OS configuration role for Windows Server 2022 lab environments.

## Description

This role standardizes the initial configuration of every Windows Server in the
WinOpsLab environment. It covers hostname, networking, regional settings,
Windows Update, security hardening, WinRM, RDP, PowerShell and performance
tuning.

## Requirements

- Ansible >= 2.15
- Collections: `ansible.windows`, `community.windows`
- Target: Windows Server 2022 with WinRM enabled

## Role Variables

See `defaults/main.yml` for the full list of variables and their default values.

## Tags

| Tag | Scope |
|---|---|
| `hostname` | Machine rename + reboot |
| `network` | DNS servers, DNS suffix |
| `regional` | Timezone, NTP, locale, keyboard |
| `updates` | WSUS / Windows Update configuration |
| `security` | Firewall, SMBv1, audit policies |
| `winrm` | HTTPS listener, certificate, tuning |
| `rdp` | Remote Desktop + NLA |
| `powershell` | Execution policy |
| `features` | RSAT, .NET Framework |
| `performance` | Telemetry, visual effects, pagefile |

## Example Playbook

```yaml
- hosts: windows
  roles:
    - role: metabaseos
```

## License

Apache-2.0

## Author

Julien Bombled
