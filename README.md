# Hybrid Infrastructure Lab

Two virtual machines, one Linux and one Windows, configured from a third
place that is neither of them. Nothing is installed on either machine to
make that possible, and nobody logs in to change anything: one command
from the Mac, and both end up in the state a file says they should be in.

It runs entirely on an Apple Silicon laptop and costs nothing. It exists
to answer a question with something better than a definition: what does
configuration management actually do that a shell script doesn't?

"Hybrid" here means two operating systems managed as if they were
machines in a server room, not instances in a cloud.

## The idea, in one line

Write down how a machine should look, and let Ansible make it true — on
Linux over SSH, on Windows over WinRM, from the same repo.

The part worth seeing is what happens when someone breaks it on purpose.
A security setting gets flipped by hand on the Windows machine, Ansible
notices on the next run, shows exactly what changed, and puts it back.

## Architecture

The Mac is the control node. It runs Ansible, holds the inventory, the
playbooks and the encrypted vault, and talks to two UTM virtual machines
on UTM's shared network (`192.168.64.0/24`), a private range behind NAT
inside the Mac that nothing outside can reach. Both guests run as native
ARM64 on Apple's hypervisor, with no emulation anywhere.

The Ubuntu Server 26.04 machine is reached over SSH with a key, and
password login is switched off by the playbook itself. The Windows 11
machine is reached over WinRM on port 5985 with NTLM authentication.
Ansible is agentless, so there is no daemon running on either guest: it
uses whatever each operating system already ships for remote
administration, SSH on one side and WinRM on the other.

The repo is small on purpose. `inventory/hosts.yml` defines two groups,
`linux` and `windows`, each with its own connection settings.
`inventory/group_vars/all/vault.yml` holds the two passwords, encrypted.
`playbooks/` has one baseline per operating system.

## What's actually worth reading here

**Idempotency is easy to claim and more convincing to watch.** The first
Linux task installs `tree`, which isn't there, so the first run reports
`changed=1`. The second run, with nothing edited, reports `changed=0`.
The playbook never says "install tree"; it says "tree should be present,"
and if that is already true, nothing happens. A shell script would try
again.

**The drift test is the point of the whole lab.** The Windows playbook
sets one registry value, SMBv1 server off. Windows 11 already ships with
it disabled, but only by default: no value is stored at all, so nothing
on the machine records that anyone decided it. The first run writes the
value (`changed=1`) and makes it a decision. Then, from an elevated
PowerShell on the VM, `Set-ItemProperty` flips it back on by hand. Back
on the Mac, `--check --diff` reports exactly what would change and
touches nothing:

```
--- before
+++ after
@@ -2,7 +2,7 @@
     "HKLM:\\SYSTEM\\CurrentControlSet\\Services\\LanmanServer\\Parameters": {
         "SMB1": {
             "type": "DWord",
-            "value": "0x00000001"
+            "value": "0x00000000"
```

Without `--check`, the same command puts it back, and the run after that
is clean. The limit is worth stating plainly: Ansible watches nothing on
its own. It corrects drift when something runs it, which in practice means
cron, `ansible-pull`, a scheduled CI job or AWX. Running `--check --diff`
on a schedule and alerting when it reports a change is the careful way to
start, because correcting automatically can overwrite a fix someone made
on purpose.

**Turning off SSH passwords is harder than it looks, because editing the
obvious file does nothing.** Ubuntu's `sshd_config` includes
`sshd_config.d/*.conf` near the top, and sshd keeps the first value it
reads for any setting, so a file edited in the main config can quietly
lose to a drop-in that was already there. The playbook writes its own
`10-hardening.conf`, which sorts ahead of `50-cloud-init.conf` and wins.
It validates the file with `sshd -t` before putting it in place, and a
handler reloads SSH only when the file actually changed. The proof is a
forced password attempt, which gets back `Permission denied (publickey)`
instead of a prompt. The server no longer offers passwords as an option at
all.

**Windows on ARM inside UTM pushed back at every step.** The first boot
showed "Display output is not active," and the cause was a single
unchecked box: UEFI Boot was off, and Windows on ARM only boots through
UEFI. The installer then sat on the firmware logo for over half an hour,
because the ISO was attached as emulated USB storage and the boot never
advanced. Dropping to the UEFI shell and starting the installer by hand
(`FS0:`, then `EFI\BOOT\bootaa64.efi`) got it moving. Windows setup then
had no network, since Windows doesn't ship a driver for the virtio network
card; UTM Guest Tools is what provides it.

**Ansible worked on Linux and died on Windows, from the same Mac.** Every
Windows run ended with `A worker was found in a dead state`. macOS kills
forked processes that touch certain system libraries, and the WinRM
library reads the system proxy settings, while Ansible forks worker
processes to do its work. SSH never triggers it, which is why Linux was
fine the whole time. Setting `OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES`
fixes it.

**Ubuntu 26.04 broke privilege escalation in a way that looked like a
wrong password.** The error was `Timeout waiting for privilege escalation
prompt`, and the password was correct. Ubuntu now ships `sudo-rs` as the
default `sudo`, and Ansible doesn't recognize its prompt, so it waits
until it times out. The classic `sudo` is still installed beside it as
`sudo.ws`, and pointing the inventory at it with `ansible_become_exe`
fixed it.

**A few choices here only make sense in a lab, and they were made on
purpose.** `host_key_checking` is off, because rebuilt VMs get new host
keys and would prompt every time; in production it stays on. WinRM runs
over plain HTTP, which is tolerable because NTLM encrypts the message
payload and the traffic never leaves the Mac, but production wants HTTPS
on 5986 with a real certificate. The passwords are never in the inventory:
they are `{{ vault_* }}` references to an encrypted file, and the file that
holds the vault password is in `.gitignore`. Before the first commit, the
vault was checked to confirm it starts with `$ANSIBLE_VAULT` rather than
being plain text.

## Cost

Nothing. UTM and Ansible are free, the Windows 11 ARM64 image is a free
download from Microsoft, and no cloud account is involved. It is the only
project in this portfolio without a kill-switch, because there is nothing
here that can run up a bill.

## Deploying it

This assumes two running VMs on the same UTM shared network, your SSH key
already copied to the Ubuntu one, and a Mac with Homebrew.

```bash
brew install ansible
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
```

On the Windows VM, in an elevated PowerShell. `Enable-PSRemoting` refuses
to run without elevation, and refuses on a network profile set to Public:

```powershell
Set-NetConnectionProfile -InterfaceAlias Ethernet -NetworkCategory Private
Enable-PSRemoting -Force
```

Back on the Mac, put the two IPs in `inventory/hosts.yml`, then create the
vault with the passwords it refers to:

```bash
echo "a long passphrase only you know" > .vault_pass
ansible-vault create inventory/group_vars/all/vault.yml
```

```yaml
vault_ubuntu_sudo_password: "..."
vault_windows_password: "..."
```

Check both connections, then apply. Running either playbook a second time
should report `changed=0`.

```bash
ansible linux -m ping
ansible windows -m ansible.windows.win_ping

ansible-playbook playbooks/linux-baseline.yml
ansible-playbook playbooks/windows-baseline.yml
```

## What's next

Running `--check --diff` on a schedule and alerting when it reports a
change, so drift gets noticed without anyone having to ask. Moving WinRM
from HTTP to HTTPS on 5986, which is the honest fix for the one setting
above that is lab-only for a real reason. Growing both baselines past a
single setting each: firewall rules and local accounts on Windows, `ufw`
and unattended upgrades on Linux. Splitting into roles once two playbooks
stop being enough. And an architecture diagram in the same style as the
other projects, which this one doesn't have yet.
