# Platform.sh Backup

An Ansible role which automates backing up your sites on Platform.sh to S3 or SFTP.

## Requirements

* The `platform` command must be installed and executable by the user running the role.
* PHP must be installed for the `platform` command to function. 
* The `rclone` command must be installed and executable by the user running the role.
* The `s3cmd` command must be installed and executable by the user running the role.
* The `scp` command must be installed and executable by the user running the role.
* The `ssh` command must be installed and executable by the user running the role.

Please see [Platform.sh's CLI documentation](https://docs.platform.sh/gettingstarted/introduction/template/cli-install.html) on how to install the command.

## Dependencies

The following collections must be installed:

* cloud.common
* amazon.aws
* community.general

## Role Variables

This role requires one dictionary as configuration, `platformsh_backup`:

```yaml
    platformsh_backup:
      cliPath: "/usr/local/bin/platform"
      debug: true
      stopOnFailure: false
      sources: {}
      remotes: {}
      backups: []
```

Where:
* `cliPath` is the full path to the `platform` executable. Optional, defaults to `platform`.
* `debug` is `true` to enable debugging output. Optional, defaults to `false`.
* `stopOnFailure` is `true` to stop the entire role if any one backup fails. Optional, defaults to `false`.
* `sources` is a dictionary of sites and environments. Required.
* `remotes` is a dictionary of remote upload locations. Required.
* `backups` is a list of backups to perform. Required.

### Specifying Sources

In this role, "sources" specify the source from which to download backups. Each must have a unique key which is later used in the `platformsh_backup.backups` list.

```yaml
platformsh_backup:
  sources:
    example.com:
      project: "abcdef1234567"
      cliTokenFile: "/path/to/platform-cli-token.txt"
      keyFile: "/path/to/id_rsa"
      retryCount: 3
      retryDelay: 30
```

Where, in each entry:
* `project` is the Platform.sh project ID. Required.
* `cliTokenFile` is the path to a file containing the Platform.sh CLI token. Optional if `cliToken` is specified.
* `cliToken` contains the Platform.sh CLI token. Ignored if `cliTokenFile` is specified.
* `keyFile` is the path to the SSH private key configured on your Platform account. The public key must be in the same directory with a matching name. Required.
* `retryCount` is the number of time to retry `platform` commands if they fail. Optional, defaults to `3`.
* `retryDelay` is the time in seconds to wait before retrying a failed `platform` command. Optional, defaults to `30`.

### Specifying remotes

In this role "remotes" are upload destinations for backups. This role supports S3 or SFTP as remotes. Each remote must have a unique key which is later used in the `platformsh_backup.backups` list.

```yaml
    - hosts: servers
      vars:
        platformsh_backup:
          remotes:
            example-s3-bucket:
              type: "s3"
              bucket: "my-s3-bucket"
              provider: "AWS"
              accessKeyFile: "/path/to/aws-s3-key.txt"
              secretKeyFile: "/path/to/aws-s3-secret.txt"
              hostBucket: "my-example-bucket.s3.example.com"
              s3Url: "https://my-example-bucket.s3.example.com"
              region: "us-east-1"
              acl: "private"
            sftp.example.com:
              type: "sftp"
              host: "sftp.example.com"
              user: "example_user"
              keyFile: "/config/id_example_sftp"
              pubKeyFile: "/config/id_example_sftp.pub"
```

For `s3` type remotes:
* `bucket` is the name of the S3 bucket. 
* `accessKeyFile` is the path to a file containing the access key. Optional if `accessKey` is specified.
* `accessKey` is the value of the access key necessary to access the bucket. Ignored if `accessKeyFile` is specified.
* `secretKeyFile` is the path to a file containing the secret key. Optional if `secretKey` is specified.
* `provider` is the S3 provider. [See rclone's S3 documenation on `--s3-provider`](https://rclone.org/s3/#s3-provider) for possible values. Optional, defaults to `AWS`.
* `secretKey` is the value of the access key necessary to secret the bucket. Ignored if `secretKeyFile` is specified.
* `hostBucket` is the hostname of the bucket, typically `bucketName.s3EndPointHostname.tld`. Required if not using AWS.
* `s3Url` is the full URL to the S3 bucket, including protocol, typically `https://bucketName.s3EndPointHostname.tld`. Required if not using AWS.
* `region` is the AWS region in which the bucket resides. Required if using AWS S3, may be optional for other providers.
* `endpoint` is the S3 endpoint to use. Optional if using AWS, required for other providers.
* `acl` is the ACL with which to upload the files. [See the rclone's S3 documentation on `--s3-acl`](https://rclone.org/s3/#s3-acl) for possible values. Optional, defaults to `private`.

For `sftp` type remotes:
* `host` is the hostname of the SFTP server. Required.
* `user` is the username necessary to login to the SFTP server. Required.
* `keyFile` is the path to a file containing the SSH private key. Required.
* `pubKeyFile` si the path to a file containing the SSH public key. Required for database backups, ignored for file backups.

### Specifying database backups

The `platformsh_backup.backups` list specifies the database backups perform, referencing the `platformsh_backup.sources` and `platformsh_backup.remotes` sections for connectivity details.

```yaml
platformsh_backup:
  backups:
    - name: "example.com database"
      source: "example.com"
      env: "live"
      relationship: "my_db"
      disabled: false
      targets: []
```

Where:
* `name` is the display name of the backup. Optional, but makes the logs easier.
* `source` is the name of the key under `platformsh_backups.sources` from which to generate the backup. Required.
* `env` is the Platform.sh environment to backup. Required.
* `relationship` is the name of the database relationship to backup configured in your `platform.app.yaml` file. Required.
* `disabled` is `true` to disable (skip) the backup. Optional, defaults to `false`.
* `targets` is a list of remotes and additional destination information about where to upload backups. Required.

### Specifying file backups

File backups are specified the same way as database backups, save for one parameter difference:

```yaml
platformsh_backup:
  backups:
    - name: "example.com database"
      source: "example.com"
      env: "live"
      mount: "web/site/default/files"
      disabled: false
      targets: []
```

Where:
* `mount` is the path within the Platform site from which to backup files. Required for file backups.

Unlike database backups -- where a single compressed dump file is backed up -- Platform.sh does not offer archived tarballs of files. Instead, this role synchronizes files between the `source` and the `targets` as a whole directory using `rclone`. This removes the need to local cache directories and saves compute resources needed to produce the archive.

### Backup targets

Backup targets reference a key in `platformsh_backup.remotes`, and combine that with additional information used to upload this specific backup.

```yaml
platformsh_backup:
  backups:
    - name: "example.com database"
      source: "example.com"
      env: "live"
      relationship: "my_db"
      targets:
        - remote: "example-s3-bucket"
          path: "example.com/database"
          disabled: true
        - remote: "sftp.example.com"
          path: "backups/example.com/database"
          disabled: false
```

Where:
* `remote` is the key under `platformsh_backup.remotes` to use when uploading the backup. Required.
* `path` is the path on the remote to upload the backup. Optional.
* `disabled` is `true` to skip uploading to the specifed `remote`. Optional, defaults to `false`.

### Rotating database backups

Database backups are uplaoded to the remote with the `&lt;project&gt;.&lt;env&gt;.&lt;relationship&gt;-0.tar.gz`. Often, you'll want to retain previous backups in the case an older backup can aid in research or recovery. This role supports retaining and rotating multiple backups using the `retainCount` key.

```yaml
platformsh_backup:
  backups:
    - name: "example.com database"
      source: "example.com"
      env: "live"
      relationship: "my_db"
      targets:
        - remote: "example-s3-bucket"
          path: "example.com/database"
          retainCount: 3
          disabled: true
        - remote: "sftp.example.com"
          path: "backups/example.com/database"
          retainCount: 3
          disabled: false
```

Where:
* `retainCount` is the total number of backups to retain in the directory. Optional. Defaults to `1`, or no rotation.

During a backup, if `retainCount` is set:
1. The backup with the ending `&lt;retainCount - 1&gt;.tar.gz` is deleted.
2. Starting with `&lt;retainCount - 2&gt;.tar.gz`, each backup is renamed incremending the ending index.
3. The new backup is uploaded with a `0` index as `&lt;site_id&gt;.&lt;env&gt;.&lt;element&gt;-0.tar.gz`.

This feature works for database backups only, and supports both in S3 and SFTP remotes.

### Rotating file backups

As a whole directory is synchronized for files, it is not possible to rotate the backups similar to database dumps. 

An alternative would be to configure this role to execute using slightly different configurations for each backup period. One such way is to change the `path` in your `targets` to include the weekday (`example.com/files/monday`). Then, create multiple configuration files for each weekday. When the next week begins, the previous week's files are overwritten with updates. 

### Ping URL on completion

When a backup completes, you have the option to ping an URL via HTTP:

```yaml
platformsh_backup:
  backups:
    - name: "example.com database"
      source: "example.com"
      env: "live"
      relationship: "my_db"
      healthcheckUrl: "https://pings.example.com/path/to/service"
      targets:
        - remote: "example-s3-bucket"
          path: "example.com/database"
          retainCount: 3
          disabled: true
        - remote: "sftp.example.com"
          path: "backups/example.com/database"
          retainCount: 3
          disabled: false
```

Where:
* `healthcheckUrl` is the URL to ping when the backup completes successfully. Optional.

## Example Playbook

```yaml
    - hosts: servers
      vars:
        platformsh_backup:
          cliPath: "/usr/local/bin/platform"
          debug: true
          sources:
            example.com:
              project: "abcdef1234567"
              cliTokenFile: "/path/to/platform-cli-token.txt"
              keyFile: "/path/to/id_rsa"
              retryCount: 3
              retryDelay: 30
          remotes:
            example-s3-bucket:
              type: "s3"
              bucket: "my-s3-bucket"
              accessKeyFile: "/path/to/aws-s3-key.txt"
              secretKeyFile: "/path/to/aws-s3-secret.txt"
              region: "us-east-1"
            sftp.example.com:
              type: "sftp"
              host: "sftp.example.com"
              user: "example_user"
              keyFile: "/config/id_example_sftp"
              pubKeyFile: "/config/id_example_sftp.pub"
          backups:
            - name: "example.com database"
              source: "example.com"
              env: "live"
              relationship: "my_db"
              targets:
                - remote: "example-s3-bucket"
                  path: "example.com/database"
                  retainCount: 3
                  disabled: true
                - remote: "sftp.example.com"
                  path: "backups/example.com/database"
                  retainCount: 3
                  disabled: false
            - name: "example.com files"
              source: "example.com"
              env: "live"
              mount: "web/sites/default/files"
              targets:
                - remote: "example-s3-bucket"
                  path: "example.com/files"
                  disabled: true
                - remote: "sftp.example.com"
                  path: "backups/example.com/files"
                  disabled: false
      roles:
         - { role: ten7.platformsh_backup }
```

## License

GPL v3

## Author Information

This role was created by [TEN7](https://ten7.com/).
