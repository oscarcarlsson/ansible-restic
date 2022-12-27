# Ansible role for Restic

This role will setup [Restic](https://restic.net/) backups on a Debian/Ubuntu machine using a systemd service and timer.

It supports all restic backends but will help you setup SSH keys if you go the SFTP route. It will optionally also download and install restic from GitHub.

## Role Variables

### Restic installation

The role will download and install the restic binary (version `restic_version`) into `restic_path` if the file does not exist, or if it's not of the right version.

If you want to force the installation, overwrite the binary or update restic, you can run ansible with `--extra-vars restic_force_install=true`.

If you are using an architecture different from Linux x86, set variable `restic_arch` to match.  This is used as a suffix for downloading restic from [the Github release page](https://github.com/restic/restic/releases).  Defaults to `linux_amd64`.

### Restic configuration

- `restic_user`: user to run restic as (`root`)
- `restic_user_home`: home directory of the restic_user (`/root`)
- `restic_password`: password used for repository encryption
- `restic_repository_name`: the name of the repository (`restic`)
- `restic_check`: run `restic check` as `ExecStartPre` if true (`false`)
- `restic_default_folders`: a default list of folders that restic will backup (`/etc/`, `/root` and `/var/log`)
- `restic_folders`: the list of folder you want to backup
- `restic_dump_compression_enabled`: enable piping to pigz for database dumps

Each folder has a `path` and an `exclude` property (which defaults to nothing). The `exclude` property is the literal argument passed to restic (exemple: `--exclude .cache --exclude .local`).

`restic_default_folders` and `restic_folders` are combined to form the final list of backuped folders.

- `restic_databases`: a list of databases to dump

Each database has a `name` property which will be the name of the restic snapshot (`{{ database.name }}.sql`). They also have a `dump_command` property which is the command to dump the database to stdout (like `mysqldump dbname`).

- `restic_forget`: run `restic forget` as `ExecStartPost` with `--keep-within {{ restic_forget_keep_within }}` (`true`)
- `restic_forget_keep_within`: period of time to use with `--keep-within` (`30d`)
- `restic_prune`: run `restic prune` as `ExecStartPost` (`true`)

### SSH/SFTP backend configuration

The SSH configuration will be written in `{{ restic_user_home }}/.ssh/config`.

- `restic_ssh_host`: backend name and SSH alias for the backup host
- `restic_ssh_user`: user for SSH connection
- `restic_ssh_hostname`: actual SSH hostname of the backup machine
- `restic_ssh_private_key`: private SSH key used to connect to the backup host
- `restic_ssh_private_key_path`: path of the private key to use (`~/.ssh/backup`)
- `restic_ssh_port`: SSH port to use with the backup machine (`23`)

### S3 backend configuration

- `restic_ssh_enabled`: set to false
- `restic_repository_name`: set to s3 endpoint + bucket, restic syntax (e.g. `s3:https://s3.fr-par.scw.cloud/restic-bucket`)
- `restic_aws_access_key_id`: `AWS_ACCESS_KEY_ID`
- `restic_aws_secret_access_key`: `AWS_SECRET_ACCESS_KEY`

### Generic backend configuration

Useful for using the restic restserver as a backend.

- `restic_ssh_enabled`: set to false
- `restic_repository_name`: set to valid restic syntax.

For example, `rest:https://user:password@example.com/repository` to use HTTP basic authentication with [restic-restserver](https://github.com/restic/rest-server).

### Systemd service and timer

A `restic-backup.service` service will be created with all the parameters defined above. The service is of type `oneshot` and will be triggered periodically with `restic-backup.timer`.

The timer is configurable as follows:

- `restic_systemd_timer_on_calender`: defines the `OnCalendar` directive (`*-*-* 03:00:00`)
- `restic_systemd_timer_randomized_delay_sec`: Delay the timer by a random amount of time between 0 and the specified time value. (`0`)

See the [systemd.timer](https://www.freedesktop.org/software/systemd/man/systemd.timer.html) documentation for more information.

You can see the logs of the backup with `journalctl`. (`journalctl -xefu restic-backup`).

## Example playbook

```yaml
---

- hosts: myhost
  roles: restic
  vars:
    restic_ssh_user: backupuser
    restic_ssh_hostname: storage-server.infra.tld
    restic_folders:
      - {path: "/srv"}
      - {path: "/var/www"}
    restic_databases:
    - {name: website, dump_command: sudo -Hiu postgres pg_dump -Fc website}
    - {name: website2, dump_command: mysqldump website2}
    restic_password: mysuperduperpassword
    restic_ssh_private_key: |-
      -----BEGIN OPENSSH PRIVATE KEY-----
      b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
      QyNTUxOQAAACAocs5g1I4kFQ1HH/YZiVU+zLhRDu4tfzZ9CmFAfKhL2AAAAJi02XEwtNlx
      MAAAAAtzc2gtZWQyNTUxOQAAACAocs5g1I4kFQ1HH/YZiVU+zLhRDu4tfzZ9CmFAfKhL2A
      AAAEADZf2Pv4G74x+iNtuwSV/ItnR3YQJ/KUaNTH19umA/tChyzmDUjiQVDUcf9hmJVT7M
      uFEO7i1/Nn0KYUB8qEvYAAAAE3N0YW5pc2xhc0BtYnAubG9jYWwBAg==
      -----END OPENSSH PRIVATE KEY-----
```

S3 example:

```yaml
---

- hosts: myhost
  roles: restic
  vars:
    restic_ssh_enabled: false
    restic_repository: "s3:https://s3.fr-par.scw.cloud/restic-bucket"
    restic_aws_access_key_id: xxxxx
    restic_aws_secret_access_key: xxxxx
    restic_folders:
      - {path: "/srv"}
      - {path: "/var/www"}
    restic_databases:
    - {name: website, dump_command: sudo -Hiu postgres pg_dump -Fc website}
    - {name: website2, dump_command: mysqldump website2}
    restic_password: mysuperduperpassword
```

Of course, `restic_password` and `restic_ssh_private_key` should be stored using ansible-vault.

## License

MIT

## Author Information

See my other Ansible roles at [angristan/ansible-roles](https://github.com/angristan/ansible-roles).
