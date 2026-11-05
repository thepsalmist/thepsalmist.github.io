---
layout: post
title: "How to Set Up Dokku on Ubuntu 24.04 with Ansible (Hardened, Re-runnable Bootstrap)"
ogTitle: "From Fresh VPS to Your Own Heroku: A Re-runnable Dokku Bootstrap"
slug: dokku-ubuntu-ansible-bootstrap
permalink: /blog/dokku-ubuntu-ansible-bootstrap/
date: 2026-09-27 09:00:00 +0300
last_modified_at: 2026-11-05
description: "Turn a fresh Ubuntu 22.04/24.04 VPS into a hardened Dokku host with one re-runnable Ansible playbook, and the hardening steps that reported success without taking effect."
keywords:
  - dokku ansible
  - install dokku ubuntu 24.04
  - dokku vps setup
  - self-hosted heroku alternative
  - ansible ubuntu server hardening
  - passwordauthentication no not working
  - fail2ban banned my ip ansible
  - docker bypasses ufw
  - dokku plugins pin version
tags: [ansible, dokku, ubuntu, ssh, fail2ban, docker, infrastructure]
---

`PasswordAuthentication no` is probably the most copied line of server hardening advice on the internet. On a lot of Ubuntu cloud images, it does nothing.

I found that out while turning my new-server checklist into an Ansible playbook, [ansible-server-bootstrap](https://github.com/thepsalmist/ansible-server-bootstrap). It takes a fresh Ubuntu 22.04 or 24.04 VPS to a hardened [Dokku](https://dokku.com/) host with Heroku-style `git push` deploys, and it's built to be re-run. Automating the checklist meant checking it, and a few steps I had always trusted turned out not to do what I thought.

> **TL;DR**
> - A cloud-init drop-in can keep SSH password login enabled after you disable it in `sshd_config`. Write your settings to `sshd_config.d/00-hardening.conf` and confirm them with `sshd -T`.
> - Installing fail2ban before exempting your own address can ban the machine running the playbook, mid-run.
> - Docker and Dokku come from their signed apt repositories, not `curl | sh`, and Dokku and its plugins are pinned to exact versions.
> - Still not covered: ports published by Docker bypass ufw, and there are no backups yet.

## A cloud-init drop-in keeps password login on

Ubuntu's `/etc/ssh/sshd_config` starts with a line most people scroll past:

```
Include /etc/ssh/sshd_config.d/*.conf
```

Those drop-ins are read before the rest of the main file, and for most settings [sshd uses the first value it obtains](https://man.openbsd.org/sshd_config). On many VPS images, cloud-init has already written a drop-in that allows passwords:

```
sshd_config.d/50-cloud-init.conf   PasswordAuthentication yes   <- read first, wins
sshd_config                        PasswordAuthentication no    <- ignored
```

`sshd -t` passes, the reload succeeds, and password login keeps working. The only way to see it is to ask sshd what it will actually do:

```bash
sudo sshd -T | grep -Ei '^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin) '
```

So the playbook leaves the main file alone and writes `/etc/ssh/sshd_config.d/00-hardening.conf`. Drop-ins load in name order, so `00-` is read before cloud-init's `50-`.

That still isn't proof. A `Match` block or a provider's own `00-aaa.conf` could override it, so the role checks the effective config before trusting its own file, and removes the file if a check fails:

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

Three details carry the weight:

- `-C` evaluates the config for one connection, a root login from localhost, so `Match` blocks that apply to it are included. [Without `-C`, `sshd -T` doesn't apply `Match` rules](https://man.openbsd.org/sshd).
- The reload is a task, not a handler, because handlers run at the end of the play, after the check that's meant to test them.
- `reset_connection` followed by `wait_for_connection` proves a brand new login works, not just the connection Ansible already had open.

Before any of this runs, `bootstrap.yml` logs in as root, creates an admin user with no password, and hands over to `site.yml`, which connects as that admin. It's the "open a second terminal before you disable root" step, except it can't be skipped.

The rollback is best-effort. It only deletes a drop-in this run changed, and it needs a working connection: [Ansible doesn't run `rescue` for unreachable hosts](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_blocks.html). On a first run I still keep the provider's console open.

## Installing fail2ban banned the machine running the playbook

Most tutorials install [fail2ban](https://github.com/fail2ban/fail2ban), then configure it. On a fresh server, that order can ban the machine running the playbook.

apt starts fail2ban as soon as it's installed, and it starts reading the recent SSH log history straight away. A first run against a server that only accepts a root password leaves a few failed key attempts from your address, which is enough for a ban. Because fail2ban's `REJECT` rule sits above ufw's rule for established connections, the ban also kills the session Ansible is using. The run dies with a privilege-escalation timeout that points nowhere near the firewall.

The fix is ordering: write the ignore list before the package exists.

```yaml
- name: Find the address Ansible connects from
  ansible.builtin.shell: echo "$SSH_CONNECTION"
  become: false
  changed_when: false
  register: firewall_ssh_connection

- name: Create the fail2ban jail directory
  ansible.builtin.file:
    path: /etc/fail2ban/jail.d
    state: directory

- name: Exempt the deploy address from fail2ban
  ansible.builtin.template:
    src: ignoreip.local.j2
    dest: /etc/fail2ban/jail.d/00-ignoreip.local

- name: Install fail2ban
  ansible.builtin.apt:
    name: fail2ban
```

`$SSH_CONNECTION` is your address as the server sees it, which is the address fail2ban would ban, so it's right even behind NAT. The same role reads the SSH port from `sshd -T` rather than the inventory, so ufw opens the port sshd actually listens on.

SSH and the firewall were about proving that hardening actually took effect. The rest of the playbook had a different job: installing Docker and Dokku so that a second run leaves a working server alone.

## Installing Docker and Dokku from signed apt repositories

[Docker](https://docs.docker.com/engine/install/ubuntu/) and [Dokku](https://dokku.com/docs/getting-started/installation/) both document a one-line installer that runs a downloaded script as root. The playbook installs both from their signed apt repositories instead, with Dokku pinned to an exact version:

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

The package is also held, so the base role's apt upgrade can't move it. A second run is a no-op, and upgrading Dokku is a one-line version bump. Packages can still run maintainer scripts as root, so this isn't "nothing downloaded runs". The gain is that everything arrives through apt with its signatures checked. Docker installs first, so apt satisfies Dokku's Docker dependency with `docker-ce` rather than pulling in Ubuntu's older `docker.io`.

Replacing the scripts exposed one thing Dokku does quietly: it writes `live-restore` into `/etc/docker/daemon.json`. Templating that file to add log rotation would delete the setting, so the playbook merges in only the log options and compares the parsed JSON. A text comparison would never match, because Dokku writes the file unsorted, and Docker would restart on every run.

## Making Dokku's CLI safe to re-run

Dokku's CLI does things; it doesn't declare state. So every task reads what's there and acts only on the difference. Three commands needed more than that:

- **SSH keys.** `ssh-keys:add` fails if the name already exists, so each key is named after a hash of its own material (`<admin_user>-<first 8 of sha1>`), not its position in a list.
- **Plugins.** `plugin:install <url>` with no tag clones whatever the default branch holds that day. Each plugin is pinned to a release tag and compared with `dokku plugin:list --format json`.
- **Let's Encrypt.** [The plugin](https://github.com/dokku/dokku-letsencrypt) won't issue certificates until a global email is set. Querying just that value exits non-zero when it's empty, so the playbook reads the full `letsencrypt:report --global --format json` instead.

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

Pinning works because Dokku's official plugins keep the version in `plugin.toml` equal to the release tag. Pin a branch name instead and it never matches, so the plugin reinstalls on every run.

## What it doesn't protect you from

**Docker bypasses ufw.** Ports published with `docker run -p` or a Compose `ports:` entry [go through Docker's own firewall rules](https://docs.docker.com/engine/network/packet-filtering-firewalls/), so they're reachable from the internet even though `ufw status` doesn't list them. Dokku apps are fine, because nginx on 80 and 443 proxies to internal addresses. A database exposed "just for a minute" isn't: bind it to `127.0.0.1` and use an SSH tunnel.

**Backups.** There are none yet. Off-server backups with a tested restore come before I put data I can't lose on this host.

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

If there's one habit to take from this: a command that exits 0 hasn't proved anything. Have the playbook check the state the command was meant to change, the way `sshd -T` checks what sshd will actually do.

The same playbook also runs Prometheus, Loki and Grafana on the host, and building dashboards for a server that deploys by renaming containers had its own surprises. That's the [next post](/blog/dokku-monitoring-prometheus-loki-grafana/).
