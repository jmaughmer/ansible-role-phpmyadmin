# ansible-role-phpmyadmin

Ansible role that installs and hardens phpMyAdmin behind Apache on RHEL 8, 9 and 10 (and compatible: AlmaLinux, Rocky, Oracle Linux).

Features:

- phpMyAdmin from EPEL, from Red Hat Satellite or Katello repositories, from repository definitions you supply, or from repositories already on the host.
- A local MariaDB reached through its socket by default, or remote servers over TLS with certificate verification.
- Hardened settings: cookie authentication with a 15-minute idle logout, no root login, no empty passwords, no arbitrary servers, no version check, server or PHP info, or error reports, and internal directories never served.
- Access only over HTTPS and only from listed networks.
- SELinux labels for phpMyAdmin's writable directories and the database boolean when needed, plus fapolicyd trust for the edited `config.inc.php` and a scoped rule for compiled templates.
- Molecule tests with podman on EL8, EL9 and EL10, together with the MariaDB and Apache roles, including a real login.

## Requirements

- [ansible-role-apache](https://github.com/jmaughmer/ansible-role-apache) runs first in the same play with `apache_php_enabled: true`, which provides httpd, mod_ssl, TLS and PHP-FPM from AppStream. The role checks for them.
- A database server, typically from [ansible-role-mariadb](https://github.com/jmaughmer/ansible-role-mariadb).
- Ansible core 2.15 or newer (2.16 or older for RHEL 8 targets), and the collections in `requirements.yml`.

## Quick start

```yaml
- hosts: db_servers
  become: true
  roles:
    - role: ansible-role-mariadb
    - role: ansible-role-apache
      vars:
        apache_php_enabled: true
        apache_php_packages: [php-mysqlnd]
    - role: ansible-role-phpmyadmin
      vars:
        phpmyadmin_allowed_networks: [10.0.50.0/24]
```

phpMyAdmin is then at `https://<host>/phpmyadmin/`. Log in with a MariaDB account. Root cannot log in, since `root@localhost` uses `unix_socket` authentication and phpMyAdmin is not allowed to use it anyway.

## Package repository

phpMyAdmin is not in AppStream on any EL release, so `phpmyadmin_repo_source` picks where it comes from.

| Source | What the role does |
|---|---|
| `epel` (default) | On RHEL: imports the EPEL key and installs `epel-release` from the Fedora project, then enables CodeReady Builder with subscription-manager. On AlmaLinux/Rocky: installs `epel-release` from extras and enables CRB (PowerTools on EL8). Oracle Linux uses `oracle-epel-release-elN`. Turn off CRB with `phpmyadmin_epel_enable_crb: false`. |
| `satellite` | Checks the host is registered, then enables `phpmyadmin_satellite_repos` by label with subscription-manager. Works for Red Hat Satellite and Katello (AlmaLinux and Rocky register to Katello with subscription-manager too). |
| `custom` | Writes `.repo` files from `phpmyadmin_custom_repos` (any `yum_repository` arguments), for example a Katello/Pulp published URL or an internal mirror. |
| `none` | Leaves repositories alone, for hosts where an activation key or another role already set them up. |

### Satellite / Katello

Sync EPEL into Satellite/Katello as a custom product (with CodeReady Builder for RHEL), add it to the content view the host uses, then list the repository labels:

```yaml
phpmyadmin_repo_source: satellite
phpmyadmin_satellite_repos:
  - MyOrg_EPEL_EPEL_9_x86_64                     # subscription-manager repos --list
  - codeready-builder-for-rhel-9-x86_64-rpms     # RHEL only
```

Or point at the published content directly:

```yaml
phpmyadmin_repo_source: custom
phpmyadmin_custom_repos:
  - name: katello-epel
    description: EPEL 9 from Katello
    baseurl: https://katello.example.com/pulp/content/MyOrg/Library/custom/EPEL/EPEL_9_x86_64/
    gpgkey: https://katello.example.com/pulp/content/MyOrg/Library/custom/EPEL/RPM-GPG-KEY-EPEL-9
    sslcacert: /etc/rhsm/ca/katello-server-ca.pem
```

## Database servers

```yaml
phpmyadmin_servers:
  # Local MariaDB over its socket (the default). No TLS or SELinux boolean needed.
  - verbose: local
    host: localhost
    socket: /var/lib/mysql/mysql.sock
  # Remote server over TLS, verified against the CA below.
  - verbose: db02
    host: db02.corp.example.com
    port: 3306
    ssl: true
    ssl_ca: /etc/phpMyAdmin/db-ca.crt
    ssl_verify: true
    hide_db: "^(information_schema|performance_schema|mysql|sys)$"

phpmyadmin_db_ca_src: files/db02-ca.crt   # written to /etc/phpMyAdmin/db-ca.crt
```

ansible-role-mariadb enforces TLS on TCP and gives remote accounts `REQUIRE SSL`, so remote servers need `ssl: true`. When any server uses TCP, the role turns on the `httpd_can_network_connect_db` SELinux boolean.

## Hardening

| Setting | Variable | Default |
|---|---|---|
| Idle logout | `phpmyadmin_login_cookie_validity` | 900 s |
| Root login | `phpmyadmin_allow_root` | false |
| Empty passwords | `phpmyadmin_allow_no_password` | false |
| Version check, server info, PHP info | `phpmyadmin_version_check`, `phpmyadmin_show_*` | false |
| Error reports to phpMyAdmin.net | `phpmyadmin_send_error_reports` | never |
| Allowed client networks | `phpmyadmin_allowed_networks` | 127.0.0.1, ::1 |
| HTTPS only | `phpmyadmin_require_https` | true |

The role replaces the package's `/etc/httpd/conf.d/phpMyAdmin.conf` (a `noreplace` file, so updates keep it). It denies `setup/`, `libraries/`, `templates/`, `vendor/`, `sql/`, `src/` and `tmp/`. `config.inc.php` is `root:apache 0640`. The cookie encryption secret is generated on the host (`/etc/phpMyAdmin/.blowfish_secret`) unless you set `phpmyadmin_blowfish_secret` (32 characters).

Apache-level hardening (TLS, headers, cookie flags, PHP settings) comes from ansible-role-apache.

## SELinux

The temp, upload and save directories under `/var/lib/phpMyAdmin` are labelled `httpd_sys_rw_content_t`. `httpd_can_network_connect_db` is set for TCP servers, and `phpmyadmin_selinux_booleans` adds others.

## fapolicyd

The phpMyAdmin code comes from an RPM and is trusted, but two things need help:

- `config.inc.php` is edited, so its hash no longer matches the RPM database. The role adds it to `/etc/fapolicyd/trust.d/phpmyadmin`.
- phpMyAdmin compiles its Twig templates to PHP files in `/var/lib/phpMyAdmin/temp` at run time. fapolicyd's language rules would stop php-fpm from opening them, so `/etc/fapolicyd/rules.d/65-phpmyadmin.rules` allows exactly that: `php-fpm` opening PHP files in that directory. Set `phpmyadmin_fapolicyd_allow_temp: false` to leave it out; phpMyAdmin then needs `TempDir` disabled via `phpmyadmin_extra_config`.

## FIPS

Nothing in phpMyAdmin's setup changes under FIPS: TLS to the browser is Apache's and follows the system crypto policy, and TLS to remote databases goes through PHP's OpenSSL, which does too. phpMyAdmin encrypts its login cookie with libsodium, which is not a FIPS-validated module. The cookie only lives in the user's browser for the session, but note it if your accreditation counts it.

## Variables

See [`defaults/main.yml`](defaults/main.yml) for every variable with comments.

Tags: `phpmyadmin_repo`, `phpmyadmin_install`, `phpmyadmin_config`, `phpmyadmin_security`, `phpmyadmin_selinux`, `phpmyadmin_fapolicyd`.

## Testing

Tests run locally with podman; there is no CI. On the AlmaLinux 10 WSL distribution (or any EL10 host):

```bash
scripts/test.sh                                              # lint, default (EL9 + EL10), el8
PHPMYADMIN_DEPS_REF=some-branch scripts/test.sh default      # test against role branches
```

The scenarios also run ansible-role-mariadb and ansible-role-apache. If those repositories are checked out next to this one, they are used as they are. Otherwise the script clones them into `.cache/test/roles` at `PHPMYADMIN_DEPS_REF` (default `main`). The repository directory must be named `ansible-role-phpmyadmin`.

## License

MIT
