# windows_exporter

Deploy and configure Prometheus windows_exporter on Windows Server 2022 via
Chocolatey.

## Description

This role installs windows_exporter using the Chocolatey package
`prometheus-windows-exporter.install`, configures the enabled collectors,
opens the firewall port, and ensures the service starts automatically.

For domain controllers (hosts in the `dc` inventory group), Active Directory
collectors (`ad`, `dns`, `dhcp`) are enabled automatically.

## Requirements

- Ansible >= 2.15
- Collections: `ansible.windows`, `community.windows`, `community.chocolatey`
- Target: Windows Server 2022 with WinRM enabled
- Internet access or local Chocolatey repository

## Role Variables

See `defaults/main.yml` for the full list of variables and their default values.

| Variable | Default | Description |
|---|---|---|
| `winexp_version` | `0.29.2` | Chocolatey package version |
| `winexp_listen_port` | `9182` | HTTP listener port |
| `winexp_listen_address` | `0.0.0.0` | Listener bind address |
| `winexp_collectors` | `cpu,cs,logical_disk,...` | Standard collectors |
| `winexp_collectors_ad` | `ad,dns,dhcp` | AD collectors (DC only) |
| `winexp_textfile_dir` | `C:\Program Files\windows_exporter\textfile_inputs` | Textfile input directory |
| `winexp_firewall_profiles` | `[Domain, Private]` | Firewall rule profiles |

## Tags

| Tag | Scope |
|---|---|
| `install` | Chocolatey package installation |
| `configure` | Textfile directory, service startup |
| `firewall` | Firewall rule for metrics port |

## Example Playbook

```yaml
- hosts: windows
  roles:
    - role: windows_exporter
```

## License

Apache-2.0

## Author

Julien Bombled
