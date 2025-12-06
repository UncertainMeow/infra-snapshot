# Infrastructure Baseball Cards

**Stop hand-waving. Start snapsh

otting.** 🎴📸

> *"Give your AI agents actual facts instead of embarrassingly vague descriptions of your homelab"*

[![Ansible](https://img.shields.io/badge/ansible-2.9+-blue.svg)](https://www.ansible.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

**Automated infrastructure snapshots at multiple detail levels - like baseball cards for your servers.**

---

## 🎯 The Problem

**Before:**
```
You: "I have like... 3 or 4 Proxmox hosts? Maybe 5?
     One's a Dell with GPUs I think... or wait, is it 2 GPUs?
     It runs... stuff? Docker? LXC? Both?"

AI: "Based on your vague hand-waving..."
You: *dies inside*
```

**After:**
```
You: "See AGENTS.md for my current infrastructure"

AI: [reads file]
    "I see you have socrates (Dell R730, 24 vCPU, 128GB RAM,
     2x NVIDIA GPUs, currently at 64% RAM utilization, running
     Ollama and OpenWebUI containers, uptime 3 days 14 hours).

     For your new GitLab deployment, I recommend rawls
     (32GB RAM, only 25% utilized, low CPU load)..."

You: *chef's kiss*
```

---

## 🃏 Snapshot Levels

### Level 1: Baseball Card ⚾

**Quick essential facts** - Perfect for AI agents

**Includes:**
- Status (online/offline)
- IP address, hostname
- OS and version
- CPU/RAM basics
- Uptime
- What's running (containers, VMs)
- Role and purpose

**Output:** Concise markdown table format

**Time:** ~30 seconds

**Use for:** Updating AGENTS.md, quick status checks, "what do I have?"

### Level 2: Work Friend 👔

**Extended conversational detail** - Like chatting with a coworker

**Includes everything from Baseball Card PLUS:**
- Detailed disk usage (all mounts)
- Top 10 largest directories
- Storage pools (ZFS, LVM)
- Network interfaces and connections
- Listening services with ports
- Top processes (CPU and memory)
- Recent log entries (last 50)
- Error/warning counts
- Failed services
- Temperature readings
- Docker container details
- Proxmox cluster status
- Available updates

**Output:** Detailed conversational markdown

**Time:** ~2-3 minutes

**Use for:** Troubleshooting, capacity planning, deep dives

---

## 🚀 Quick Start

### Installation

```bash
# Clone the repo
git clone https://github.com/yourusername/infra-snapshot.git
cd infra-snapshot

# Install Ansible
brew install ansible  # macOS
# or
apt install ansible   # Ubuntu/Debian

# Update inventory with your hosts
vim playbooks/inventory.example.ini
mv playbooks/inventory.example.ini playbooks/inventory.ini

# Test connectivity
ansible -i playbooks/inventory.ini ssh_accessible -m ping

# Run Level 1 snapshot (Baseball Card)
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml

# Check output
cat snapshots/baseball-card-$(date +%Y-%m-%d).md

# Run Level 2 snapshot (Work Friend)
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-work-friend.yml

# Check output
cat snapshots/work-friend-$(date +%Y-%m-%d).md
```

---

## 🏗️ How It Works

```
┌─────────────────────┐
│  Your Infrastructure│
│  • Proxmox hosts    │
│  • VMs & containers │
│  • Storage servers  │
│  • Network devices  │
└──────────┬──────────┘
           │
           │ Ansible queries via SSH
           ▼
┌─────────────────────┐
│  Ansible Playbook   │
│  • Gather facts     │
│  • Run commands     │
│  • Check services   │
│  • Read logs        │
└──────────┬──────────┘
           │
           │ Renders template
           ▼
┌─────────────────────┐
│  Markdown Output    │
│  • Baseball Card    │
│  • Work Friend      │
│  • AGENTS.md        │
└─────────────────────┘
           │
           │ Copy to
           ▼
┌─────────────────────┐
│  AI Agent Context   │
│  "See AGENTS.md"    │
└─────────────────────┘
```

### The Magic

1. **Ansible connects** to your infrastructure via SSH
2. **Gathers facts** (OS, CPU, RAM, uptime, etc.)
3. **Runs commands** to get additional data
4. **Renders Jinja2 template** with all the data
5. **Outputs markdown** perfect for AI agents

**No manual documentation!** Just run a command, copy to AGENTS.md, done.

---

## 📊 Example Output

### Baseball Card Level

```markdown
## 🖥️ socrates

**Status:** ✅ Online
**IP Address:** 10.203.3.42
**Role:** compute
**OS:** Proxmox VE 8.1
**Hostname:** socrates
**Architecture:** x86_64
**Uptime:** up 3 days, 14 hours
**CPU:** 24 vCPUs
**RAM:** 128.0 GB (64.5/128 GB used)
**Disk Usage:** 45%
**Type:** Proxmox Host
**Proxmox Version:** pve-manager/8.1.3
**LXC Containers:** ollama, openwebui
**Description:** Dell R730 with 2x GPUs for AI workloads
**Notes:** Requires fan control container
```

### Work Friend Level

```markdown
# 🖥️ SOCRATES

## Basic Info
- **Status:** ✅ Online
- **IP Address:** 10.203.3.42
- **Load Average:** 2.14, 1.89, 1.75
...

## Storage
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       100G   45G   55G  45% /
/dev/sdb1       500G  320G  180G  64% /var/lib/vz
```

## Performance
### Top 5 Processes by CPU
```
USER       PID %CPU %MEM COMMAND
root      1234  45.2  12.1 ollama serve
```

## Recent Logs (Last 50)
```
Dec 05 10:23:15 socrates systemd[1]: Started Ollama LLM Service
```
```

---

## 🎯 Use Cases

### 1. AI Agent Context

**Problem:** AI doesn't know what you have
**Solution:** Give it AGENTS.md with current snapshot

```bash
# Before talking to AI
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml
# Copy relevant sections to AGENTS.md
# Start AI conversation with "See AGENTS.md"
```

### 2. Capacity Planning

**Problem:** Which host has room for new service?
**Solution:** Work Friend level snapshot shows resource usage

```bash
# Run detailed snapshot
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-work-friend.yml
# Review RAM/CPU/disk usage
# Deploy to host with capacity
```

### 3. Troubleshooting

**Problem:** Something's wrong but don't know where
**Solution:** Work Friend shows errors, failed services, logs

```bash
# Run detailed snapshot
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-work-friend.yml
# Review "Health & Logs" sections
# See failed services, error counts, recent logs
```

### 4. Change Tracking

**Problem:** What changed since last week?
**Solution:** Commit snapshots to git, compare

```bash
# Snapshot before change
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml
git add snapshots/ && git commit -m "Pre-deployment snapshot"

# Make changes...

# Snapshot after
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml
git diff HEAD snapshots/baseball-card-*.md
```

### 5. Documentation

**Problem:** Infrastructure docs always out of date
**Solution:** Re-snapshot regularly, auto-update docs

```bash
# Cron job runs weekly
0 2 * * 0 cd /path/to/repo && ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml
```

---

## 📖 Documentation

- [Quick Start Guide](docs/QUICKSTART.md) - Get up and running
- [Customization Guide](docs/CUSTOMIZATION.md) - Add your own checks
- [AI Agents Guide](docs/AI-AGENTS-GUIDE.md) - Best practices for AI context
- [Scheduling Guide](docs/SCHEDULING.md) - Automate with cron/systemd

---

## 🛠️ Configuration

### Inventory File

Define your infrastructure in `playbooks/inventory.ini`:

```ini
[proxmox_hosts]
socrates ansible_host=10.203.3.42 ansible_user=root role=compute
rawls ansible_host=10.203.3.47 ansible_user=root role=general

[storage]
stuffs ansible_host=10.203.3.99 ansible_user=kellen ansible_become=yes role=nas

[production_services]
gitlab ansible_host=10.203.4.15 ansible_user=root role=iac

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

### Custom Checks

Add your own checks by editing playbooks:

```yaml
- name: Check custom service
  ansible.builtin.shell: systemctl status my-service
  register: my_service_status
  changed_when: false
  failed_when: false
```

Then reference in template:

```jinja
{% if facts.my_service_status is defined %}
**My Service:** {{ facts.my_service_status.stdout }}
{% endif %}
```

---

## 🎨 Customization

### Change Output Format

Templates use Jinja2 - easy to customize:

```jinja
{# Add emoji indicators #}
**Status:** {% if ping_result.ping is defined %}✅ Online{% else %}❌ Offline{% endif %}

{# Color code disk usage #}
**Disk:** {% if disk_usage.stdout | int > 90 %}🔴{% elif disk_usage.stdout | int > 70 %}🟡{% else %}🟢{% endif %} {{ disk_usage.stdout }}

{# Group by role #}
{% for role in ['compute', 'storage', 'networking'] %}
## {{ role | upper }} Hosts
{% for host in groups['ssh_accessible'] if hostvars[host].role == role %}
...
{% endfor %}
{% endfor %}
```

### Filter Hosts

Run snapshot on specific hosts:

```bash
# Only Proxmox hosts
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml --limit proxmox_hosts

# Specific host
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml --limit socrates
```

---

## 📅 Scheduling

### Cron

```bash
# Daily baseball card at 2 AM
0 2 * * * cd /path/to/repo && ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml

# Weekly work friend on Sundays
0 3 * * 0 cd /path/to/repo && ansible-playbook -i playbooks/inventory.ini playbooks/playbook-work-friend.yml
```

### Systemd Timer

```bash
# /etc/systemd/system/homelab-snapshot.service
[Unit]
Description=Homelab Snapshot

[Service]
Type=oneshot
WorkingDirectory=/path/to/repo
ExecStart=/usr/bin/ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml

# /etc/systemd/system/homelab-snapshot.timer
[Unit]
Description=Daily Snapshot

[Timer]
OnCalendar=daily

[Install]
WantedBy=timers.target

# Enable
systemctl enable --now homelab-snapshot.timer
```

---

## 🐛 Troubleshooting

### Host Unreachable

```bash
# Test SSH manually
ssh root@10.203.3.42

# Test with Ansible
ansible -i playbooks/inventory.ini socrates -m ping

# Check inventory
ansible-inventory -i playbooks/inventory.ini --list
```

### Permission Denied

Add `ansible_become=yes` to inventory:

```ini
stuffs ansible_host=10.203.3.99 ansible_user=kellen ansible_become=yes
```

### Slow Execution

Run in parallel:

```bash
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml --forks=10
```

---

## 💡 Pro Tips

### Quick Access Aliases

Add to `~/.zshrc` or `~/.bashrc`:

```bash
alias hl-snap='cd /path/to/repo && ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml'
alias hl-deep='cd /path/to/repo && ansible-playbook -i playbooks/inventory.ini playbooks/playbook-work-friend.yml'
alias hl-show='cat /path/to/repo/snapshots/baseball-card-$(date +%Y-%m-%d).md'
```

### Version Control Snapshots

```bash
# Commit snapshots for history
git add snapshots/
git commit -m "Snapshot $(date +%Y-%m-%d)"

# Compare over time
git log --oneline snapshots/
git diff HEAD~7 snapshots/baseball-card-*.md
```

### JSON Output

Export as JSON for automation:

```yaml
- name: Save as JSON
  ansible.builtin.copy:
    content: "{{ hostvars | to_nice_json }}"
    dest: "./snapshots/snapshot-{{ ansible_date_time.date }}.json"
```

---

## 📜 License

MIT License - see [LICENSE](LICENSE) file for details

---

## 🙏 Acknowledgments

- Built for homelabbers tired of saying "I think I have..."
- Inspired by actual embarrassing conversations with AI agents
- Named "Baseball Cards" because that's what they are
- No more hand-waving, just facts

---

## 🎤 Author

Built with ☕ and 🤖 by Kellen & Claude

**Status:** Production-ready (used in real homelab since 2025-12-05)

**Philosophy:** "If you can't measure it, you can't manage it. If you can't snapshot it, you can't describe it to AI without looking silly."

---

*"Stop hand-waving. Start snapshotting."* 📸
