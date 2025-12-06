# Homelab Snapshot System

**Purpose:** Generate infrastructure snapshots at varying detail levels for AI agents and documentation.

**Philosophy:** Stop waving your hands when describing your homelab to AI agents. Give them actual, current data.

---

## 📊 Snapshot Levels

### Level 0: Existence ❌
**"You exist at the party but you're about to throw up"**

This level is too basic - just knowing something exists isn't helpful. **We don't support this level.**

### Level 1: Baseball Card ✅
**"Quick stats on the back of a baseball card"**

**Perfect for:** Dropping into AGENTS.md for AI conversations
**Time to run:** ~30 seconds
**Detail level:** Essential facts only

**Includes:**
- Host status (online/offline)
- IP address, hostname
- OS and version
- CPU/RAM basics
- Uptime
- What's running (containers, VMs)
- Role and purpose

**Output:** Concise markdown table format

**Run:**
```bash
ansible-playbook -i inventory.ini playbook-baseball-card.yml
```

**Example output snippet:**
```markdown
## 🖥️ socrates

**Status:** ✅ Online
**IP Address:** 10.203.3.42
**Role:** compute
**OS:** Proxmox VE 8.1
**Uptime:** up 3 days, 14 hours
**CPU:** 24 vCPUs
**RAM:** 128.0 GB (64.5/128 GB used)
**LXC Containers:** ollama, openwebui
**Description:** Dell R730 with 2x GPUs for AI workloads
```

### Level 2: Work Friend ✅
**"Chatting with a coworker about infrastructure"**

**Perfect for:** Deep dives, troubleshooting, planning
**Time to run:** ~2-3 minutes
**Detail level:** Extended information

**Includes everything from Baseball Card PLUS:**
- Detailed disk usage (all mounts)
- Top 10 largest directories
- Storage pools (ZFS, LVM)
- Network interfaces and active connections
- Listening services with ports
- Top processes (CPU and memory)
- Recent log entries (last 50)
- Error/warning counts
- Failed services
- Temperature readings (if available)
- Docker container details
- Proxmox cluster status
- Available updates

**Output:** Conversational markdown format with context

**Run:**
```bash
ansible-playbook -i inventory.ini playbook-work-friend.yml
```

**Example output snippet:**
```markdown
# 🖥️ SOCRATES

## Basic Info
- **Status:** ✅ Online
- **IP Address:** 10.203.3.42
...

## Storage
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       100G   45G   55G  45% /
/dev/sdb1       500G  320G  180G  64% /var/lib/vz
```

### Top 10 Largest Directories
```
45G     /var/lib/vz/images
12G     /var/log
8G      /opt/ollama
...
```

## Performance
### Top 5 Processes by CPU
```
USER       PID %CPU %MEM COMMAND
root      1234  45.2  12.1 ollama serve
...
```

## Recent Logs
```
Dec 05 10:23:15 socrates systemd[1]: Started Ollama LLM Service
Dec 05 10:23:20 socrates kernel: NVIDIA GPU initialized
...
```
```

### Level 3: Oppo Research ❌
**"Every detail for competitive intelligence"**

This level would include:
- Complete log analysis
- Security audit trails
- Configuration file dumps
- Network traffic analysis
- Performance benchmarking

**We don't support this level.** Too much detail, too slow, and usually not needed.

---

## 🎯 Which Level When?

### Use Baseball Card When:
- Starting a conversation with an AI agent
- Quick status check
- Updating AGENTS.md
- Giving someone a tour of your lab
- Creating a "what do I have?" reference

### Use Work Friend When:
- Troubleshooting performance issues
- Planning capacity expansion
- Investigating errors or failures
- Deep-dive with AI on specific problems
- Before/after major changes
- Documentation for compliance or handoff

---

## 📁 Files

### Ansible Inventory
**`inventory.ini`** - Defines all infrastructure hosts and groups

**Key groups:**
- `proxmox_hosts` - Hypervisors
- `storage` - NAS and file servers
- `production_services` - LXC/VMs running services
- `ssh_accessible` - All hosts you can SSH to

**Update this file** when adding/removing hosts.

### Playbooks

**`playbook-baseball-card.yml`** - Level 1 snapshot
- Pings all hosts
- Gathers essential facts
- Generates concise markdown report

**`playbook-work-friend.yml`** - Level 2 snapshot
- Extended fact gathering
- Queries logs, processes, storage
- Generates detailed markdown report

### Templates

**`templates/baseball-card.md.j2`** - Jinja2 template for Level 1 output
**`templates/work-friend.md.j2`** - Jinja2 template for Level 2 output

Edit these to customize output format.

### Output

**`snapshots/baseball-card-YYYY-MM-DD.md`** - Generated Level 1 reports
**`snapshots/work-friend-YYYY-MM-DD.md`** - Generated Level 2 reports

Timestamped so you can track changes over time.

---

## 🚀 Quick Start

### Prerequisites

```bash
# Install Ansible
brew install ansible  # macOS
# or
apt install ansible   # Ubuntu/Debian

# Verify
ansible --version
```

### First Run

```bash
# 1. Update inventory with your hosts
vim inventory.ini

# 2. Test connectivity
ansible -i inventory.ini ssh_accessible -m ping

# 3. Run baseball card snapshot
ansible-playbook -i inventory.ini playbook-baseball-card.yml

# 4. View output
cat snapshots/baseball-card-$(date +%Y-%m-%d).md
```

### Update AGENTS.md

```bash
# 1. Run snapshot
ansible-playbook -i inventory.ini playbook-baseball-card.yml

# 2. Copy relevant sections to AGENTS.md
# (Manual step - copy the host info you want)
```

---

## 🔧 Customization

### Add Custom Facts

Edit the playbook to add your own tasks:

```yaml
- name: Check custom service
  ansible.builtin.shell: systemctl status my-service
  register: my_service_status
  changed_when: false
  failed_when: false
```

Then reference in the template:

```jinja
{% if facts.my_service_status is defined %}
**My Service Status:** {{ facts.my_service_status.stdout }}
{% endif %}
```

### Change Output Format

Templates use Jinja2. Common changes:

**Add emoji status indicators:**
```jinja
**Status:** {% if ping_result.ping is defined %}✅ Online{% else %}❌ Offline{% endif %}
```

**Format disk usage with colors (for terminal output):**
```jinja
**Disk Usage:** {% if disk_usage.stdout | replace('%','') | int > 90 %}🔴{% elif disk_usage.stdout | replace('%','') | int > 70 %}🟡{% else %}🟢{% endif %} {{ disk_usage.stdout }}
```

**Group hosts by role:**
```jinja
{% for role in ['compute', 'storage', 'networking'] %}
## {{ role | upper }} Hosts
{% for host in groups['ssh_accessible'] %}
{% if hostvars[host].role == role %}
...
{% endif %}
{% endfor %}
{% endfor %}
```

### Filter Hosts

Run snapshot on specific hosts only:

```bash
# Only Proxmox hosts
ansible-playbook -i inventory.ini playbook-baseball-card.yml --limit proxmox_hosts

# Only production services
ansible-playbook -i inventory.ini playbook-baseball-card.yml --limit production_services

# Specific host
ansible-playbook -i inventory.ini playbook-baseball-card.yml --limit socrates
```

### Change Output Location

Edit playbook:

```yaml
- name: Generate baseball card markdown
  ansible.builtin.template:
    src: templates/baseball-card.md.j2
    dest: "/path/to/custom/location/snapshot.md"
```

---

## 📅 Scheduling

### Cron (Manual)

```bash
# Run baseball card snapshot daily at 2 AM
0 2 * * * cd /path/to/repo && ansible-playbook -i inventory.ini playbook-baseball-card.yml

# Run work friend snapshot weekly on Sunday at 3 AM
0 3 * * 0 cd /path/to/repo && ansible-playbook -i inventory.ini playbook-work-friend.yml
```

### Systemd Timer

```bash
# /etc/systemd/system/homelab-snapshot.service
[Unit]
Description=Homelab Baseball Card Snapshot

[Service]
Type=oneshot
WorkingDirectory=/path/to/repo
ExecStart=/usr/bin/ansible-playbook -i inventory.ini playbook-baseball-card.yml

# /etc/systemd/system/homelab-snapshot.timer
[Unit]
Description=Daily Homelab Snapshot

[Timer]
OnCalendar=daily
Persistent=true

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

# Check inventory
ansible-inventory -i inventory.ini --list

# Test ping
ansible -i inventory.ini socrates -m ping
```

**Common issues:**
- SSH key not configured (use `ansible_ssh_pass` or `ssh-copy-id`)
- Host is down
- Firewall blocking SSH
- Wrong IP in inventory

### Permission Denied

**Solution 1:** Use `ansible_become=yes` in inventory:
```ini
stuffs ansible_host=10.203.3.99 ansible_user=kellen ansible_become=yes
```

**Solution 2:** Add to playbook:
```yaml
- hosts: all
  become: yes
```

### Command Not Found

Some commands may not exist on all systems:

```yaml
- name: Get uptime (fallback)
  ansible.builtin.shell: uptime -p || uptime
  register: uptime_output
```

### Slow Execution

**Use --forks to run in parallel:**
```bash
ansible-playbook -i inventory.ini playbook-baseball-card.yml --forks=10
```

**Disable fact gathering for faster runs:**
```yaml
gather_facts: no
```

(But then you won't get as much detail!)

---

## 💡 Pro Tips

### 1. Version Control Snapshots

```bash
# Commit snapshots to git
git add snapshots/
git commit -m "Snapshot $(date +%Y-%m-%d): Post network reset"
git push
```

Now you have historical snapshots!

### 2. Compare Snapshots

```bash
# See what changed
diff snapshots/baseball-card-2025-12-01.md snapshots/baseball-card-2025-12-05.md
```

### 3. JSON Output for Automation

Modify playbook to also output JSON:

```yaml
- name: Save as JSON
  ansible.builtin.copy:
    content: "{{ hostvars | to_nice_json }}"
    dest: "./snapshots/snapshot-{{ ansible_date_time.date }}.json"
```

Then parse with scripts:

```bash
jq '.socrates.ansible_facts.ansible_memtotal_mb' snapshots/snapshot-2025-12-05.json
```

### 4. Slack/Discord Notifications

Add to playbook:

```yaml
- name: Send to Slack
  ansible.builtin.uri:
    url: https://hooks.slack.com/services/YOUR/WEBHOOK/URL
    method: POST
    body_format: json
    body:
      text: "Homelab snapshot complete! {{ ansible_date_time.iso8601 }}"
```

### 5. Create Diff Reports

```bash
#!/bin/bash
# snapshot-diff.sh

YESTERDAY=$(date -d yesterday +%Y-%m-%d)
TODAY=$(date +%Y-%m-%d)

ansible-playbook -i inventory.ini playbook-baseball-card.yml

diff -u snapshots/baseball-card-$YESTERDAY.md snapshots/baseball-card-$TODAY.md > snapshots/diff-$TODAY.txt

echo "Changes since yesterday:" snapshots/diff-$TODAY.txt
```

---

## 🤖 Using with AI Agents

### Include in Agent Context

**Option 1: Paste into conversation**
```
Here's my current homelab snapshot:
[paste baseball-card output]
```

**Option 2: Reference AGENTS.md**
```
See AGENTS.md for my current infrastructure
```

**Option 3: Provide on-demand**
```
AI: "What Proxmox hosts do you have?"
You: [run playbook, paste output]
```

### Update AGENTS.md Regularly

```bash
# Weekly update workflow
ansible-playbook -i inventory.ini playbook-baseball-card.yml
# Copy relevant sections to AGENTS.md
vim AGENTS.md
git commit -am "Update AGENTS.md with current state"
```

### AI-Friendly Output

The markdown format is designed to be AI-readable:
- Clear section headers
- Consistent formatting
- Key-value pairs
- Status indicators (✅/❌)
- Contextual notes

AI agents can parse this easily and give accurate answers.

---

## 📊 Example Workflow

**Scenario:** Planning to add a new service

1. **Generate current snapshot:**
   ```bash
   ansible-playbook -i inventory.ini playbook-work-friend.yml
   ```

2. **Review available resources:**
   - Check CPU/RAM usage
   - Check disk space
   - Check which hosts have capacity

3. **Ask AI with context:**
   ```
   Here's my current infrastructure: [paste snapshot]

   I want to deploy GitLab. Which host should I use?
   ```

4. **AI responds with data-driven recommendation:**
   ```
   Based on your snapshot:
   - socrates: 64.5/128 GB RAM used, high CPU (AI workloads)
   - rawls: 8/32 GB RAM used, low CPU ✅ RECOMMENDED

   Deploy GitLab LXC on rawls at 10.203.4.15
   ```

5. **Update inventory and re-snapshot after deployment:**
   ```bash
   # Add to inventory.ini
   gitlab ansible_host=10.203.4.15 ...

   # Re-snapshot
   ansible-playbook -i inventory.ini playbook-baseball-card.yml

   # Update AGENTS.md
   ```

---

## 🎉 You're Ready!

You now have a system to:
- ✅ Generate infrastructure snapshots automatically
- ✅ Choose detail level based on need
- ✅ Give AI agents accurate, current data
- ✅ Track changes over time
- ✅ Stop waving your hands in descriptions

**Next steps:**
1. Update `inventory.ini` with your hosts
2. Run your first snapshot
3. Update `AGENTS.md` with the output
4. Use it in your next AI conversation

**No more:** "I have like... 3 or 4 Proxmox hosts? Maybe 5? One's a Dell with GPUs I think..."

**Now:** "See AGENTS.md - I have socrates (Dell R730, 24 vCPU, 128GB RAM, 2x GPU, 64% utilization, uptime 3 days)..."

🎤 *Mic drop* 🎤
