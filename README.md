# AdGuard Home, Caddy, Docker Compose, and Ansible on a Raspberry Pi

An example config for turning a Raspberry Pi into your network's DNS server: AdGuard Home blocks ads, trackers, and malware for every device on the network, and sends everything else upstream over DNS-over-HTTPS so your ISP can't read or rewrite your lookups. Caddy puts AdGuard's admin UI behind a real HTTPS certificate. Ansible sets the whole thing up from a freshly flashed SD card.

It's how I over-engineer my own home network. For the backstory, see [How I re-over-engineered my home network for privacy and security](https://ben.balter.com/2021/09/01/how-i-re-over-engineered-my-home-network/) (and the [original over-engineering](https://ben.balter.com/2020/12/04/over-engineered-home-network-for-privacy-and-security/)).

## What's in the box

- **[AdGuard Home](https://github.com/AdguardTeam/AdGuardHome)** answers DNS on port 53. It ships with Cloudflare's malware-blocking resolver as its DoH upstream, DNSSEC validation, an optimistic cache, and the [HaGeZi](https://github.com/hagezi/dns-blocklists) blocklists. It only answers clients on private networks, so a forwarded port can't turn it into an open resolver.
- **[Caddy](https://caddyserver.com)** serves the admin UI at a hostname you choose, with a Let's Encrypt certificate it gets through a Cloudflare DNS-01 challenge. That means you never have to open ports 80 or 443 to the internet. Requests from outside private IP ranges get a 403 anyway.
- **An Ansible playbook** that hardens the Pi (SSH keys only, unattended upgrades, fail2ban, ufw), installs Docker from Docker's own apt repository, clones your config repository, writes the secrets, and starts everything. It fails the run if AdGuard doesn't come up healthy, since a broken DNS server breaks the whole network.

## What you'll need

- A Raspberry Pi running the current 64-bit [Raspberry Pi OS Lite](https://www.raspberrypi.com/software/), with SSH key login set up in Raspberry Pi Imager and a fixed IP (a DHCP reservation on your router).
- A domain in [Cloudflare](https://www.cloudflare.com), and an API token with `Zone:Read` and `DNS:Edit` permissions on it.
- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/) on your computer.
- A private GitHub repository to hold your copy of this config, plus a fine-grained token that can add a deploy key to it.

## Setup

1. Create a private repository from this one, and clone it.
2. Install the roles and collections: `ansible-galaxy install -r requirements.yml && ansible-galaxy collection install -r requirements.yml`.
3. Copy `hosts.example.yml` to `hosts.yml`, and set the Pi's address, your SSH user, `github_user` (whose public SSH keys get access), and `config_repo`.
4. Copy `playbook.env.example` to `playbook.env`, and fill in the three secrets. Generate the AdGuard password hash with `htpasswd -nbB -C 10 admin 'your-password' | cut -d: -f2`.
5. Replace `dns.example.com` and `you@example.com` in the `Caddyfile`, and add a rewrite for that hostname in `adguard-conf/AdGuardHome.example.yaml` (or later, in the UI). Commit and push.
6. Run the playbook. With 1Password: `op run --env-file=playbook.env -- ansible-playbook playbook.yml --inventory hosts.yml`. With anything else, export the variables first.
7. Point your router's DHCP DNS setting at the Pi.

AdGuard saves settings you change in its UI to `adguard-conf/AdGuardHome.yaml` on the Pi. The playbook only creates that file the first time, so rerunning it won't undo your changes. That includes the admin password: after the first run, change it in the UI, not in `playbook.env`.

AdGuard's safe browsing check, which is on by default, sends hash prefixes of the domains you visit to AdGuard's servers. Everything else goes to your upstream. Turn it off in the config if you'd rather send every lookup to Cloudflare.

Give the Pi itself a public DNS server (such as `1.1.1.2`) in its DHCP reservation or network settings, rather than pointing it at its own AdGuard. Otherwise, if AdGuard goes down, the Pi can't resolve Docker Hub to pull the fix.

## Coming from the Pi-hole version?

This repository used to run Pi-hole with cloudflared as a DNS-over-HTTPS proxy. cloudflared [removed its DNS proxy](https://github.com/cloudflare/cloudflared/blob/master/CHANGES.md) in 2026.2.0, and Pi-hole v6 ignores the environment variables the old `docker-compose.yml` set, so that setup no longer works. AdGuard Home does DNS-over-HTTPS itself, which means one container instead of two.

To move over: export your Pi-hole allowlist and custom blocklists, then run `docker compose down` in the old `pi-hole` directory on the Pi, so Pi-hole lets go of ports 53, 80, and 443. Run the new playbook, and add them under **Filters** in AdGuard. The old setup marked the `caddy_data` volume `external`, so `docker volume rm` it (or leave it; nothing uses it now). Caddy will request a fresh certificate.
