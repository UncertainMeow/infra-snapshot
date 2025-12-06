# Infra Snapshot - Implementation Plan
**Architecture Review Date:** 2025-12-05
**Current Status:** Production-ready homelab tool (Grade: B+)
**Next Phase:** Security hardening and code quality improvements

---

## Project Overview

### What This Is
An Ansible-based automation system that generates markdown documentation (snapshots) of infrastructure at multiple detail levels. Eliminates hand-waving when describing infrastructure to AI agents.

### Architecture Pattern
**Two-Phase Pipeline (ETL Model)**
1. **Data Collection:** Ansible connects via SSH, gathers facts, executes commands
2. **Transformation:** Jinja2 templates render collected data as markdown

### Current Project Structure
```
infra-snapshot/
├── playbooks/
│   ├── inventory.example.ini          # Infrastructure inventory (240 lines)
│   ├── playbook-baseball-card.yml     # Level 1: Quick facts (110 lines)
│   └── playbook-work-friend.yml       # Level 2: Deep details (285 lines)
├── templates/
│   ├── baseball-card.md.j2            # Quick snapshot format (110 lines)
│   └── work-friend.md.j2              # Extended snapshot format (323 lines)
├── docs/
│   ├── CUSTOMIZATION.md
│   └── QUICKSTART.md
├── examples/
│   ├── AGENTS.md.example
│   └── sample-output/
├── README.md                          # Comprehensive docs (527 lines)
└── snapshots/                         # Generated outputs (created at runtime)
```

### Technology Stack
- **Ansible:** Infrastructure automation (≥2.9 required)
- **Jinja2:** Template rendering (bundled with Ansible)
- **SSH:** Host connectivity
- **Python 3:** Runtime requirement

---

## Current State Assessment

### Strengths ✅
- Clean separation of concerns (collection vs presentation)
- Stateless design (excellent reliability, no corruption risk)
- Well-documented with comprehensive README
- Production-ready for homelab use (1-50 hosts)
- Simple, understandable architecture
- No external dependencies (database, APIs)

### Critical Issues 🔴
1. **SSH Host Key Checking Disabled** (inventory.example.ini:44,206)
   - Setting: `StrictHostKeyChecking=no`
   - Risk: Man-in-the-middle attack vulnerability
   - Impact: HIGH - Credentials/data interception possible

2. **Broad Root Access Required**
   - Most hosts use `ansible_user=root`
   - Violates principle of least privilege
   - Impact: HIGH - Compromised control node = full infrastructure access

3. **Secrets in Plaintext**
   - Inventory contains IPs, usernames, hostnames
   - No encryption at rest
   - Impact: MEDIUM - Repository compromise exposes infrastructure

### Medium Issues 🟡
4. **Code Duplication**
   - Basic facts gathering duplicated between playbooks
   - Lines: playbook-baseball-card.yml:13-88, playbook-work-friend.yml:13-87
   - Impact: Maintenance burden, divergence risk

5. **Missing Templates**
   - Playbooks reference templates but don't validate existence
   - playbook-baseball-card.yml:103, playbook-work-friend.yml:278
   - Impact: Silent failures possible

6. **Hardcoded Values**
   - Magic numbers throughout (./snapshots, log limits, etc.)
   - Impact: Reduced configurability

7. **No Error Aggregation**
   - Uses `ignore_errors: yes` but doesn't report failures
   - Impact: Silent failures, hard to troubleshoot

### Scalability Profile
- **Current:** 1-50 hosts (comfortable)
- **Theoretical Max:** 200-500 hosts (with `--forks` tuning)
- **Enterprise Scale (1000+):** Requires architectural changes

---

## Implementation Roadmap

### Phase 1: Security & Stability (PRIORITY 1 - Week 1)
**Goal:** Eliminate critical security vulnerabilities

**Status:** NOT STARTED
**Estimated Effort:** 8-12 hours
**Blocking Issues:** None

#### Task 1.1: Enable SSH Host Key Verification ⚠️ CRITICAL
**File:** `playbooks/inventory.ini` (will be created from inventory.example.ini)
**Lines to Change:** 44, 206

**Current Code:**
```ini
[proxmox_hosts:vars]
ansible_ssh_common_args='-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null'

[all:vars]
ansible_ssh_common_args='-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null'
```

**New Code:**
```ini
[proxmox_hosts:vars]
ansible_ssh_common_args='-o UserKnownHostsFile=~/.ssh/known_hosts'

[all:vars]
ansible_ssh_common_args='-o UserKnownHostsFile=~/.ssh/known_hosts'
```

**Pre-requisite Steps:**
```bash
# Step 1: Populate known_hosts for all inventory hosts
# Read inventory and extract all hosts
ansible-inventory -i playbooks/inventory.ini --list | jq -r '.._meta.hostvars | to_entries[].value.ansible_host' > /tmp/hosts.txt

# Step 2: Add each host to known_hosts
while read -r host; do
  ssh-keyscan -H "$host" >> ~/.ssh/known_hosts 2>/dev/null
done < /tmp/hosts.txt

# Step 3: Verify known_hosts populated
wc -l ~/.ssh/known_hosts

# Step 4: Test connectivity with new settings
ansible -i playbooks/inventory.ini ssh_accessible -m ping
```

**Success Criteria:**
- All hosts pingable with host key checking enabled
- No MITM warnings during playbook execution

**Rollback Plan:**
- Revert inventory.ini changes
- Remove added known_hosts entries: `sed -i.bak '/10.203./d' ~/.ssh/known_hosts`

---

#### Task 1.2: Implement Ansible Vault ⚠️ CRITICAL
**Files to Encrypt:** `playbooks/inventory.ini`
**New File:** `playbooks/.vault_pass` (optional, for automation)

**Implementation Steps:**

**Step 1: Create vault password file (optional)**
```bash
# For interactive use, skip this and use --ask-vault-pass instead
echo "YOUR_SECURE_PASSWORD_HERE" > playbooks/.vault_pass
chmod 600 playbooks/.vault_pass

# Add to .gitignore
echo "playbooks/.vault_pass" >> .gitignore
```

**Step 2: Encrypt inventory**
```bash
cd /Users/kellen/_code/UncertainMeow/infra-snapshot

# Backup first
cp playbooks/inventory.ini playbooks/inventory.ini.backup

# Encrypt
ansible-vault encrypt playbooks/inventory.ini

# Or with password file
ansible-vault encrypt --vault-password-file playbooks/.vault_pass playbooks/inventory.ini
```

**Step 3: Update playbook execution commands**

**OLD:**
```bash
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml
```

**NEW:**
```bash
# Interactive password prompt
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml --ask-vault-pass

# Or with password file
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml --vault-password-file playbooks/.vault_pass
```

**Step 4: Update README.md**
Update all playbook execution examples (lines 115, 121, 245, 257, 269, 281, 383, 386, 397, 400, 459) to include vault flag.

**Success Criteria:**
- inventory.ini encrypted at rest
- Playbooks execute successfully with vault password
- README updated with vault usage instructions

**Troubleshooting:**
```bash
# View encrypted file
ansible-vault view playbooks/inventory.ini

# Edit encrypted file
ansible-vault edit playbooks/inventory.ini

# Decrypt (for emergencies)
ansible-vault decrypt playbooks/inventory.ini
```

---

#### Task 1.3: Create Read-Only Service Accounts
**Goal:** Replace root access with dedicated ansible-reader accounts

**Scope:** All hosts in inventory (currently 11 hosts)
**Effort per Host:** 15 minutes
**Total Effort:** ~3 hours

**Implementation (Per Host):**

**Step 1: Create service account**
```bash
# SSH to target host as root
ssh root@10.203.3.42

# Create ansible-reader user
useradd -m -s /bin/bash ansible-reader

# Create SSH directory
mkdir -p /home/ansible-reader/.ssh
chmod 700 /home/ansible-reader/.ssh
```

**Step 2: Copy SSH key**
```bash
# On control node
cat ~/.ssh/id_rsa.pub

# On target host
echo "PASTE_PUBLIC_KEY_HERE" > /home/ansible-reader/.ssh/authorized_keys
chmod 600 /home/ansible-reader/.ssh/authorized_keys
chown -R ansible-reader:ansible-reader /home/ansible-reader/.ssh
```

**Step 3: Configure read-only sudo**
```bash
# On target host
visudo
# Add this line:
ansible-reader ALL=(ALL) NOPASSWD: /usr/bin/cat, /usr/bin/ls, /usr/bin/df, /usr/bin/free, /usr/bin/uptime, /usr/sbin/ss, /usr/bin/systemctl status *, /usr/bin/docker ps, /usr/bin/pct list, /usr/sbin/qm list, /usr/sbin/pvesm status, /usr/sbin/pvecm status, /usr/bin/journalctl
```

**Step 4: Update inventory**
```ini
# OLD
socrates ansible_host=10.203.3.42 ansible_user=root

# NEW
socrates ansible_host=10.203.3.42 ansible_user=ansible-reader ansible_become=yes
```

**Step 5: Test**
```bash
ansible -i playbooks/inventory.ini socrates -m ping
ansible -i playbooks/inventory.ini socrates -m shell -a "df -h" --become
```

**Automation Script (Optional):**
Create `scripts/setup-service-account.sh`:
```bash
#!/bin/bash
# Usage: ./setup-service-account.sh root@10.203.3.42

HOST=$1
PUBLIC_KEY=$(cat ~/.ssh/id_rsa.pub)

ssh "$HOST" << 'EOF'
useradd -m -s /bin/bash ansible-reader
mkdir -p /home/ansible-reader/.ssh
chmod 700 /home/ansible-reader/.ssh
echo "$PUBLIC_KEY" > /home/ansible-reader/.ssh/authorized_keys
chmod 600 /home/ansible-reader/.ssh/authorized_keys
chown -R ansible-reader:ansible-reader /home/ansible-reader/.ssh

# Add sudo rules
echo 'ansible-reader ALL=(ALL) NOPASSWD: /usr/bin/cat, /usr/bin/ls, /usr/bin/df, /usr/bin/free, /usr/bin/uptime, /usr/sbin/ss, /usr/bin/systemctl status *, /usr/bin/docker ps, /usr/bin/pct list, /usr/sbin/qm list, /usr/sbin/pvesm status, /usr/sbin/pvecm status, /usr/bin/journalctl' > /etc/sudoers.d/ansible-reader
chmod 440 /etc/sudoers.d/ansible-reader
EOF

echo "Service account created on $HOST"
```

**Success Criteria:**
- All 11 hosts have ansible-reader accounts
- Playbooks execute successfully without root
- Inventory updated and tested

---

#### Task 1.4: Add Template Validation
**Files to Modify:**
- `playbooks/playbook-baseball-card.yml`
- `playbooks/playbook-work-friend.yml`

**Implementation:**

**File:** `playbooks/playbook-baseball-card.yml`
**Insert After:** Line 92 (before "Generate baseball card markdown" task)

```yaml
    - name: Validate baseball card template exists
      ansible.builtin.stat:
        path: templates/baseball-card.md.j2
      register: template_check
      failed_when: not template_check.stat.exists
      delegate_to: localhost
```

**File:** `playbooks/playbook-work-friend.yml`
**Insert After:** Line 269 (before "Generate work friend markdown" task)

```yaml
    - name: Validate work friend template exists
      ansible.builtin.stat:
        path: templates/work-friend.md.j2
      register: template_check
      failed_when: not template_check.stat.exists
      delegate_to: localhost
```

**Test:**
```bash
# Test failure case
mv templates/baseball-card.md.j2 templates/baseball-card.md.j2.backup
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml
# Should fail with clear error

# Restore and test success
mv templates/baseball-card.md.j2.backup templates/baseball-card.md.j2
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml
# Should succeed
```

**Success Criteria:**
- Playbook fails fast if template missing
- Clear error message displayed

---

#### Task 1.5: Parameterize Hardcoded Values
**File to Modify:** `playbooks/inventory.ini` (or inventory.example.ini as template)

**Add to [all:vars] section (after line 203):**

```ini
# Snapshot configuration
snapshot_output_dir=./snapshots
snapshot_date_format=%Y-%m-%d

# Data collection limits
log_entries_recent=50
log_entries_error_check=100
top_processes_count=5
large_dirs_count=10
top_cpu_count=6
top_mem_count=6

# Template paths
template_dir=templates
baseball_card_template=baseball-card.md.j2
work_friend_template=work-friend.md.j2
```

**Files to Update with Variables:**

**File:** `playbooks/playbook-baseball-card.yml`

**Line 97-98 (OLD):**
```yaml
    - name: Create report directory
      ansible.builtin.file:
        path: ./snapshots
        state: directory
        mode: '0755'
```

**Line 97-98 (NEW):**
```yaml
    - name: Create report directory
      ansible.builtin.file:
        path: "{{ snapshot_output_dir }}"
        state: directory
        mode: '0755'
```

**Line 103 (OLD):**
```yaml
        src: templates/baseball-card.md.j2
        dest: "./snapshots/baseball-card-{{ ansible_date_time.date }}.md"
```

**Line 103 (NEW):**
```yaml
        src: "{{ template_dir }}/{{ baseball_card_template }}"
        dest: "{{ snapshot_output_dir }}/baseball-card-{{ ansible_date_time.date }}.md"
```

**File:** `playbooks/playbook-work-friend.yml`

**Similar changes at lines:**
- 273-274: snapshot_output_dir
- 278-279: template paths
- 120: log_entries_recent (change `journalctl -n 50` to `journalctl -n {{ log_entries_recent }}`)
- 126: log_entries_error_check (change `-n 100` to `-n {{ log_entries_error_check }}`)
- 53: large_dirs_count (change `head -10` to `head -{{ large_dirs_count }}`)
- 82: top_cpu_count (change `head -6` to `head -{{ top_cpu_count }}`)
- 88: top_mem_count (change `head -6` to `head -{{ top_mem_count }}`)

**Test Configuration Override:**
```bash
# Test with custom output directory
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml \
  -e "snapshot_output_dir=/tmp/test-snapshots"

# Verify output location
ls -la /tmp/test-snapshots/
```

**Success Criteria:**
- All hardcoded paths replaced with variables
- Configuration overridable via -e flag
- Playbooks execute successfully with defaults

---

### Phase 1 Completion Checklist
- [ ] SSH host key checking enabled
- [ ] Ansible Vault implemented
- [ ] Read-only service accounts created (11 hosts)
- [ ] Template validation added
- [ ] Hardcoded values parameterized
- [ ] README updated with new procedures
- [ ] All tests passing

**Phase 1 Deliverables:**
- Secured inventory (encrypted)
- Validated templates
- Configurable playbooks
- Updated documentation

---

## Phase 2: Code Quality (PRIORITY 2 - Week 2-3)
**Goal:** Reduce technical debt, improve maintainability

**Status:** NOT STARTED
**Estimated Effort:** 16-20 hours
**Blocking Issues:** None (can run parallel with Phase 1 tasks 1.4-1.5)

---

### Task 2.1: Extract Shared Tasks into Ansible Roles
**Goal:** Eliminate code duplication (DRY principle)

**Current Problem:**
- Basic fact gathering duplicated in both playbooks
- playbook-baseball-card.yml:13-88 duplicates playbook-work-friend.yml:13-87
- Changes require editing multiple files

**Solution Architecture:**
```
roles/
├── facts_basic/
│   └── tasks/
│       └── main.yml          # OS, uptime, CPU, RAM, disk
├── facts_proxmox/
│   └── tasks/
│       └── main.yml          # Proxmox version, VMs, containers
├── facts_docker/
│   └── tasks/
│       └── main.yml          # Docker version, containers
└── facts_performance/
    └── tasks/
        └── main.yml          # Top processes, load average
```

**Implementation Steps:**

**Step 1: Create role structure**
```bash
cd /Users/kellen/_code/UncertainMeow/infra-snapshot

mkdir -p roles/{facts_basic,facts_proxmox,facts_docker,facts_performance}/tasks
```

**Step 2: Create facts_basic role**

**File:** `roles/facts_basic/tasks/main.yml`
```yaml
---
# Basic system facts collection
# Used by both baseball-card and work-friend playbooks

- name: Gather system facts
  ansible.builtin.setup:
    gather_subset:
      - '!all'
      - '!min'
      - network
      - hardware
      - virtual
  register: facts_output

- name: Check if host is reachable
  ansible.builtin.ping:
  register: ping_result

- name: Get uptime
  ansible.builtin.command: uptime -p
  register: uptime_output
  changed_when: false
  failed_when: false

- name: Get disk usage summary
  ansible.builtin.shell: df -h / | tail -1 | awk '{print $5}'
  register: disk_usage
  changed_when: false
  failed_when: false

- name: Get memory usage
  ansible.builtin.shell: free -h | grep Mem | awk '{print $3 "/" $2}'
  register: memory_usage
  changed_when: false
  failed_when: false

- name: Get listening ports
  ansible.builtin.shell: ss -tlnp | grep LISTEN | awk '{print $4}' | awk -F: '{print $NF}' | sort -u | tr '\n' ', ' | sed 's/,$//'
  register: listening_ports
  changed_when: false
  failed_when: false
```

**Step 3: Create facts_proxmox role**

**File:** `roles/facts_proxmox/tasks/main.yml`
```yaml
---
# Proxmox-specific fact collection

- name: Check if Proxmox
  ansible.builtin.stat:
    path: /usr/bin/pvesh
  register: is_proxmox

- name: Get Proxmox version
  ansible.builtin.command: pveversion
  register: proxmox_version
  when: is_proxmox.stat.exists
  changed_when: false
  failed_when: false

- name: List running LXC containers (Proxmox)
  ansible.builtin.shell: pct list | tail -n +2 | awk '{print $3}' | tr '\n' ', ' | sed 's/,$//'
  register: lxc_containers
  when: is_proxmox.stat.exists
  changed_when: false
  failed_when: false

- name: List running VMs (Proxmox)
  ansible.builtin.shell: qm list | tail -n +2 | awk '{print $2}' | tr '\n' ', ' | sed 's/,$//'
  register: running_vms
  when: is_proxmox.stat.exists
  changed_when: false
  failed_when: false
```

**Step 4: Create facts_docker role**

**File:** `roles/facts_docker/tasks/main.yml`
```yaml
---
# Docker-specific fact collection

- name: Check for Docker
  ansible.builtin.stat:
    path: /usr/bin/docker
  register: has_docker

- name: List running Docker containers
  ansible.builtin.shell: docker ps --format '{{.Names}}' | tr '\n' ', ' | sed 's/,$//'
  register: docker_containers
  when: has_docker.stat.exists
  changed_when: false
  failed_when: false
```

**Step 5: Update playbook-baseball-card.yml**

**Lines 13-88 (OLD):**
```yaml
  tasks:
    - name: Gather system facts
      ansible.builtin.setup:
    # ... [70 lines of tasks] ...
```

**Lines 13-20 (NEW):**
```yaml
  roles:
    - facts_basic
    - facts_proxmox
    - facts_docker

  tasks:
    # All role tasks now executed automatically
    # Only playbook-specific tasks remain here
```

**Step 6: Update playbook-work-friend.yml similarly**

**Step 7: Test**
```bash
# Test baseball card with roles
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml --limit socrates

# Test work friend with roles
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-work-friend.yml --limit socrates

# Compare output with previous snapshots
diff snapshots/baseball-card-OLD.md snapshots/baseball-card-$(date +%Y-%m-%d).md
```

**Success Criteria:**
- Playbooks use roles instead of duplicated tasks
- Output identical to previous runs
- Easier to add new fact collection (single location)
- playbook-baseball-card.yml reduced from 110 to ~40 lines
- playbook-work-friend.yml reduced from 285 to ~200 lines

---

### Task 2.2: Add Error Reporting and Aggregation
**Goal:** Surface failures instead of silent errors

**Current Problem:**
- Playbooks use `ignore_errors: yes` (line 9 in both)
- Failed hosts appear in templates but no summary
- No way to quickly see what failed

**Solution:** Add error summary section to snapshots

**Implementation Steps:**

**Step 1: Add error collection task**

**File:** `playbooks/playbook-baseball-card.yml`
**Insert at:** Line 89 (end of first play, before reporting play)

```yaml
    - name: Collect error summary
      ansible.builtin.set_fact:
        error_info:
          host: "{{ inventory_hostname }}"
          reachable: "{{ ping_result.ping is defined }}"
          errors: []
      delegate_to: localhost
      delegate_facts: true

    - name: Record SSH connection failure
      ansible.builtin.set_fact:
        error_info: "{{ error_info | combine({'errors': error_info.errors + ['SSH connection failed']}) }}"
      when: ping_result.ping is not defined
      delegate_to: localhost
      delegate_facts: true
```

**Step 2: Update template to show errors**

**File:** `templates/baseball-card.md.j2`
**Insert at:** Line 88 (before infrastructure notes)

```jinja
## ⚠️ Errors and Warnings

{% set failed_hosts = [] %}
{% set warning_hosts = [] %}
{% for host in groups['ssh_accessible'] %}
{% set facts = hostvars[host] %}
{% if facts.ping_result is not defined or facts.ping_result.ping is not defined %}
{% set _ = failed_hosts.append(host) %}
{% elif facts.disk_usage.stdout is defined and (facts.disk_usage.stdout | replace('%', '') | int) > 90 %}
{% set _ = warning_hosts.append(host) %}
{% endif %}
{% endfor %}

{% if failed_hosts | length > 0 %}
### ❌ Failed Hosts ({{ failed_hosts | length }})
{% for host in failed_hosts %}
- **{{ host }}** ({{ hostvars[host].ansible_host }}) - Unreachable
{% endfor %}
{% endif %}

{% if warning_hosts | length > 0 %}
### 🟡 Warnings ({{ warning_hosts | length }})
{% for host in warning_hosts %}
- **{{ host }}** - Disk usage > 90%
{% endfor %}
{% endif %}

{% if failed_hosts | length == 0 and warning_hosts | length == 0 %}
✅ All systems operational
{% endif %}

---
```

**Step 3: Add error count to quick stats**

**File:** `templates/baseball-card.md.j2`
**Line 10-14 (update table):**

```jinja
| Metric | Value |
|--------|-------|
| Total Hosts | {{ groups['ssh_accessible'] | length }} |
| Online Hosts | {{ groups['ssh_accessible'] | select('in', hostvars) | selectattr('ping_result.ping', 'defined') | list | length }} |
| Failed Hosts | {{ groups['ssh_accessible'] | length - (groups['ssh_accessible'] | select('in', hostvars) | selectattr('ping_result.ping', 'defined') | list | length) }} |
| Proxmox Hosts | {{ groups['proxmox_hosts'] | length }} |
| Production Services | {{ groups['production_services'] | length }} |
| Storage Hosts | {{ groups['storage'] | length }} |
```

**Step 4: Similar updates for work-friend.md.j2**

**Step 5: Test error reporting**
```bash
# Introduce intentional failure (unreachable host)
# Edit inventory.ini temporarily
echo "fake-host ansible_host=192.168.99.99 ansible_user=root role=test" >> playbooks/inventory.ini

# Add to ssh_accessible group
sed -i '' '/\[ssh_accessible:children\]/a\
fake_host_group' playbooks/inventory.ini
echo "[fake_host_group]" >> playbooks/inventory.ini
echo "fake-host" >> playbooks/inventory.ini

# Run playbook
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml

# Check snapshot for error section
cat snapshots/baseball-card-$(date +%Y-%m-%d).md | grep -A 10 "Errors and Warnings"

# Should show:
# ❌ Failed Hosts (1)
# - fake-host (192.168.99.99) - Unreachable

# Cleanup
git checkout playbooks/inventory.ini
```

**Success Criteria:**
- Error section appears in snapshots
- Failed hosts clearly listed
- Warnings for high disk usage (>90%)
- Quick stats show online vs failed count

---

### Task 2.3: Implement Snapshot Schema Validation
**Goal:** Verify generated snapshots contain required sections

**Implementation Steps:**

**Step 1: Create validation playbook**

**File:** `playbooks/validate-snapshot.yml`
```yaml
---
- name: Validate Snapshot Schema
  hosts: localhost
  gather_facts: no

  vars:
    snapshot_file: "{{ snapshot_path }}"
    snapshot_type: "{{ type | default('baseball-card') }}"

  tasks:
    - name: Check snapshot file exists
      ansible.builtin.stat:
        path: "{{ snapshot_file }}"
      register: snapshot_stat
      failed_when: not snapshot_stat.stat.exists

    - name: Read snapshot content
      ansible.builtin.slurp:
        src: "{{ snapshot_file }}"
      register: snapshot_content

    - name: Decode snapshot
      ansible.builtin.set_fact:
        snapshot_text: "{{ snapshot_content.content | b64decode }}"

    - name: Validate required sections (baseball-card)
      ansible.builtin.assert:
        that:
          - "'## 📊 Quick Stats' in snapshot_text"
          - "'**Status:**' in snapshot_text"
          - "'**IP Address:**' in snapshot_text"
          - "'Generated by Ansible' in snapshot_text"
        fail_msg: "Snapshot missing required sections"
        success_msg: "✅ Snapshot schema valid"
      when: snapshot_type == 'baseball-card'

    - name: Validate required sections (work-friend)
      ansible.builtin.assert:
        that:
          - "'## Infrastructure Overview' in snapshot_text"
          - "'## Basic Info' in snapshot_text"
          - "'## Hardware Resources' in snapshot_text"
          - "'## Storage' in snapshot_text"
          - "'## Network' in snapshot_text"
        fail_msg: "Snapshot missing required sections"
        success_msg: "✅ Snapshot schema valid"
      when: snapshot_type == 'work-friend'

    - name: Check for empty sections
      ansible.builtin.assert:
        that:
          - snapshot_text | length > 1000
        fail_msg: "Snapshot suspiciously short (< 1000 chars)"
        success_msg: "✅ Snapshot has substantial content"

    - name: Display snapshot stats
      ansible.builtin.debug:
        msg:
          - "Snapshot Type: {{ snapshot_type }}"
          - "File Size: {{ snapshot_stat.stat.size }} bytes"
          - "Line Count: {{ snapshot_text.split('\n') | length }}"
          - "Host Count: {{ snapshot_text | regex_findall('## 🖥️') | length }}"
```

**Step 2: Integrate validation into main playbooks**

**File:** `playbooks/playbook-baseball-card.yml`
**Add at end (after line 109):**

```yaml
    - name: Validate generated snapshot
      ansible.builtin.include_tasks: validate-snapshot.yml
      vars:
        snapshot_path: "./snapshots/baseball-card-{{ ansible_date_time.date }}.md"
        type: baseball-card
```

**Step 3: Create pre-commit hook (optional)**

**File:** `.git/hooks/pre-commit`
```bash
#!/bin/bash
# Validate snapshots before committing

if git diff --cached --name-only | grep -q "snapshots/.*\.md"; then
    echo "Validating snapshots..."

    for snapshot in $(git diff --cached --name-only | grep "snapshots/.*\.md"); do
        if [[ "$snapshot" == *"baseball-card"* ]]; then
            ansible-playbook playbooks/validate-snapshot.yml -e "snapshot_path=$snapshot" -e "type=baseball-card"
        elif [[ "$snapshot" == *"work-friend"* ]]; then
            ansible-playbook playbooks/validate-snapshot.yml -e "snapshot_path=$snapshot" -e "type=work-friend"
        fi

        if [ $? -ne 0 ]; then
            echo "❌ Snapshot validation failed: $snapshot"
            exit 1
        fi
    done

    echo "✅ All snapshots valid"
fi
```

```bash
chmod +x .git/hooks/pre-commit
```

**Step 4: Test validation**
```bash
# Test success case
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml
# Should end with "✅ Snapshot schema valid"

# Test failure case (corrupt snapshot)
echo "broken" > snapshots/test-broken.md
ansible-playbook playbooks/validate-snapshot.yml -e "snapshot_path=snapshots/test-broken.md" -e "type=baseball-card"
# Should fail with "Snapshot missing required sections"
rm snapshots/test-broken.md
```

**Success Criteria:**
- Validation playbook catches malformed snapshots
- Clear error messages for missing sections
- Integration with main playbooks works
- Pre-commit hook prevents bad snapshots (optional)

---

### Task 2.4: Add Pre-commit Hooks for Code Quality
**Goal:** Enforce code quality automatically

**Implementation Steps:**

**Step 1: Install pre-commit framework**
```bash
# On control node
brew install pre-commit  # macOS
# or
pip3 install pre-commit  # Linux/Python
```

**Step 2: Create pre-commit configuration**

**File:** `.pre-commit-config.yaml`
```yaml
repos:
  # Ansible linting
  - repo: https://github.com/ansible/ansible-lint
    rev: v6.22.1
    hooks:
      - id: ansible-lint
        files: \.(yaml|yml)$
        args:
          - --exclude=.github/
          - --exclude=examples/

  # YAML validation
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: check-yaml
        args: ['--unsafe']  # Allow Jinja2 in YAML
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-added-large-files
        args: ['--maxkb=1000']
      - id: check-merge-conflict

  # Markdown linting
  - repo: https://github.com/igorshubovych/markdownlint-cli
    rev: v0.38.0
    hooks:
      - id: markdownlint
        args:
          - --config=.markdownlint.json

  # Custom snapshot validation
  - repo: local
    hooks:
      - id: validate-snapshots
        name: Validate snapshot schemas
        entry: bash -c 'for f in snapshots/*.md; do ansible-playbook playbooks/validate-snapshot.yml -e "snapshot_path=$f" -e "type=$(basename $f | cut -d- -f1)"; done'
        language: system
        files: snapshots/.*\.md$
        pass_filenames: false
```

**Step 3: Create ansible-lint config**

**File:** `.ansible-lint`
```yaml
skip_list:
  - 'yaml[line-length]'  # Allow long lines in templates
  - 'no-changed-when'    # We use changed_when: false explicitly
  - 'command-instead-of-shell'  # Shell needed for pipes

warn_list:
  - 'risky-shell-pipe'   # Warn but don't fail

exclude_paths:
  - .github/
  - examples/
  - docs/

# Enable specific rules
enable_list:
  - 'fqcn-builtins'      # Require fully qualified collection names
  - 'name[play]'         # Require play names
  - 'name[task]'         # Require task names
```

**Step 4: Create markdownlint config**

**File:** `.markdownlint.json`
```json
{
  "default": true,
  "MD013": false,
  "MD033": false,
  "MD041": false,
  "MD046": {
    "style": "fenced"
  }
}
```

**Step 5: Install hooks**
```bash
cd /Users/kellen/_code/UncertainMeow/infra-snapshot

# Install hooks
pre-commit install

# Run manually on all files (first time)
pre-commit run --all-files

# Fix any issues reported
# Then commit
git add .pre-commit-config.yaml .ansible-lint .markdownlint.json
git commit -m "Add pre-commit hooks for code quality"
```

**Step 6: Test hooks**
```bash
# Introduce intentional error
echo "tasks:" >> playbooks/playbook-baseball-card.yml  # Invalid YAML
git add playbooks/playbook-baseball-card.yml
git commit -m "Test commit"
# Should fail with ansible-lint errors

# Fix and retry
git checkout playbooks/playbook-baseball-card.yml
```

**Success Criteria:**
- Pre-commit hooks installed and active
- Ansible-lint catches playbook issues
- YAML validation prevents syntax errors
- Markdown linting enforces consistency
- Team can't commit broken code

---

### Phase 2 Completion Checklist
- [ ] Ansible roles created (facts_basic, facts_proxmox, facts_docker, facts_performance)
- [ ] Playbooks refactored to use roles
- [ ] Error reporting added to templates
- [ ] Snapshot validation playbook created
- [ ] Pre-commit hooks configured and tested
- [ ] Documentation updated
- [ ] All tests passing

**Phase 2 Deliverables:**
- DRY codebase (roles-based)
- Error visibility in snapshots
- Automated quality checks
- Validated snapshot schemas

---

## Phase 3: Operational Excellence (PRIORITY 3 - Month 2)
**Goal:** Production-grade operations and monitoring

**Status:** NOT STARTED
**Estimated Effort:** 20-24 hours
**Blocking Issues:** Requires Phase 1 completion (security)

---

### Task 3.1: Multi-Distribution Support
**Goal:** Support RedHat/CentOS in addition to Debian/Ubuntu

**Current Limitation:**
- Hardcoded Debian commands (apt, systemctl style)
- Lines: playbook-work-friend.yml:223 (apt list)

**Implementation:**

**Step 1: Add OS detection tasks**

**File:** `roles/facts_basic/tasks/main.yml`
**Add after fact gathering:**

```yaml
- name: Set package manager facts
  ansible.builtin.set_fact:
    pkg_manager: "{{ 'apt' if ansible_os_family == 'Debian' else 'yum' if ansible_os_family == 'RedHat' else 'unknown' }}"
    service_manager: "{{ 'systemctl' if ansible_service_mgr == 'systemd' else 'service' }}"
```

**Step 2: Create OS-specific task files**

**File:** `roles/facts_basic/tasks/debian.yml`
```yaml
---
- name: Get available updates (Debian/Ubuntu)
  ansible.builtin.shell: apt list --upgradable 2>/dev/null | wc -l
  register: available_updates
  changed_when: false
  failed_when: false

- name: Get last apt update time
  ansible.builtin.stat:
    path: /var/cache/apt/pkgcache.bin
  register: pkg_cache
```

**File:** `roles/facts_basic/tasks/redhat.yml`
```yaml
---
- name: Get available updates (RedHat/CentOS)
  ansible.builtin.shell: yum list updates 2>/dev/null | grep -v "^Loaded" | grep -v "^Updated" | wc -l
  register: available_updates
  changed_when: false
  failed_when: false

- name: Get last yum update time
  ansible.builtin.stat:
    path: /var/cache/yum
  register: pkg_cache
```

**Step 3: Include OS-specific tasks**

**File:** `roles/facts_basic/tasks/main.yml`
**Add at end:**

```yaml
- name: Include OS-specific tasks
  ansible.builtin.include_tasks: "{{ ansible_os_family | lower }}.yml"
  when: ansible_os_family in ['Debian', 'RedHat']
```

**Step 4: Update templates for OS display**

**File:** `templates/baseball-card.md.j2`
**Line 27:**

```jinja
**OS:** {{ facts.ansible_facts.ansible_distribution | default('Unknown') }} {{ facts.ansible_facts.ansible_distribution_version | default('') }} ({{ facts.pkg_manager | default('unknown') }})
```

**Step 5: Test with multiple distros**
```bash
# If you have RedHat/CentOS hosts, add to inventory
# [redhat_hosts]
# rhel-test ansible_host=10.203.3.100 ansible_user=ansible-reader ansible_become=yes

# Run against mixed environment
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml

# Verify both Debian and RedHat hosts appear correctly
```

**Success Criteria:**
- Debian and RedHat systems both supported
- Package manager auto-detected
- Updates counted correctly for both
- Templates show OS-appropriate information

---

### Task 3.2: Incremental Snapshot Mode
**Goal:** Only scan changed hosts to speed up large environments

**Implementation:**

**Step 1: Create snapshot diff script**

**File:** `scripts/snapshot-diff.py`
```python
#!/usr/bin/env python3
"""
Compare current host facts with previous snapshot.
Output: List of changed hosts for incremental update.
"""

import json
import sys
from pathlib import Path

def load_snapshot(path):
    """Load previous snapshot JSON."""
    if not path.exists():
        return {}
    with open(path) as f:
        return json.load(f)

def compare_hosts(previous, current):
    """Compare host states, return changed hosts."""
    changed = []

    for host, current_facts in current.items():
        prev_facts = previous.get(host, {})

        # Check key fields
        checks = [
            prev_facts.get('uptime') != current_facts.get('uptime'),
            prev_facts.get('memory_usage') != current_facts.get('memory_usage'),
            prev_facts.get('disk_usage') != current_facts.get('disk_usage'),
            prev_facts.get('container_count') != current_facts.get('container_count'),
        ]

        if any(checks) or host not in previous:
            changed.append(host)

    return changed

if __name__ == '__main__':
    prev_path = Path(sys.argv[1]) if len(sys.argv) > 1 else Path('snapshots/previous.json')
    curr_path = Path(sys.argv[2]) if len(sys.argv) > 2 else Path('snapshots/current.json')

    previous = load_snapshot(prev_path)
    current = load_snapshot(curr_path)

    changed = compare_hosts(previous, current)

    print(','.join(changed))  # CSV output for Ansible
```

**Step 2: Add JSON export to playbooks**

**File:** `playbooks/playbook-baseball-card.yml`
**Add task in reporting play:**

```yaml
    - name: Export host facts as JSON
      ansible.builtin.copy:
        content: "{{ hostvars | to_nice_json }}"
        dest: "./snapshots/snapshot-{{ ansible_date_time.date }}.json"
      delegate_to: localhost
```

**Step 3: Create incremental playbook**

**File:** `playbooks/playbook-incremental.yml`
```yaml
---
- name: Determine changed hosts
  hosts: localhost
  gather_facts: yes

  tasks:
    - name: Run diff script
      ansible.builtin.command: python3 scripts/snapshot-diff.py snapshots/previous.json snapshots/current.json
      register: changed_hosts_output
      changed_when: false
      failed_when: false

    - name: Set changed hosts fact
      ansible.builtin.set_fact:
        changed_hosts: "{{ changed_hosts_output.stdout.split(',') }}"

    - name: Display changed hosts
      ansible.builtin.debug:
        msg: "Changed hosts: {{ changed_hosts }}"

- name: Snapshot changed hosts only
  hosts: ssh_accessible
  gather_facts: yes
  ignore_errors: yes

  tasks:
    - name: Skip unchanged hosts
      ansible.builtin.meta: end_host
      when: inventory_hostname not in hostvars['localhost'].changed_hosts

    # Include normal baseball-card tasks here
    - ansible.builtin.include_role:
        name: facts_basic
    - ansible.builtin.include_role:
        name: facts_proxmox
    - ansible.builtin.include_role:
        name: facts_docker

# Generate report (same as playbook-baseball-card.yml)
- ansible.builtin.import_playbook: playbook-baseball-card.yml
```

**Step 4: Test incremental mode**
```bash
# First run (full)
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml
cp snapshots/snapshot-$(date +%Y-%m-%d).json snapshots/previous.json

# Second run (incremental - should be fast)
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-incremental.yml

# Make a change on one host
ssh root@10.203.3.42 "systemctl restart docker"

# Third run (should only scan changed host)
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-incremental.yml
```

**Success Criteria:**
- Incremental mode significantly faster (>50% time reduction)
- Only changed hosts scanned
- Full snapshot still available via normal playbook
- JSON diff script accurate

---

### Task 3.3: Monitoring Integration (Prometheus)
**Goal:** Export metrics for trending and alerting

**Implementation:**

**Step 1: Create Prometheus exporter template**

**File:** `templates/metrics.prom.j2`
```jinja
# HELP homelab_host_up Host reachability (1 = up, 0 = down)
# TYPE homelab_host_up gauge
{% for host in groups['ssh_accessible'] %}
{% set facts = hostvars[host] %}
homelab_host_up{host="{{ host }}",ip="{{ facts.ansible_host }}",role="{{ facts.role | default('unknown') }}"} {{ 1 if facts.ping_result.ping is defined else 0 }}
{% endfor %}

# HELP homelab_host_uptime_seconds Host uptime in seconds
# TYPE homelab_host_uptime_seconds gauge
{% for host in groups['ssh_accessible'] %}
{% set facts = hostvars[host] %}
{% if facts.uptime_output is defined and facts.uptime_output.stdout is defined %}
homelab_host_uptime_seconds{host="{{ host }}"} {{ facts.uptime_output.stdout | regex_replace('up (\\d+) days.*', '\\1') | int * 86400 }}
{% endif %}
{% endfor %}

# HELP homelab_disk_usage_percent Disk usage percentage
# TYPE homelab_disk_usage_percent gauge
{% for host in groups['ssh_accessible'] %}
{% set facts = hostvars[host] %}
{% if facts.disk_usage is defined and facts.disk_usage.stdout is defined %}
homelab_disk_usage_percent{host="{{ host }}",mount="/"} {{ facts.disk_usage.stdout | replace('%', '') | int }}
{% endif %}
{% endfor %}

# HELP homelab_memory_usage_bytes Memory usage in bytes
# TYPE homelab_memory_usage_bytes gauge
{% for host in groups['ssh_accessible'] %}
{% set facts = hostvars[host] %}
{% if facts.ansible_facts is defined and facts.ansible_facts.ansible_memtotal_mb is defined %}
homelab_memory_usage_bytes{host="{{ host }}",type="total"} {{ facts.ansible_facts.ansible_memtotal_mb * 1024 * 1024 }}
{% endif %}
{% endfor %}

# HELP homelab_container_count Number of running containers
# TYPE homelab_container_count gauge
{% for host in groups['ssh_accessible'] %}
{% set facts = hostvars[host] %}
{% if facts.docker_containers is defined and facts.docker_containers.stdout is defined %}
homelab_container_count{host="{{ host }}",type="docker"} {{ facts.docker_containers.stdout.split(',') | length }}
{% endif %}
{% if facts.lxc_containers is defined and facts.lxc_containers.stdout is defined %}
homelab_container_count{host="{{ host }}",type="lxc"} {{ facts.lxc_containers.stdout.split(',') | length }}
{% endif %}
{% endfor %}

# HELP homelab_snapshot_timestamp_seconds Last snapshot time
# TYPE homelab_snapshot_timestamp_seconds gauge
homelab_snapshot_timestamp_seconds {{ ansible_date_time.epoch }}
```

**Step 2: Add Prometheus export task**

**File:** `playbooks/playbook-baseball-card.yml`
**Add in reporting play:**

```yaml
    - name: Generate Prometheus metrics
      ansible.builtin.template:
        src: templates/metrics.prom.j2
        dest: "./snapshots/homelab-metrics.prom"
      delegate_to: localhost

    - name: Copy metrics to node_exporter textfile directory (if configured)
      ansible.builtin.copy:
        src: "./snapshots/homelab-metrics.prom"
        dest: "{{ prometheus_textfile_dir }}/homelab-metrics.prom"
      when: prometheus_textfile_dir is defined
      delegate_to: localhost
```

**Step 3: Configure Prometheus scraping**

**File:** `docs/PROMETHEUS_SETUP.md` (documentation)
```markdown
# Prometheus Integration

## Setup Node Exporter Textfile Collector

1. Install node_exporter with textfile collector:
```bash
# On monitoring host
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xvfz node_exporter-1.7.0.linux-amd64.tar.gz
sudo cp node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/

# Create textfile directory
sudo mkdir -p /var/lib/node_exporter/textfile_collector

# Run with textfile collector
node_exporter --collector.textfile.directory=/var/lib/node_exporter/textfile_collector
```

2. Configure inventory:
```ini
[all:vars]
prometheus_textfile_dir=/var/lib/node_exporter/textfile_collector
```

3. Run playbook:
```bash
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml
```

4. Verify metrics:
```bash
curl http://localhost:9100/metrics | grep homelab_
```

## Example Prometheus Queries

- Host availability: `homelab_host_up`
- Disk usage > 90%: `homelab_disk_usage_percent > 90`
- Memory usage: `homelab_memory_usage_bytes`
- Container count by host: `sum by (host) (homelab_container_count)`

## Example Alerts

```yaml
groups:
  - name: homelab
    rules:
      - alert: HostDown
        expr: homelab_host_up == 0
        for: 5m
        annotations:
          summary: "Host {{ $labels.host }} is down"

      - alert: DiskSpaceHigh
        expr: homelab_disk_usage_percent > 90
        for: 10m
        annotations:
          summary: "Disk usage on {{ $labels.host }} is {{ $value }}%"
```
```

**Step 4: Test Prometheus integration**
```bash
# Generate metrics
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml

# Verify metrics file
cat snapshots/homelab-metrics.prom

# Should show:
# homelab_host_up{host="socrates",ip="10.203.3.42",role="compute"} 1
# homelab_disk_usage_percent{host="socrates",mount="/"} 45
# etc.

# If node_exporter configured, verify scraping
curl http://your-monitoring-host:9100/metrics | grep homelab_
```

**Success Criteria:**
- Metrics exported in Prometheus format
- Node exporter textfile collector integration works
- Documentation for Prometheus/Grafana setup complete
- Example queries and alerts provided

---

### Phase 3 Completion Checklist
- [ ] Multi-distribution support (Debian + RedHat)
- [ ] Incremental snapshot mode implemented
- [ ] Prometheus metrics export working
- [ ] Monitoring documentation complete
- [ ] Performance testing (time savings measured)
- [ ] All tests passing

**Phase 3 Deliverables:**
- Multi-OS support
- Performance optimization (incremental mode)
- Metrics pipeline (Prometheus)
- Monitoring documentation

---

## Phase 4: Advanced Integrations (PRIORITY 4 - Month 3+)
**Goal:** Enterprise-grade integrations and extensibility

**Status:** NOT STARTED
**Estimated Effort:** 30-40 hours
**Blocking Issues:** Requires Phase 1-2 completion

---

### Task 4.1: NetBox IPAM Integration
**Goal:** Sync inventory with NetBox (already in environment!)

**Current Opportunity:**
- NetBox already in inventory (10.203.4.16)
- Manual inventory maintenance is error-prone
- NetBox has authoritative device data

**Implementation:**

**Step 1: Create NetBox dynamic inventory script**

**File:** `scripts/netbox-inventory.py`
```python
#!/usr/bin/env python3
"""
Ansible dynamic inventory from NetBox IPAM.
Returns JSON inventory structure.
"""

import os
import sys
import json
import requests

NETBOX_URL = os.getenv('NETBOX_URL', 'http://10.203.4.16')
NETBOX_TOKEN = os.getenv('NETBOX_TOKEN', '')  # Set via environment

def get_netbox_devices():
    """Query NetBox API for all devices."""
    headers = {
        'Authorization': f'Token {NETBOX_TOKEN}',
        'Content-Type': 'application/json'
    }

    response = requests.get(
        f'{NETBOX_URL}/api/dcim/devices/',
        headers=headers,
        params={'limit': 1000}
    )
    response.raise_for_status()
    return response.json()['results']

def build_inventory(devices):
    """Convert NetBox devices to Ansible inventory."""
    inventory = {
        '_meta': {'hostvars': {}},
        'all': {'children': ['ungrouped']}
    }

    # Group by device role
    for device in devices:
        role = device['device_role']['slug']

        # Create group if needed
        if role not in inventory:
            inventory[role] = {'hosts': []}
            if 'children' not in inventory['all']:
                inventory['all']['children'] = []
            if role not in inventory['all']['children']:
                inventory['all']['children'].append(role)

        # Add host
        hostname = device['name']
        inventory[role]['hosts'].append(hostname)

        # Build hostvars
        hostvars = {
            'ansible_host': device['primary_ip4']['address'].split('/')[0] if device.get('primary_ip4') else None,
            'ansible_user': 'ansible-reader',
            'ansible_become': True,
            'role': role,
            'device_type': device['device_type']['model'],
            'serial': device.get('serial', ''),
            'asset_tag': device.get('asset_tag', ''),
            'site': device['site']['name'],
        }

        # Add custom fields
        if device.get('custom_fields'):
            for key, value in device['custom_fields'].items():
                hostvars[f'netbox_{key}'] = value

        inventory['_meta']['hostvars'][hostname] = hostvars

    return inventory

if __name__ == '__main__':
    if '--list' in sys.argv:
        devices = get_netbox_devices()
        inventory = build_inventory(devices)
        print(json.dumps(inventory, indent=2))
    elif '--host' in sys.argv:
        # Ansible expects this but we return hostvars in --list
        print(json.dumps({}))
    else:
        print("Usage: netbox-inventory.py --list")
        sys.exit(1)
```

**Step 2: Make script executable**
```bash
chmod +x scripts/netbox-inventory.py
```

**Step 3: Test dynamic inventory**
```bash
# Set NetBox API token
export NETBOX_TOKEN="your-netbox-api-token-here"

# Test script
scripts/netbox-inventory.py --list | jq .

# Test with Ansible
ansible-inventory -i scripts/netbox-inventory.py --list

# Run playbook with NetBox inventory
ansible-playbook -i scripts/netbox-inventory.py playbooks/playbook-baseball-card.yml
```

**Step 4: Create wrapper script**

**File:** `scripts/sync-netbox.sh`
```bash
#!/bin/bash
# Sync inventory with NetBox and run snapshot

set -e

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(dirname "$SCRIPT_DIR")"

echo "🔄 Syncing with NetBox..."

# Check NetBox token
if [ -z "$NETBOX_TOKEN" ]; then
    echo "❌ NETBOX_TOKEN environment variable not set"
    echo "   Export your NetBox API token:"
    echo "   export NETBOX_TOKEN='your-token-here'"
    exit 1
fi

# Fetch dynamic inventory
echo "📡 Fetching devices from NetBox..."
python3 "$SCRIPT_DIR/netbox-inventory.py" --list > /tmp/netbox-inventory.json

# Validate inventory
HOST_COUNT=$(jq -r '._meta.hostvars | length' /tmp/netbox-inventory.json)
echo "✅ Found $HOST_COUNT hosts in NetBox"

# Run snapshot with NetBox inventory
echo "📸 Running snapshot..."
cd "$PROJECT_ROOT"
ansible-playbook -i scripts/netbox-inventory.py playbooks/playbook-baseball-card.yml "$@"

echo "✅ NetBox sync complete"
```

```bash
chmod +x scripts/sync-netbox.sh
```

**Step 5: Update documentation**

**File:** `docs/NETBOX_INTEGRATION.md`
```markdown
# NetBox IPAM Integration

## Overview
Use NetBox as single source of truth for infrastructure inventory.

## Setup

1. Get NetBox API token:
   - Login to NetBox: http://10.203.4.16
   - Navigate to: Profile → API Tokens
   - Create token with read permissions

2. Export token:
```bash
export NETBOX_TOKEN="your-api-token-here"

# Or add to ~/.zshrc for persistence
echo 'export NETBOX_TOKEN="your-api-token-here"' >> ~/.zshrc
```

3. Test connection:
```bash
scripts/netbox-inventory.py --list
```

4. Run snapshot with NetBox:
```bash
scripts/sync-netbox.sh
```

## Advantages

- ✅ Single source of truth (no duplicate maintenance)
- ✅ Automatic host discovery
- ✅ Role-based grouping
- ✅ Custom field support
- ✅ Asset tracking integration

## Migration from Static Inventory

1. Backup current inventory:
```bash
cp playbooks/inventory.ini playbooks/inventory.ini.backup
```

2. Import existing hosts to NetBox (one-time):
```bash
scripts/import-to-netbox.py playbooks/inventory.ini
```

3. Switch to NetBox inventory:
```bash
# Test
ansible-playbook -i scripts/netbox-inventory.py playbooks/playbook-baseball-card.yml --check

# Production
scripts/sync-netbox.sh
```

## Custom Fields

Map custom NetBox fields to Ansible variables:

NetBox Field → Ansible Variable
- `monitoring_enabled` → `netbox_monitoring_enabled`
- `backup_schedule` → `netbox_backup_schedule`
- `maintenance_window` → `netbox_maintenance_window`

Access in playbooks:
```yaml
- name: Skip hosts without monitoring
  when: netbox_monitoring_enabled | default(true)
```
```

**Step 6: Test NetBox integration**
```bash
# Export token
export NETBOX_TOKEN="your-token"

# Run sync
scripts/sync-netbox.sh

# Verify snapshot generated
cat snapshots/baseball-card-$(date +%Y-%m-%d).md
```

**Success Criteria:**
- Dynamic inventory script pulls from NetBox
- Playbooks work with NetBox inventory
- Documentation complete
- Migration path from static inventory clear

---

### Task 4.2: Ansible Molecule Testing Framework
**Goal:** Automated testing for playbooks

**Implementation:**

**Step 1: Install Molecule**
```bash
pip3 install molecule[docker] molecule-plugins[docker]
```

**Step 2: Initialize Molecule scenario**
```bash
cd /Users/kellen/_code/UncertainMeow/infra-snapshot

# Create molecule scenario
molecule init scenario default -d docker
```

**Step 3: Configure Molecule**

**File:** `molecule/default/molecule.yml`
```yaml
dependency:
  name: galaxy
driver:
  name: docker
platforms:
  - name: debian-test
    image: geerlingguy/docker-debian11-ansible:latest
    pre_build_image: true
    privileged: true
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    command: /lib/systemd/systemd
  - name: ubuntu-test
    image: geerlingguy/docker-ubuntu2204-ansible:latest
    pre_build_image: true
    privileged: true
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    command: /lib/systemd/systemd
provisioner:
  name: ansible
  inventory:
    group_vars:
      all:
        ansible_python_interpreter: /usr/bin/python3
verifier:
  name: ansible
```

**Step 4: Create test playbook**

**File:** `molecule/default/converge.yml`
```yaml
---
- name: Converge
  hosts: all
  gather_facts: yes

  pre_tasks:
    - name: Install required packages
      ansible.builtin.apt:
        name:
          - procps
          - iproute2
          - systemd
        state: present
        update_cache: yes

  tasks:
    - name: Include facts_basic role
      ansible.builtin.include_role:
        name: facts_basic

    - name: Verify facts collected
      ansible.builtin.assert:
        that:
          - ansible_facts.ansible_hostname is defined
          - uptime_output is defined
          - disk_usage is defined
          - memory_usage is defined
        fail_msg: "Required facts not collected"
        success_msg: "✅ All facts collected successfully"
```

**Step 5: Create verification tests**

**File:** `molecule/default/verify.yml`
```yaml
---
- name: Verify
  hosts: all
  gather_facts: no

  tasks:
    - name: Check registered variables exist
      ansible.builtin.assert:
        that:
          - hostvars[inventory_hostname].ping_result is defined
          - hostvars[inventory_hostname].uptime_output is defined
          - hostvars[inventory_hostname].disk_usage is defined
        fail_msg: "Required variables not registered"
        success_msg: "✅ Variables registered correctly"

    - name: Verify uptime format
      ansible.builtin.assert:
        that:
          - hostvars[inventory_hostname].uptime_output.stdout is regex('^up .*')
        fail_msg: "Uptime format invalid"
        success_msg: "✅ Uptime format valid"

    - name: Verify disk usage is percentage
      ansible.builtin.assert:
        that:
          - hostvars[inventory_hostname].disk_usage.stdout is regex('^\d+%$')
        fail_msg: "Disk usage format invalid"
        success_msg: "✅ Disk usage format valid"
```

**Step 6: Run Molecule tests**
```bash
# Full test cycle
molecule test

# Individual steps
molecule create    # Create test containers
molecule converge  # Run playbook
molecule verify    # Run verification tests
molecule destroy   # Cleanup

# Debug mode
molecule converge -- -vvv
molecule login -h debian-test  # SSH into test container
```

**Step 7: Add Molecule to CI/CD**

**File:** `.github/workflows/test.yml`
```yaml
name: Molecule Tests
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install dependencies
        run: |
          pip install molecule[docker] molecule-plugins[docker] ansible-lint

      - name: Run Molecule tests
        run: molecule test

      - name: Run ansible-lint
        run: ansible-lint playbooks/
```

**Success Criteria:**
- Molecule tests run successfully
- Both Debian and Ubuntu platforms tested
- Verification tests pass
- CI/CD pipeline configured

---

### Task 4.3: CI/CD Pipeline (GitHub Actions)
**Goal:** Automated testing and quality checks

**Implementation:**

**File:** `.github/workflows/ci.yml`
```yaml
name: CI Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lint:
    name: Lint Playbooks
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install ansible-lint
        run: pip install ansible-lint

      - name: Run ansible-lint
        run: ansible-lint playbooks/ roles/

  test:
    name: Molecule Tests
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install dependencies
        run: |
          pip install molecule[docker] molecule-plugins[docker]

      - name: Run Molecule tests
        run: molecule test

  validate-templates:
    name: Validate Jinja2 Templates
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install jinja2-cli
        run: pip install jinja2-cli pyyaml

      - name: Validate template syntax
        run: |
          for template in templates/*.j2; do
            echo "Validating $template..."
            jinja2 "$template" --format=yaml <(echo "{}") > /dev/null
          done

  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'config'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  docs:
    name: Build Documentation
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Check documentation links
        uses: gaurav-nelson/github-action-markdown-link-check@v1
        with:
          use-quiet-mode: 'yes'
          use-verbose-mode: 'yes'
```

**File:** `.github/workflows/release.yml`
```yaml
name: Release
on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    name: Create Release
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Generate changelog
        id: changelog
        uses: metcalfc/changelog-generator@v4
        with:
          myToken: ${{ secrets.GITHUB_TOKEN }}

      - name: Create Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
          body: ${{ steps.changelog.outputs.changelog }}
          draft: false
          prerelease: false
```

**Success Criteria:**
- CI pipeline runs on every push
- Linting catches syntax errors
- Tests run automatically
- Security scanning enabled
- Documentation validation working

---

### Phase 4 Completion Checklist
- [ ] NetBox integration implemented
- [ ] Dynamic inventory working
- [ ] Molecule testing framework configured
- [ ] CI/CD pipeline active
- [ ] Security scanning enabled
- [ ] Release automation working
- [ ] All tests passing

**Phase 4 Deliverables:**
- Enterprise integration (NetBox)
- Comprehensive test coverage (Molecule)
- Automated CI/CD pipeline
- Release automation

---

## Quick Reference

### File Locations
```
Key Files:
- Playbooks: playbooks/playbook-{baseball-card,work-friend}.yml
- Templates: templates/{baseball-card,work-friend}.md.j2
- Inventory: playbooks/inventory.ini (create from inventory.example.ini)
- Docs: README.md, docs/

Critical Lines:
- SSH settings: inventory:44,206
- Template refs: playbook-baseball-card.yml:103, playbook-work-friend.yml:278
- Hardcoded paths: playbook-baseball-card.yml:97, playbook-work-friend.yml:273
- Duplicate code: playbook-baseball-card.yml:13-88 ≈ playbook-work-friend.yml:13-87
```

### Common Commands
```bash
# Test connectivity
ansible -i playbooks/inventory.ini ssh_accessible -m ping

# Run baseball card snapshot
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml

# Run work friend snapshot
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-work-friend.yml

# With vault (after Phase 1.2)
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml --ask-vault-pass

# Limit to specific host
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml --limit socrates

# Parallel execution
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml --forks=10

# Dry run
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml --check

# View output
cat snapshots/baseball-card-$(date +%Y-%m-%d).md
```

### Environment Details
```
Infrastructure:
- 11 hosts total (6 Proxmox, 3 MS-01 nodes, 1 NAS, 1 misc)
- VLANs: 1 (network), 2 (trusted), 3 (lab), 4 (production), 6 (staging), 9 (VPN)
- Domain: doofus.co
- Subnet: 10.203.0.0/16

Technologies:
- Proxmox hosts (some with GPUs)
- LXC containers
- Docker containers
- Technitium DNS cluster
- Authentik SSO
- NetBox IPAM (10.203.4.16)
- GitLab (10.203.4.15)

Key Hosts:
- socrates (10.203.3.42) - Dell R730 with 2x GPUs for AI
- rawls (10.203.3.47) - General purpose Proxmox
- stuffs (10.203.3.99) - Primary NAS
- ms01-node{1,2,3} (10.203.3.11-13) - Thunderbolt cluster
```

### Testing Strategy
```
Phase 1 (Security):
1. Test each change on single host first (--limit socrates)
2. Backup inventory before modifications
3. Verify SSH access after each security change
4. Keep rollback plan ready

Phase 2 (Quality):
1. Create test branch for refactoring
2. Verify output identical before/after
3. Run ansible-lint continuously
4. Test with subset of hosts first

Phase 3 (Operations):
1. Benchmark current performance (time playbook runs)
2. Test incremental mode with controlled changes
3. Validate metrics against Prometheus
4. Monitor for performance regressions

Phase 4 (Integrations):
1. Test NetBox integration in non-production first
2. Run Molecule tests before merging
3. Use CI/CD dry-run mode initially
4. Gradual rollout of automations
```

### Troubleshooting Guide
```
Issue: "Host unreachable"
→ Check: ansible -i playbooks/inventory.ini <host> -m ping
→ Fix: Verify SSH keys, check network connectivity

Issue: "Template not found"
→ Check: ls -la templates/
→ Fix: Verify paths in playbook match template location

Issue: "Permission denied"
→ Check: ansible_become=yes in inventory
→ Fix: Add sudo privileges for ansible-reader user

Issue: "Vault password failed"
→ Check: ansible-vault view playbooks/inventory.ini
→ Fix: Verify correct password, check .vault_pass file

Issue: "Ansible-lint errors"
→ Check: ansible-lint playbooks/
→ Fix: Follow lint recommendations, update .ansible-lint config

Issue: "Snapshot missing data"
→ Check: Run with -vvv for debug output
→ Fix: Verify commands execute successfully on target hosts
```

### Success Metrics
```
Phase 1 Goals:
- Zero security vulnerabilities (Trivy scan clean)
- 100% test coverage with validation
- <1min playbook execution overhead from changes

Phase 2 Goals:
- 50% reduction in code duplication (measured by lines)
- Zero ansible-lint warnings
- 100% of errors visible in snapshots

Phase 3 Goals:
- 50%+ time reduction for incremental snapshots
- Metrics exported to Prometheus (100% coverage)
- Support for 2+ OS families (Debian + RedHat)

Phase 4 Goals:
- NetBox as single source of truth (0 manual inventory edits)
- 90%+ test coverage (Molecule)
- CI/CD pipeline <5min execution time
```

---

## Next Steps for New Claude Instance

### Day 1: Get Context
1. Read this entire document carefully
2. Review architecture review findings (sections 1-10 above)
3. Examine current codebase:
   - Read all playbooks (playbooks/*.yml)
   - Review templates (templates/*.j2)
   - Check inventory structure (playbooks/inventory.example.ini)
4. Run playbooks to understand current behavior:
```bash
cd /Users/kellen/_code/UncertainMeow/infra-snapshot
ansible-playbook -i playbooks/inventory.ini playbooks/playbook-baseball-card.yml --check
```

### Day 1: Verify Understanding
Ask the user:
- Which phase should we prioritize? (Default: Phase 1)
- Are there specific security concerns to address first?
- Any constraints (time, access, testing environment)?
- NetBox API token available for Phase 4?

### Day 1-2: Phase 1 Execution
Start with Phase 1, Task 1.1 (SSH host key checking):
1. Confirm user wants to proceed with security hardening
2. Create backups of all files before modifications
3. Execute tasks sequentially (1.1 → 1.2 → 1.3 → 1.4 → 1.5)
4. Test after each task, don't batch changes
5. Document any issues encountered
6. Get user approval before moving to Phase 2

### Week 2-3: Phase 2 Execution
After Phase 1 complete:
1. Create feature branch: `git checkout -b phase-2-code-quality`
2. Extract roles (Task 2.1)
3. Implement error reporting (Task 2.2)
4. Add validation (Task 2.3)
5. Configure pre-commit hooks (Task 2.4)
6. Merge after testing

### General Principles
- **Ask before breaking changes:** Always confirm destructive operations
- **Test incrementally:** One task at a time, verify before proceeding
- **Document decisions:** If you deviate from plan, explain why
- **Backup first:** Copy files before editing
- **Use version control:** Commit after each completed task
- **Communicate clearly:** Show progress, surface issues early

### Red Flags to Watch For
- Any task taking 2x longer than estimated → Ask user if we should pause/reassess
- Security changes breaking connectivity → Roll back immediately
- Tests failing after refactoring → Don't proceed until fixed
- User seems uncertain about changes → Stop and clarify expectations

---

## Project Philosophy & Context

### User's Goals
From README.md:518-522:
> Built with ☕ and 🤖 by Kellen & Claude
> Status: Production-ready (used in real homelab since 2025-12-05)
> Philosophy: "If you can't measure it, you can't manage it. If you can't snapshot it, you can't describe it to AI without looking silly."

**Translation for Next Claude:**
- This is a REAL, PRODUCTION homelab (not a toy project)
- User values facts over hand-waving
- Tool is FOR AI agents (you and future Claudes)
- Balance pragmatism with quality (homelab-ready → enterprise-grade)

### Current Infrastructure State
From inventory.example.ini:238-239:
> infrastructure_state=post_thanksgiving_rebuild
> notes="Mid-rebuild: rseau removed, MS-01 nodes being added, network reset not yet started"

**What This Means:**
- Infrastructure in flux (rebuilding)
- Some hosts may be offline/unreachable
- MS-01 nodes (10.203.3.11-13) might not respond yet
- Don't be surprised by partial failures
- User expects this, handle gracefully

### Communication Style
From CLAUDE.md context:
> README&docs_tone = Remember, in the README.md and similar documents and documentation, you're not writing TO ME - the readme is more ME talking TO OTHERS. So please keep that in mind in tone and structure of the readme and similar documents. Use a professional, neutral tone that presents projects as production-ready infrastructure.

**Translation for Next Claude:**
- Documentation = user speaking to future users (not you speaking to user)
- Use professional, neutral tone in docs
- Present as production-ready (because it is)
- Skip emojis unless user requests them
- Be concise and actionable

### Expected Work Style
Based on architecture review process:
- User values thoroughness (comprehensive 10-section architecture review)
- User wants actionable plans (not just analysis)
- User appreciates specificity (file paths, line numbers, code examples)
- User expects you to work autonomously (this handoff doc proves it)

---

## Appendix: Architecture Review Summary

### Overall Grade: B+ (Very Good)
**Justification:**
- Solid architectural foundation (ETL pipeline pattern)
- Production-ready for intended use case (homelab)
- Clean separation of concerns
- Stateless design (excellent reliability)
- Good documentation and examples

**Improvement Path:** Homelab-ready → Enterprise-grade via 4-phase plan

### Critical Security Issues (Fix First)
1. SSH host key checking disabled (MITM vulnerability)
2. Broad root access (privilege escalation risk)
3. Secrets in plaintext (repository compromise risk)

### Key Technical Debt
1. Code duplication (70 lines duplicated between playbooks)
2. Missing template validation
3. Hardcoded magic numbers
4. No error aggregation
5. Limited extensibility

### Architectural Strengths to Preserve
- Two-phase pipeline pattern (keep this!)
- Stateless execution (don't add state!)
- Template-based output (maintain flexibility)
- Role-based inventory grouping (expand on this)

### Performance Characteristics
- Current: 30s (baseball card), 2-3min (work friend)
- Scalability: 1-50 hosts comfortable
- Bottlenecks: Sequential command execution, log parsing
- Optimization opportunity: Incremental mode (Phase 3.2)

---

**End of Implementation Plan**

This document is comprehensive and complete. The next Claude Code instance has everything needed to:
1. Understand the project architecture
2. Know the current state and issues
3. Follow the prioritized implementation roadmap
4. Execute each task with specific code examples
5. Test and validate their work
6. Communicate effectively with the user

Good luck! 🚀
