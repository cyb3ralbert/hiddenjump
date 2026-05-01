# dotfiles

Personal config to bootstrap a new Arch machine with reliable remote access
to a home laptop via a reverse SSH tunnel through a public VPS.

All host-specific values (VPS IP, usernames, port) live in `.env` and are
substituted into the templates by `bin/install`. The repo itself is safe
to publish — no IPs, no usernames, no keys.

## Architecture

```
[ Android / laptop / anywhere ]  --ssh-->  [ VPS:22 root ]  <--reverse tunnel--  [ home laptop ]
                                                  |
                                       127.0.0.1:${TUNNEL_PORT}  --forwards-->  home laptop sshd
```

The home laptop holds open a persistent reverse tunnel to the VPS as a
restricted, no-shell user `tunnel`. Anyone with a key authorized as
`root@vps` can then `ssh home` (uses ProxyJump from `ssh/config`).

## Layout

```
dotfiles/
├── bin/
│   └── install                    # render templates from .env, install
├── ssh/
│   └── config                     # template -> ~/.ssh/config
├── systemd/
│   └── reverse-tunnel.service     # template -> /etc/systemd/system/  (home laptop only)
├── .env.example                   # copy to .env and fill in
├── .gitignore
└── README.md
```

## Prerequisites

- `gettext` (provides `envsubst`): `pacman -S gettext` / `apt install gettext-base` / `pkg install gettext` (Termux)
- `openssh`, plus `autossh` on the home laptop

## Configure

```bash
cp .env.example .env
$EDITOR .env        # set VPS_HOST, VPS_USER, HOME_USER, TUNNEL_PORT
```

## Bootstrap a client (laptop, desktop, Termux)

```bash
bin/install ssh

# Generate an interactive key and authorize it on the VPS
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N '' -C "$(whoami)@$(hostname)"
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@$(grep ^VPS_HOST .env | cut -d= -f2)

ssh vps  hostname
ssh home hostname        # only works if the home laptop's tunnel is up
```

## Bootstrap the home laptop (the side that holds the tunnel open)

```bash
sudo pacman -S --needed autossh

# Tunnel key — separate from any interactive key
ssh-keygen -t ed25519 -f ~/.ssh/id_tunnel -N '' -C "tunnel@$(hostname)"

# On the VPS (one-time, as root):
#   useradd -m -s /usr/sbin/nologin tunnel
#   install -d -m 700 -o tunnel -g tunnel /home/tunnel/.ssh
#   cat > /home/tunnel/.ssh/authorized_keys <<'KEY'
#   restrict,port-forwarding,permitlisten="2222" ssh-ed25519 AAAA... tunnel@<host>
#   KEY
#   chown tunnel:tunnel /home/tunnel/.ssh/authorized_keys
#   chmod 600 /home/tunnel/.ssh/authorized_keys
#
# `restrict,port-forwarding,permitlisten=...` denies shell, PTY, agent
# forwarding, X11, and any remote forward other than the tunnel port.

bin/install tunnel
sudo systemctl enable --now reverse-tunnel.service
systemctl status reverse-tunnel.service
```

## Android (Termux)

```bash
pkg install openssh gettext git
git clone <this-repo> ~/dotfiles && cd ~/dotfiles
cp .env.example .env && $EDITOR .env
bin/install ssh

ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ''
# Copy ~/.ssh/id_ed25519.pub into root@vps:~/.ssh/authorized_keys
# AND into ${HOME_USER}@home:~/.ssh/authorized_keys

ssh home
```

For terminal apps that don't speak `~/.ssh/config` (e.g. Termius), point
them at `${VPS_HOST}:22 root` and run `ssh -p ${TUNNEL_PORT} ${HOME_USER}@localhost`
from inside.

## VPS hardening

Already applied:

- fail2ban with the `sshd` jail (`bantime=1h findtime=10m maxretry=5`)
- Tunnel user has `/usr/sbin/nologin` and is restricted via authorized_keys
  options to only listen on the tunnel port

Optional:

- `PasswordAuthentication no` in `/etc/ssh/sshd_config`. Always verify key
  login from a **second** SSH session before `systemctl reload sshd` (never
  `restart` an in-use sshd).

## Verify after reboot

On the home laptop:

```bash
systemctl status reverse-tunnel.service
journalctl -u reverse-tunnel.service -n 20 --no-pager
```

From the VPS:

```bash
ss -tnlp | grep ":${TUNNEL_PORT} "
ssh -p ${TUNNEL_PORT} ${HOME_USER}@localhost
```

From any client:

```bash
ssh home
```
