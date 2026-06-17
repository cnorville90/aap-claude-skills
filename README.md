# aap-claude-skills

Claude Code skills for Ansible Automation Platform (AAP) on OpenShift.

## Skills

| Skill | Description |
|-------|-------------|
| `/aap-enable-dashboard-ocp` | Enable the automation dashboard post-installation on an OCP operator deployment |

## Installation

**1. Add the marketplace:**
```bash
claude plugin marketplace add https://github.com/cnorville90/aap-claude-skills
```

**2. Install the plugin:**
```bash
claude plugin install aap-claude-skills@aap-claude-skills
```

**3. Start a new Claude Code session** — skills load at session start.

**4. Use the skill:**
```
/aap-enable-dashboard-ocp
```

## Requirements

- Red Hat Ansible Automation Platform 2.7
- OpenShift operator deployment (not containerized)
- `oc` CLI authenticated to the cluster
