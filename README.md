# ansible-wordpress

Deploy WordPress on **two Ubuntu servers** with Ansible: a web server (Apache, PHP, WordPress) and a separate database server (MySQL).

```
                       web group                          db group
 Browser ──HTTP :80──▶ node03                             db01
                       Apache + PHP + WordPress ──TCP :3306──▶ MySQL
                       roles: apache, php, wordpress      role: mysql
```

## Project layout

```
├── site.yaml                  # db play first, then web play
├── ansible.cfg                # inventory path, roles path, sudo by default
├── requirements.yaml          # collections to install (community.mysql)
├── inventory/
│   ├── hosts.ini              # [web] and [db] servers
│   └── group_vars/all/
│       ├── vars.yaml          # shared variables (db name, db user, paths)
│       └── vault.yaml         # encrypted db password, created by you (git-ignored)
├── roles/
│   ├── mysql/                 # install, listen on the network, create db + user
│   ├── apache/                # install, WordPress virtual host, rewrite module
│   ├── php/                   # PHP + extensions (list in files/requires)
│   └── wordpress/             # download, wp-config.php pointing at the db server
└── vagrant/Vagrantfile        # optional: builds the two VMs
```

## Requirements

- Control machine: `ansible-core` (developed with 2.20) and `ansible-galaxy`.
- Two Ubuntu 22.04 servers reachable over SSH with key login, and a user with sudo (`vagrant` by default, set `ansible_user` in `hosts.ini`).
- The servers need internet access (apt packages, wordpress.org).

## Quick start

```bash
# 1. Install the MySQL collection
ansible-galaxy collection install -r requirements.yaml

# 2. Put your servers' IPs in inventory/hosts.ini, then test the connection
ansible all -m ping

# 3. Create the encrypted password file. In the editor add:  wp_db_password: "your-password"
ansible-vault create inventory/group_vars/all/vault.yaml

# 4. Deploy
ansible-playbook site.yaml --ask-vault-pass
```

Then open `http://<web-server-ip>` and finish the WordPress installer (site title, admin user).

## How the web server and the database server connect

On a single server WordPress talks to MySQL on the same machine. With two servers three things change, and each is one task in this project:

| One server | Two servers | Task / file |
|---|---|---|
| MySQL accepts local connections only (`bind-address = 127.0.0.1`) | MySQL listens on the network (`bind-address = 0.0.0.0`) | `roles/mysql/tasks/configure.yaml` |
| MySQL user is `wordpress@localhost` | MySQL user is `wordpress@<web-ip>` | `roles/mysql/tasks/database.yaml` |
| `DB_HOST` is `localhost` | `DB_HOST` is the db server's IP | `roles/wordpress/templates/wp-config.php.j2` |

Before writing `wp-config.php`, the `wordpress` role checks from the web server that port 3306 on the db server is reachable. At the end of `site.yaml` a request to the site proves the full chain works: WordPress answers HTTP 500 when it cannot reach its database, so HTTP 200 means the connection is good.

## Running it again

The roles are idempotent: a second run should report `changed=0`. Useful options:

```bash
ansible-playbook site.yaml --ask-vault-pass --limit db     # only the database server
ansible-playbook site.yaml --ask-vault-pass --diff         # show file changes
ansible-inventory --graph                                  # check the groups
ansible-doc community.mysql.mysql_user                     # read a module's options
```

`--check` (dry run) works for the first tasks but fails on the service tasks of a fresh server: the packages are not installed in check mode, so the services do not exist yet. Run for real instead.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `wp_db_password is not set` | Create the vault file (step 3) and pass `--ask-vault-pass` |
| `Timeout when waiting for ...:3306` | MySQL not listening on the network or a firewall blocks it. On db: `sudo ss -tlnp \| grep 3306` must show `0.0.0.0:3306` |
| `Access denied for user 'wordpress'@'...'` | The web server's source IP differs from `mysql_wp_user_host`. On db: `SELECT user, host FROM mysql.user;`. On web, `mysql -h <db-ip> -u wordpress -p` then `SELECT USER();` shows the address MySQL sees |
| Browser: "Error establishing a database connection" | Check `DB_*` in `wp-config.php`, and `php -m \| grep mysqli` on the web server |
| Changed the vault password but MySQL still uses the old one | Run once with `-e mysql_wp_update_password=always` |

## Security notes

- This is a lab setup. `host_key_checking = False` in `ansible.cfg` skips SSH host key prompts; remove it for real servers.
- MySQL listens on all interfaces, but only `wordpress@<web-ip>` may log in. Add a firewall rule that allows port 3306 from the web server only.
- The site runs on plain HTTP. Add TLS before using it for anything real.
- `vault.yaml` and the random WordPress salts (`credentials/`) are git-ignored, so the repository contains no secrets.

## Optional: build the VMs with Vagrant

```bash
cd vagrant && vagrant up
vagrant ssh node03 -c "hostname -I"    # put the IPs in inventory/hosts.ini
vagrant ssh db01 -c "hostname -I"
```

Your public key (`~/.ssh/id_ed25519.pub`) is copied into both VMs, so Ansible can log in right away.
