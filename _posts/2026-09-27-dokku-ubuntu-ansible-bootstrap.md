---
layout: post
title: "How to Set Up Dokku on Ubuntu 24.04 with Ansible (Hardened, Re-runnable Bootstrap)"
slug: dokku-ubuntu-ansible-bootstrap
permalink: /blog/dokku-ubuntu-ansible-bootstrap/
date: 2026-09-27 09:00:00 +0300
description: "Turn a fresh Ubuntu 22.04/24.04 VPS into a hardened Dokku host with one re-runnable Ansible playbook, and the checklist steps that finished without errors but didn't do what I thought: SSH hardening, fail2ban, and installing Docker and Dokku from signed apt repositories."
tags: [ansible, dokku, ubuntu, ssh, fail2ban, docker, infrastructure]
---

Every new server for my side projects started the same way. Create a user, copy an SSH key, disable password login, enable ufw, install fail2ban, pipe Docker's and Dokku's install scripts into bash, add the plugins, turn on TLS. An afternoon of copy-pasted commands, and at the end, no way to tell whether this server matched the last one.

So I turned the checklist into an Ansible playbook, [ansible-server-bootstrap](https://github.com/thepsalmist/ansible-server-bootstrap). It takes a fresh Ubuntu 22.04 or 24.04 VPS to a hardened Dokku host: your own Heroku-style `git push` deploys, on a server you control. It's also built to converge: a second run should report `changed=0` as long as the configuration hasn't changed and no new package updates have been published (the base role runs an apt upgrade).

The harder part was that automating the checklist meant checking it. I had treated commands that finished without errors as evidence that the server was configured correctly. Was password authentication actually disabled? Could the new admin log in before I removed root access? Would installing fail2ban ban the machine running the playbook? And what would happen when I ran it all again?

A couple of steps that had always "worked" turned out not to do what I thought. This post is about those, starting with the one I'd have bet on.

## sshd keeps the first value it reads

Disabling password login is one line in `/etc/ssh/sshd_config`. On Ubuntu, that line can leave password login enabled, even after a reload that reports success. The reason is the first setting in the same file, which most people scroll past:

```
Include /etc/ssh/sshd_config.d/*.conf
```

Every drop-in in that directory is read before the rest of the main file, and for almost every setting, sshd keeps the **first** value it sees. On many VPS images, cloud-init has already written a drop-in that allows passwords:

```
sshd_config.d/50-cloud-init.conf   PasswordAuthentication yes   <- read first, wins
sshd_config (line 66)              PasswordAuthentication no    <- ignored
```

So you edit the main file, `sshd -t` passes, the reload succeeds, and password login keeps working. Nothing tells you. The only way to see it is to ask sshd what it will actually do:

```bash
sudo sshd -T | grep -Ei '^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin) '
```

The fix is to leave the main file alone. The playbook writes its settings to `/etc/ssh/sshd_config.d/00-hardening.conf`. Drop-ins are read in name order, so `00-` beats cloud-init's `50-`, and it's a file you own rather than one cloud-init manages.

## Trust the config sshd reports, not the file you wrote

Writing the right file still isn't proof. A `Match` block, a provider's own drop-in or a file named `00-aaa.conf` could override it. So the SSH role doesn't trust its own drop-in until sshd confirms it, and tries to remove it if a check fails:

{% raw %}
```yaml
- name: Harden sshd, rolling back if the change doesn't take effect
  block:
    - name: Install the sshd hardening drop-in
      ansible.builtin.template:
        src: 00-hardening.conf.j2
        dest: /etc/ssh/sshd_config.d/00-hardening.conf
        validate: /usr/sbin/sshd -t -f %s
      register: ssh_hardening_dropin

    - name: Read the effective sshd config for a root login
      ansible.builtin.command: /usr/sbin/sshd -T -C user=root,host=localhost,addr=127.0.0.1
      register: ssh_hardening_effective
      changed_when: false

    - name: Check each hardening option is in effect
      ansible.builtin.assert:
        that: (item.key ~ ' ' ~ item.value) | lower in ssh_hardening_effective.stdout_lines
        fail_msg: "{{ item.key }} is overridden elsewhere in the sshd config"
      loop: "{{ ssh_hardening_options | dict2items }}"

    - name: Reload sshd  # noqa: no-handler
      ansible.builtin.service: { name: ssh, state: reloaded }
      when: ssh_hardening_dropin is changed

    - name: Drop the current SSH connection
      ansible.builtin.meta: reset_connection

    - name: Confirm a fresh SSH login still works
      ansible.builtin.wait_for_connection: { timeout: 30 }

  rescue:
    - name: Remove the hardening drop-in
      ansible.builtin.file:
        path: /etc/ssh/sshd_config.d/00-hardening.conf
        state: absent
      when: ssh_hardening_dropin is changed
    # ...reload sshd, then fail the run
```
{% endraw %}

Every line of that is there because a simpler version would lie. `sshd -T -C user=root,host=localhost,addr=127.0.0.1` evaluates the config for that specific connection context, a root login from localhost, so `Match` blocks that apply to it are included; plain `sshd -T` skips `Match` blocks altogether. It doesn't check every possible connection, only the one it describes. The reload is a task rather than a handler, because a handler runs at the end of the play, after the check that's supposed to test it. And `reset_connection` followed by `wait_for_connection` proves a brand new login works, not just the connection Ansible already had open.

There's one more safety net before any of this runs. The first playbook, `bootstrap.yml`, logs in as root and creates the admin user, then hands over to `site.yml`, which opens a **new** connection as that admin. That's the "open a second terminal and check you can still log in before you disable root" step from every manual guide, except it can't be skipped. The admin has no password, so if that login works, it used a key.

The recovery is best-effort, and it's worth being precise about its limits. The rescue deletes the drop-in only if this run changed it, so it doesn't restore whatever an earlier version contained. And it needs a working connection: Ansible's `rescue` doesn't handle unreachable-host errors, and even when it does run, its tasks still have to reach the server. If a bad reload stopped sshd accepting logins altogether, nothing in the playbook can fix that. So I still keep a root session or the provider's console open on a first run. The checks catch the realistic mistakes; the console covers the rest.

## The firewall that banned its own operator

Every tutorial installs fail2ban and then configures it. On a fresh server, that order can ban the machine running the playbook, partway through the run.

apt starts fail2ban the moment it's installed, and fail2ban's first job is to scan the **existing** `auth.log`. A first run against a server that only accepts a root password leaves a few failed key attempts from your address before the password gets through. That's enough for a ban. fail2ban's `REJECT` rule also sits above ufw's rule that accepts established connections, so the ban doesn't just block new logins: it kills the session Ansible is in the middle of using. The run dies with a privilege escalation timeout that points nowhere near the firewall.

The fix is about order. Write the ignore list before the package exists:

```yaml
- name: Find the address Ansible connects from
  ansible.builtin.shell: echo "$SSH_CONNECTION"
  become: false
  changed_when: false
  register: firewall_ssh_connection

- name: Exempt the deploy address from fail2ban
  ansible.builtin.template:
    src: ignoreip.local.j2
    dest: /etc/fail2ban/jail.d/00-ignoreip.local

- name: Install fail2ban
  ansible.builtin.apt:
    name: fail2ban
```

`$SSH_CONNECTION` gives your address as the server sees it, which is the address fail2ban would ban, so it works behind NAT where your laptop's own idea of its IP would be wrong.

The same role takes the SSH port from `sshd -T` instead of Ansible's inventory. If the port was changed years ago, or a NAT maps 2222 to 22, the firewall still opens the port sshd is actually listening on.

## No more `curl | sh`

Docker and Dokku both document a one-line installer that downloads a script and runs it as root. I replaced both with packages from their signed apt repositories, with Dokku pinned to an exact version (the Docker packages are installed by name, unpinned):

{% raw %}
```yaml
- name: Add Dokku's apt repository
  ansible.builtin.deb822_repository:
    name: dokku
    uris: https://packagecloud.io/dokku/dokku/ubuntu/
    suites: "{{ ansible_facts.distribution_release }}"
    components: main
    signed_by: https://packagecloud.io/dokku/dokku/gpgkey

- name: Install Dokku
  ansible.builtin.apt:
    name: dokku={{ dokku_version }}
```
{% endraw %}

A second run is then a no-op instead of a reinstall, and upgrading Dokku is a one-line version bump. Packages still contain executables and can run maintainer scripts as root when they install, so this isn't "nothing downloaded runs". The gain is that everything arrives through apt from repositories whose signatures are checked, instead of a script piped straight from the network into a root shell. Docker goes first, so Dokku's package depends on `docker-ce` rather than pulling in Ubuntu's older `docker.io`. A debconf answer (`dokku/skip_key_file`) stops the package importing root's SSH key, because the playbook registers the admin's keys itself.

Replacing the scripts exposed one thing they had been doing quietly. Dokku's installer writes `live-restore` into `/etc/docker/daemon.json`. The obvious way to add container log rotation is to template that whole file, which would silently delete Dokku's setting. The playbook reads the file, merges in only the log options and compares the **parsed** JSON. Comparing text doesn't work, because Dokku writes the file unsorted, and every run would "change" it and restart Docker.

## Teaching an imperative CLI to be idempotent

Dokku's CLI does things; it doesn't declare state. Wrapping it in Ansible means the same pattern everywhere: read what's there, act only on the difference. Three commands needed more than that.

**SSH keys.** `dokku ssh-keys:add` fails if the name already exists, so names based on list position break the day someone reorders the list. Each key is named after the admin user and a hash of its own key material, `<admin_user>-<first 8 of sha1>`, and keys Dokku already has are skipped.

**Plugins.** `dokku plugin:install <url>` with no tag clones whatever the default branch holds that day. Two servers built a week apart could run different Postgres plugin code. Each plugin is pinned to a release tag, and the playbook compares that tag with what `dokku plugin:list --format json` reports:

```yaml
dokku_plugins:
  - name: postgres
    url: https://github.com/dokku/dokku-postgres.git
    version: "1.48.0"
  - name: redis
    url: https://github.com/dokku/dokku-redis.git
    version: "1.43.0"
  - name: letsencrypt
    url: https://github.com/dokku/dokku-letsencrypt.git
    version: "0.25.2"
```

This only works because Dokku's own plugins keep the version in `plugin.toml` equal to the release tag. Pin a branch name instead and it never matches, so the plugin "updates" on every single run.

**Let's Encrypt.** The plugin quietly refuses every certificate until a global email is set, which is easy to miss when that's a manual step in a README. Automating it hit one more quirk: asking for just that one value with `letsencrypt:report` exits non-zero when it's empty, so the playbook reads the full `--global --format json` report instead.

## What it doesn't protect you from

**Docker bypasses ufw.** Ports published with `docker run -p` or a Compose `ports:` entry go through Docker's own iptables rules. They're reachable from the internet even though `ufw status` doesn't list them. Dokku apps are fine, because nginx on 80 and 443 proxies to internal addresses. A database you expose "just for a minute" is not. Bind it to `127.0.0.1` and use an SSH tunnel.

**Backups.** There are none. Off-server backups with a tested restore are the next thing I'd add before putting data I can't lose on this host.

## Running it yourself

```bash
git clone https://github.com/thepsalmist/ansible-server-bootstrap.git && cd ansible-server-bootstrap
uv tool install ansible-core
ansible-galaxy collection install -r requirements.yml
cp inventory.ini.example inventory.ini              # your server's address
cp group_vars/all.yml.example group_vars/all.yml    # admin_user, admin_ssh_keys

ansible-playbook bootstrap.yml    # first run, as root (keep a console open)
ansible-playbook site.yml         # every run after that
```

The same playbook also runs Prometheus, Loki and Grafana on the host, and building dashboards for a server that deploys by renaming containers had its own surprises. That's the [next post](/blog/dokku-monitoring-prometheus-loki-grafana/).
