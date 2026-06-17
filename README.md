# aap-claude-skills

Claude Code skills for Ansible Automation Platform (AAP) on OpenShift.

## Skills

| Skill | Description |
|-------|-------------|
| `/aap-enable-dashboard-ocp` | Enable the automation dashboard post-installation on an OCP operator deployment |

## Installation

```bash
# Add this plugin to Claude Code
claude plugin add https://github.com/cnorville90/aap-claude-skills
```

## Requirements

- Red Hat Ansible Automation Platform 2.7
- OpenShift operator deployment (not containerized)
- `oc` CLI authenticated to the cluster
