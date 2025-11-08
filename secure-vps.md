## Secure VPS setup commands

(Information origin from: codinginflow/next-self-hosting-instructions.md)

This document lists the exact commands and checks to:

- update packages as root
- create a new sudo user and verify sudo access
- set up SSH keys and permissions
- disable root login and password authentication
- configure UFW firewall and add an ICMP drop rule
- safely reboot and verify everythingk

IMPORTANT SAFETY NOTES

- Do NOT disable root or password authentication until the new user is created, has sudo, and you have working SSH key access for that user. If you are on a cloud VPS, ensure you have console access (provider web console / rescue mode) in case you lock yourself out.
- Always backup config files before editing.
- Test SSH restart and a second shell before closing your current session.

--

# 1. Login as root and update packages

-- On your local machine (PowerShell / macOS / Linux terminal) run:

`ssh root@<your-server-ip>`

Once connected as root, run:

`apt update && apt upgrade -y`

Optional: reboot if the kernel was upgraded:

`reboot`

# 2. Create a new administrative user (on the server)

Replace `<username>` with your chosen username.

`adduser <username>`

-- follow prompts to set a password and optional info

Add the user to the sudo group:

`usermod -aG sudo <username>`

Verify the user works (switch to the user and verify sudo):

`su - <username>`
`sudo -v`

If `sudo -v` completes without an error, sudo is configured.

# 3. Create the SSH folder and set permissions (on the server, as the new user)

Run as the new user (if you are still root, switch to the user first):

`su - <username>`

`mkdir -p ~/.ssh`

`chmod 700 ~/.ssh`

`ls -la ~ | grep .ssh`

Check ownership:

`ls -ld ~/.ssh`

-- should show the new username as owner

In case problem in login using new user, follow this steps:

`chown -R <newuser>:<newuser> /home/<newuser>/.ssh`

`chmod 700 /home/<newuser>/.ssh`

`chmod 600 /home/<newuser>/.ssh/authorized_keys`

# 4. Generate an SSH key on your local machine

On Windows (PowerShell, recommended):

`ssh-keygen -t ed25519 -C "your_email@example.com"`

This creates `~/.ssh/id_ed25519` (private) and `~/.ssh/id_ed25519.pub` (public).

On macOS / Linux (bash):

`ssh-keygen -t ed25519 -C "your_email@example.com"`

# 5. Copy the public key to the server (as new user)

Notes: ensure `~/.ssh` and `~/.ssh/authorized_keys` are owned by the new user and have correct permissions.

Windows (PowerShell) scp example:

-- from PowerShell

`scp $env:USERPROFILE/.ssh/id_ed25519.pub <username>@<your-server-ip>:~/.ssh/authorized_keys`

macOS / Linux:

`ssh-copy-id <username>@<your-server-ip>`

Or manually (macOS/Linux):

`cat ~/.ssh/id_ed25519.pub | ssh <username>@<your-server-ip> 'umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys'`

After copying, ensure permissions on the server (as the new user):

`chmod 700 ~/.ssh`

`chmod 600 ~/.ssh/authorized_keys`

`chown -R <username>:<username> ~/.ssh`

Test you can login with the new user (in a new local terminal):

`ssh <username>@<your-server-ip>`

Verify sudo works:

`sudo -v`

# 6. Backup SSH configuration files

Before editing, backup:

`sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak`

`sudo mkdir -p /etc/ssh/sshd_config.d/backup`

`for f in /etc/ssh/sshd_config.d/\*.conf; do [ -e "$f" ] && sudo cp "$f" "$f.bak"; done`

# 7. Edit SSH configuration to harden (do NOT do this until step 5 validated)

Open the main file:

`sudo nano /etc/ssh/sshd_config`

Make sure the following lines exist (uncomment or add them). Use the exact values below:

`AddressFamily inet`

`PasswordAuthentication no`

`PermitRootLogin no`

Notes:

- If `AddressFamily` was commented (#), remove the `#`.
- `PermitRootLogin no` prevents root SSH login.
- `PasswordAuthentication no` disables password auth entirely (ensure key auth works for the new user first).

If your system uses `sshd_config.d` snippets and you were instructed to "make it empty", back them up and then empty them (only do this if you understand the consequences):

`sudo sh -c 'for f in /etc/ssh/sshd_config.d/\*.conf; do [ -e "$f" ] && cp "$f" "$f.bak" && truncate -s 0 "$f"; done'`

Alternative safer approach: move them into a backup folder instead of emptying:

`sudo mkdir -p /root/sshd_config_d_backup`

`sudo mv /etc/ssh/sshd_config.d/\*.conf /root/sshd_config_d_backup/ 2>/dev/null || true`

# 8. Test SSH restart safely (keep an existing session open)

-- Determine service name (some distros use `sshd`, others `ssh`)

`sudo systemctl restart ssh || sudo systemctl restart sshd`

-- In another local terminal, test logging in with your non-root user before closing your current session:

`ssh <username>@<your-server-ip>`

If login fails, revert the backup immediately (on the server via console or open session):

`sudo cp /etc/ssh/sshd_config.bak /etc/ssh/sshd_config`

`for f in /etc/ssh/sshd_config.d/\*.conf.bak; do [ -e "$f" ] && sudo mv "$f" "${f%.bak}"; done`

`sudo systemctl restart ssh || sudo systemctl restart sshd`

# 9. Install and configure UFW firewall

Install ufw (Debian/Ubuntu):

`sudo apt install ufw -y`

Allow SSH, HTTP and HTTPS:

`sudo ufw allow OpenSSH`

`sudo ufw allow http`

`sudo ufw allow https`

Enable UFW (this will prompt to proceed):

`sudo ufw enable`

Check status:

`sudo ufw status verbose`

# 10. Add ICMP echo-request DROP to UFW before.rules (manual edit recommended)

Backup before.rules:

`sudo cp /etc/ufw/before.rules /etc/ufw/before.rules.bak`

Open it with nano:

`sudo nano /etc/ufw/before.rules`

Find the INPUT block (look for `# End required lines` / chain definitions) and add the following line in the _INPUT_ block (near other `-A ufw-before-input` rules):

`-A ufw-before-input -p icmp --icmp-type echo-request -j DROP`

Important: Make only one small change and save. If unsure, restore the backup:

`sudo cp /etc/ufw/before.rules.bak /etc/ufw/before.rules`

Reload UFW after editing:

`sudo ufw reload`

# 11. Final verification and reboot

Before rebooting, verify:

- You can SSH in as `<username>` with your SSH key from a new terminal.
- `sudo -v` works for the new user.
- `ssh root@<your-server-ip>` is rejected or denied (expected once PermitRootLogin no is active).
- Attempting password-based login should fail (if `PasswordAuthentication no` is set).
- `sudo ufw status` shows rules for OpenSSH, http, https.

If all is OK, reboot the server:

`sudo reboot`

Wait for the server to come back online and log in with the user:

`ssh <username>@<your-server-ip>`

12. Post-reboot checks

Check SSH service:

`sudo systemctl status ssh || sudo systemctl status sshd`

Check UFW:

`sudo ufw status verbose`

Check that root login via SSH is disabled (from your local machine):

`ssh root@<your-server-ip>`

-- this should refuse/deny/close the connection

Check password auth is disabled by trying to SSH and forcing password auth (it should fail):

`ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no <username>@<your-server-ip>`

If anything failed and you cannot SSH back in, use your cloud provider console or rescue mode to restore `/etc/ssh/sshd_config` from the backups created above.
