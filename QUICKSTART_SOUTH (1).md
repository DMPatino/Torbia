# South Torbia — Quick Deploy (from anywhere on the range)

Works from any control node that can SSH to the boxes as `csra_admin`.
Since the networks touch and `csra_admin` is on every box, any machine with the
kit + Ansible can drive it.

## 1. One-time setup on the control node
```bash
python3 -m pip install --user ansible pywinrm
ansible-galaxy collection install community.general ansible.windows microsoft.ad
# only if you use SSH *passwords* (not keys):
sudo apt install -y sshpass   # or: sudo dnf install -y sshpass
```

## 2. Get the kit
```bash
tar -xzf ansible-deploy.tar.gz
cd ansible-deploy
```

## 3. Linux boxes — remove old + create roster (one command)
```bash
./provision_range.sh
```
Answer the prompts:
- **Roster CSV:** press Enter (defaults to `ansible_south_torbia_users.csv`)
- **IP ranges:** e.g. `10.10.20.0/24  10.10.21.10-50  10.10.22.5`
- **SSH password:** blank if key auth
- **sudo password:** blank if NOPASSWD
Then read the DRY-RUN list and type **YES** to proceed. `csra_admin` is always kept.

## 4. Domain Controller (WinRM — run these three)
```bash
# a) remove old users in the exercise OU (dry-run first, then add -e confirm=yes)
ansible-playbook -i inventory.ini cleanup_windows_ad.yml \
  -e "target_group=south_dc" \
  -e "search_base=OU=Exercise Users,DC=southtorbia,DC=range" \
  -e "admin_user=csra_admin" --ask-pass

# b) let weak rockyou passwords import
ansible-playbook -i inventory.ini relax_ad_password_policy.yml \
  -e "target_group=south_dc" --ask-pass

# c) create the roster
ansible-playbook -i inventory.ini deploy_windows_ad.yml \
  -e "csv_file=ansible_south_torbia_users.csv" \
  -e "target_group=south_dc" \
  -e "domain_dn=DC=southtorbia,DC=range" \
  -e "upn_suffix=southtorbia.range" --ask-pass
```
(Set `south_dc` host in `inventory.ini` first. If you run these *on* the DC, add
`ansible_connection=local` to that host and drop `--ask-pass`.)

## 5. Verify
```bash
ansible -i inventory.ini south_dc -m ansible.windows.win_shell \
  -a "Get-ADUser -Filter * -SearchBase 'OU=Exercise Users,DC=southtorbia,DC=range' | Select SamAccountName,Enabled" --ask-pass
```
Spot-check: `seoah.lee` = `dragon`, `eunwoo.jeong` = `iloveyou`.

---
That's it. Linux = one script; DC = three plays.
