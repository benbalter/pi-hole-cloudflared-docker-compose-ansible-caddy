# Example Docker Compose and Ansible configuration for running Pi-Hole, Cloudflared, and Caddy

Example configuration for using Pi-Hole, Cloudflared, Docker Compose, Ansible, and Caddy to over-engineer your home network for privacy and security.

## Details

See [How I over-engineered my home network for privacy and security](https://ben.balter.com/2020/12/04/over-engineered-home-network-for-privacy-and-security/) and [How I re-over-engineered my home network for privacy and security](https://ben.balter.com/2021/09/01/how-i-re-over-engineered-my-home-network/).

## Usage

1. Download the [Raspberry Pi Imager](https://www.raspberrypi.org/software/) and flash the latest version of Raspberry Pi OS *Lite*.
2. Run `ansible-playbook playbook.yml --inventory hosts.yml`
3. Sit back and wait until you have a fully configured PiHole running in about 5-10 minutes
