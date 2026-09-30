# Hybrid Infrastructure Lab

Two virtual machines, one Linux and one Windows, configured from a third
place that is neither of them. Nothing is installed on either machine
to make that possible, and nobody logs in to change anything: one command
from the Mac, and both end up in the state a file says they should be in.

It runs entirely on an Apple Silicon laptop, costs nothing, and exists to
answer a question with something better than a definition: what does
configuration management actually do that a shell script doesn't?

"Hybrid" here means two operating systems managed as if they were
machines in a server room, not instances in a cloud.

## The idea, in one line

Write down how a machine should look, and let Ansible make it true —
on Linux over SSH, on Windows over WinRM, from the same repo.

The part worth seeing is what happens when someone breaks it on purpose.
A security setting gets flipped by hand on the Windows VM, Ansible notices
on the next run, shows exactly what changed, and puts it back.

## Architecture

```
              Mac (Apple Silicon)
              Ansible control node
              inventory + vault + playbooks
                 |                  |
         SSH, key only        WinRM :5985, NTLM
                 |                  |
      +----------v---------+  +-----v--------------+
      | ubuntu-lab         |  | windows-lab        |
      | Ubuntu Server 26.04|  | Windows 11         |
      | LTS, ARM64         |  | ARM64              |
      | 192.168.64.2       |  | 192.168.64.4       |
      +--------------------+  +--------------------+
        UTM "Shared Network", 192.168.64.0/24
        (NAT inside the Mac, not reachable from outside)
```

| Piece | What it is |
|---|---|
| Control node | The Mac itself, running Ansible core 2.21 from Homebrew |
| Virtual machines | UTM, using Apple's hypervisor, so both guests run as native ARM64 with no emulation |
| Linux connection | SSH with an ed25519 key. Password login is disabled by the playbook |
| Windows connection | WinRM over HTTP on port 5985, NTLM authentication |
| Secrets | Ansible Vault (AES256). The vault password lives in a file that Git ignores |

Ansible is agentless: there is no daemon on either VM. It uses what each
operating system already ships for remote administration, which is SSH on
Linux and WinRM on Windows.

```
.
├── ansible.cfg
├── inventory/
│   ├── hosts.yml                  two groups: linux, windows
│   └── group_vars/all/vault.yml   encrypted
└── playbooks/
    ├── linux-baseline.yml
    └── windows-baseline.yml
```

## What's actually worth reading here

### Idempotency, shown instead of claimed

The first Linux task installs `tree`. It is not installed, so the first run
reports `changed=1`. The second run, with nothing edited, reports
`changed=0`.

```
ubuntu-lab : ok=2  changed=1     first run
ubuntu-lab : ok=2  changed=0     second run
```

The playbook doesn't say "install tree", it says "tree should be present".
If that is already true, nothing happens. A shell script would try again.

### Drift, on purpose

The Windows playbook sets one registry value: SMBv1 server off. Windows 11
already ships with SMBv1 disabled, but by default there is no value stored
at all. The first run writes one (`changed=1`), so the decision is on
record instead of being an accident of defaults.

Then the experiment. From an elevated PowerShell on the VM:

```powershell
Set-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters -Name SMB1 -Value 1
```

Back on the Mac, in check mode, which only reports and touches nothing:

```
$ ansible-playbook playbooks/windows-baseline.yml --check --diff

--- before
+++ after
@@ -2,7 +2,7 @@
     "HKLM:\\SYSTEM\\CurrentControlSet\\Services\\LanmanServer\\Parameters": {
         "SMB1": {
             "type": "DWord",
-            "value": "0x00000001"
+            "value": "0x00000000"
```

Without `--check` the same command puts it back (`changed=1`), and the run
after that is clean (`changed=0`).

One limit worth stating plainly: Ansible does not watch anything by
itself. It corrects drift when something runs it. In practice that means
cron, `ansible-pull`, a scheduled CI job, or AWX. Running `--check --diff`
on a schedule and alerting on `changed > 0` is the cautious way to start,
because auto-correcting can overwrite a fix someone made on purpose.

### Turning off SSH passwords, and why the obvious way fails

Ubuntu's `sshd_config` includes `sshd_config.d/*.conf` near the top, and
sshd keeps the first value it reads for any setting. Editing the main file
can quietly lose to a drop-in that was already there. So the playbook
writes its own `10-hardening.conf`, which sorts before `50-cloud-init.conf`
and wins.

The file is validated with `sshd -t` before it is put in place, and SSH is
reloaded by a handler only when the file actually changed. The proof is a
forced password attempt:

```
$ ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password alvaro-lab@192.168.64.2
alvaro-lab@192.168.64.2: Permission denied (publickey).
```

The server no longer offers `password` as an option at all.

### What broke along the way

None of this is in the tutorials, and all of it cost time.

- **Windows showed "Display output is not active."** UEFI Boot was off in
  the UTM VM settings. Windows on ARM only boots through UEFI.
- **The installer sat on the firmware logo for over half an hour.** The ISO
  was attached as emulated USB storage and the boot never advanced. Dropping
  to the UEFI shell and starting the installer by hand worked:
  `FS0:`, then `EFI\BOOT\bootaa64.efi`.
- **No network in Windows setup.** Windows doesn't ship a driver for the
  virtio network card. UTM Guest Tools provides it.
- **`A worker was found in a dead state` on every Windows run.** macOS kills
  forked processes that touch certain system libraries, and the WinRM
  library reads the system proxy settings. Ansible forks one worker per
  host. Linux over SSH never triggers it.
  `export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES` fixes it.
- **`Timeout waiting for privilege escalation prompt` on Ubuntu.** Ubuntu
  26.04 ships `sudo-rs` as the default `sudo`, and Ansible doesn't
  recognize its password prompt. The classic `sudo` is installed alongside
  it as `sudo.ws`, so the inventory sets `ansible_become_exe: sudo.ws`.
- **`Enable-PSRemoting` refused to run.** It needs an elevated PowerShell,
  and it won't enable remoting on a network profile set to Public.

### Choices that only make sense in a lab

- `host_key_checking = False`. Rebuilt VMs get new host keys and would
  prompt every time. In production this stays on.
- WinRM over plain HTTP. NTLM encrypts the message payload and the traffic
  never leaves the Mac, but production wants HTTPS on 5986 with a real
  certificate.
- Passwords never appear in the inventory. They are `{{ vault_* }}`
  references to an encrypted file, and the vault password file is in
  `.gitignore`. Before the first commit, the vault file was checked to
  confirm it starts with `$ANSIBLE_VAULT` and isn't plain text.

## Cost

Nothing. UTM is free, Ansible is free, the Windows 11 ARM64 image is a free
download from Microsoft, and there is no cloud account involved.

It is the only project in this portfolio without a kill-switch, because
there is nothing here that can run up a bill.

## Deploying it

Assumes two running VMs on the same UTM shared network, your SSH key
copied to the Ubuntu VM, and Ansible installed on the Mac.

```bash
brew install ansible
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
```

On the Windows VM, in an elevated PowerShell:

```powershell
Set-NetConnectionProfile -InterfaceAlias Ethernet -NetworkCategory Private
Enable-PSRemoting -Force
```

On the Mac, set the IPs in `inventory/hosts.yml`, then create the vault
with the two passwords it references:

```bash
echo "a long passphrase only you know" > .vault_pass
ansible-vault create inventory/group_vars/all/vault.yml
```

```yaml
vault_ubuntu_sudo_password: "..."
vault_windows_password: "..."
```

Check both connections, then apply:

```bash
ansible linux -m ping
ansible windows -m ansible.windows.win_ping

ansible-playbook playbooks/linux-baseline.yml
ansible-playbook playbooks/windows-baseline.yml
```

Run either playbook a second time and it should report `changed=0`.

## What's next

- Run `--check --diff` on a schedule and alert when it reports a change.
- Move WinRM to HTTPS on 5986.
- Grow both baselines: firewall rules and local accounts on Windows,
  `ufw` and unattended upgrades on Linux.
- Split into roles once two playbooks stop being enough.
- An architecture diagram in the same style as the other projects.
