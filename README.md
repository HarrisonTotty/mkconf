# Summary

The following repo contains a reengineering of an older Apache Web Server configuration utility. This new version, written in Python instead of Perl, leverages the Jinja2 templating system. It is designed to work equally-well as both an interactive "on-demand" script with useful stdout/stderr messages and a hands-off automation-ready script with filesystem logging and configurable reaction emails. This repo includes a very simple set of example templates (`example/templates`) and relevant template configuration file (`example/mkconf.yaml`). This document will henceforth explain the usage of the script in regards to these example files.

## Additional Features

* Extensive validation throughout the configuration process.
* Optional file logging.
* Optional email support.
* "Dry Run" support for generating configurations without saving changes.
* Color output (with option to disable).
* Command-line Jinja token specification.
* Configuration file partitioning to break up monolithic `httpd.conf` files.
* Backing-up the output directory prior to writing to it.
* Default behavior configuration via environment variables.

## System Requirements

* Python 2.7
* [PyYAML](https://pyyaml.org/)
* [Jinja2](http://jinja.pocoo.org/docs/2.10/)
* `rsync` (used for transferring files locally)

## Important Notes

* `mkconf` runs the underlying `rsync` process _with_ the `--delete` flag when taking a backup of the output directory but _without_ the `--delete` flag when writing the newly-generated configuration files to the output directory, _unless_ the `--delete` option is passed to the script.
* Running `mkconf` with `--dry-run` still writes the generated configuration files to the specified working directory. This is so you can manually inspect the files afterward.
* `mkconf` will by default strip trailing newline characters from Jinja block statements. I (Harrison Totty) personally found the default Jinja behavior pretty ugly, so I chose this behavior to be the default instead. Of course, you can disable this "feature" by simply passing `--dont-trim-blocks` to `mkconf`.
* Make sure to let others know if your templates use a non-standard set of Jinja identification tokens. I included the ability to specify the Jinja block start string and other tokens with the understanding that some configuration templates might already need those sequences in the configuration format itself, but please try to keep with the default identifiers whenever possible.
* Note that the Apache Web Server configuration directory (the default output directory `/etc/httpd`) often contains additional symlinks/directories that should be preserved when writing the newly-generated configuration files (`/etc/httpd/modules`, for example).

## Known Bugs & Potential Issues

* The `mkconf`-provided configuration template variable `network_interfaces` is not quite reliable yet. It has been temporarily disabled.
* The `mkconf`-provided Jinja function `require` does not currently validate elements of variables more than one element deep. In other words, `require('vhost.root_domain')` will work, but `require('foo.bar.baz')` will not.

----
# Usage

`mkconf` is invoked with two required arguments: a template configuration (YAML) file, and a template source directory. With regards to this repo, the simplest example would be:

```
$ ./mkconf example/mkconf.yaml example/templates
```

If a directory is supplied as the first argument, then `mkconf` will automatically select a file called `mkconf.yaml` underneath that directory, if it exists. Otherwise, it will select the file whose name (without the `.yaml` extension) is closest to the hostname of the executing machine.

The following table describes the remaining optional arguments:

| Argument(s)                          | Value Type                                   | Default Value    | Description                                                                                                                                                                                                              |
|--------------------------------------|----------------------------------------------|------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `--block-end-string`                 | Generic String                               | `%}`             | Specifies the string marking the end of a Jinja template block.                                                                                                                                                          |
| `--block-start-string`               | Generic String                               | `{%`             | Specifies the string marking the start of a Jinja template block.                                                                                                                                                        |
| `--comment-end-string`               | Generic String                               | `#}`             | Specifies the string marking the end of a Jinja template comment.                                                                                                                                                        |
| `--comment-start-string`             | Generic String                               | `{#`             | Specifies the string marking the start of a Jinja template comment.                                                                                                                                                      |
| `--dont-trim-blocks`                 |                                              |                  | Specifies that the first newline character after a Jinja block should NOT be removed.                                                                                                                                    |
| `--delete`                           |                                              |                  | Specifies that the script should delete any files in the output directory that are not part of the generated configuration. Certain files and subdirectories may be preserved with the `--exclude` option. +             |
| `-d`, `--dry-run`                    |                                              |                  | Specifies that the script should only execute a dry-run, preventing the generated configuration files from being copied from the working directory to the output directory.                                              |
| `-e`, `--email-level`                | `never`, `error`, `warning`, or `completion` | `never`          | Specifies the condition at which the script should send an email.                                                                                                                                                        |
| `-t`, `--email-to`                   | Email Address                                |                  | Specifies the email address to receive sent emails. This option is ignored if `-e` is not specified or set to `never`.                                                                                                   |
| `--exclude`                          | Directory/File Paths                         |                  | Specifies an additional list of files or directories relative to the specified output directory that should be preserved on write (if `--delete` is supplied). ^                                                         |
| `-h`, `--help`                       |                                              |                  | Displays help and usage information.                                                                                                                                                                                     |
| `-i`, `--ignore-validation-warnings` |                                              |                  | Disables printing/logging of warning-level issues during the generated configuration validation process.                                                                                                                 |
| `-f`, `--log-file`                   | File Path                                    |                  | Specifies a log file to write to in addition to stdout/stderr.                                                                                                                                                           |
| `-l`, `--log-level`                  | `info` or `debug`                            | `info`           | Specifies the log level of the script. This option is ignored if `-f` is not specified.                                                                                                                                  |
| `-m`, `--log-mode`                   | `append` or `overwrite`                      | `append`         | Specifies whether to append or overwrite the specified log file on each run of the script. This option is ignored if `-f` is not specified.                                                                              |
| `--no-backup`                        |                                              |                  | Specifies that the script should not perform a backup of the specified output directory prior to writing the generated configuration files.                                                                              |
| `--no-color`                         |                                              |                  | Disables color output on writes to stdout/stderr.                                                                                                                                                                        |
| `-o`, `--output`                     | Directory Path                               | `/etc/httpd`     | Specifies the output directory where the generated templates will be written.                                                                                                                                            |
| `-r`, `--reload-httpd`               |                                              |                  | Specifies that the local Apache Web Server service should be reloaded in-place after the generated configuration has been written to the specified output directory. This option is incompatible with `--restart-httpd`. |
| `--restart-httpd`                    |                                              |                  | Specifies that the local Apache Web Server service should be restarted after the generated configuration has been written to the specified output directory. This option is incompatible with `--reload-httpd`.          |
| `--revert`                           | Directory Path (Optional)                    |                  | Restores the contents of an existing backup directory relative to the specified output directory. If a path is provided alongside this flag, that location will instead be utilized as the restoration source.           |
| `--rsync-executable`                 | File Path                                    | `/usr/bin/rsync` | Specifies the file path to the `rsync` executable.                                                                                                                                                                       |
| `-v`, `--validate`                   | Directory Path (Optional)                    |                  | Performs the validation step for all configuration files in the specified directory, or within the default output directory if none is supplied. *                                                                       |
| `--variable-end-string`              | Generic String                               | `}}`             | Specifies the string marking the end of a Jinja template variable.                                                                                                                                                       |
| `--variable-start-string`            | Generic String                               | `{{`             | Specifies the string marking the start of a Jinja template variable.                                                                                                                                                     |
| `-w`, `--working-directory`          | Directory Path                               | `/tmp/mkconf`    | Specifies the working directory.                                                                                                                                                                                         |

\+ Essentially this enables the "--delete" flag of the underlying rsync process used to sync the working and output directories.

\* This option skips loading of the PyYAML and Jinja2 modules, and thus only requires a correct version of Python.

^ The script automatically preserves the "logs", "modules", and "run" subdirectories.

As an example, the following command would translate the example templates into the `/home/foo/bar` directory and would send an email to `foo@example.com` on completion (or upon encountering any errors):

```
$ ./mkconf example/mkconf.yaml example/templates -o /home/foo/bar --email-to 'foo@example.com' -e completion
```

## Script Output

The output of a typical run may look something like this on stderr/stdout:

```
$ ./mkconf example/mkconf.yaml example/templates -f ~/tst.log -m overwrite -o ~/example.out
:: Validating working environment...
  --> Validating rsync executable path...
  --> Validating template source directories...
  --> Validating template source files...
  --> Validating template configuration file...
  --> Validating working directory...
:: Loading template configuration file...
  --> Reading template configuration file...
  --> Parsing template configuration file...
  --> Validating template configuration...
:: Setting-up templating environment...
  --> Initializing loader...
  --> Initializing environment...
  --> Initializing extensions...
:: Translating templates...
  --> vhosts.d/bar.conf
  --> vhosts.d/foo.conf
  --> conf/filemaps.conf
  --> vhosts.d/baz.conf
  --> conf/modules.conf
  --> conf/httpd.conf
:: Validating configuration files...
  --> vhosts.d/bar.conf
      [19] Warning: DocumentRoot directive specifies potentially non-existent directory "/www/sites/www.devel.bar.example.com/htdocs".
      [22] Warning: ServerName directive specification "www.devel.bar.example.com" does not resolve in DNS.
      [25] Warning: ServerAlias directive specification "sub1.devel.bar.example.com" does not resolve in DNS.
      [26] Warning: ServerAlias directive specification "sub2.devel.bar.example.com" does not resolve in DNS.
      [27] Warning: ServerAlias directive specification "sub3.devel.bar.example.com" does not resolve in DNS.
      [28] Warning: ServerAlias directive specification "sub4.devel.bar.example.com" does not resolve in DNS.
      [31] Warning: ServerAlias directive specification "alternative.devel.example.com" does not resolve in DNS.
      [35] Warning: TransferLog directive specifies potentially non-existent parent directory "/www/logs/www.devel.bar.example.com".
      [36] Warning: ErrorLog directive specifies potentially non-existent parent directory "/www/logs/www.devel.bar.example.com".
  --> vhosts.d/foo.conf
      [19] Warning: DocumentRoot directive specifies potentially non-existent directory "/www/sites/www.devel.example.com/htdocs".
      [22] Warning: ServerName directive specification "www.devel.example.com" does not resolve in DNS.
      [25] Warning: ServerAlias directive specification "sub1.devel.example.com" does not resolve in DNS.
      [26] Warning: ServerAlias directive specification "sub2.devel.example.com" does not resolve in DNS.
      [27] Warning: ServerAlias directive specification "sub3.devel.example.com" does not resolve in DNS.
      [28] Warning: ServerAlias directive specification "sub4.devel.example.com" does not resolve in DNS.
      [33] Warning: TransferLog directive specifies potentially non-existent parent directory "/www/logs/www.devel.example.com".
      [34] Warning: ErrorLog directive specifies potentially non-existent parent directory "/www/logs/www.devel.example.com".
  --> conf/filemaps.conf
  --> vhosts.d/baz.conf
      [25] Warning: DocumentRoot directive specifies potentially non-existent directory "/www/sites/baz.devel.example.com/htdocs".
      [28] Warning: ServerName directive specification "baz.devel.example.com" does not resolve in DNS.
      [34] Warning: TransferLog directive specifies potentially non-existent parent directory "/www/logs/baz.devel.example.com".
      [35] Warning: ErrorLog directive specifies potentially non-existent parent directory "/www/logs/baz.devel.example.com".
  --> conf/modules.conf
  --> conf/httpd.conf
      [16] Warning: DocumentRoot directive specifies potentially non-existent directory "/www/sites/harrisontlxlap.localdomain/htdocs".
      [19] Warning: ErrorLog directive specifies potentially non-existent parent directory "/www/logs/harrisontlxlap.localdomain".
      [24] Warning: PidFile directive specifies potentially non-existent parent directory "/etc/httpd/run".
      [30] Warning: ServerRoot directive specifies potentially non-existent directory "/etc/httpd".
:: Finalizing configuration process...
  --> Creating backup of existing configuration...
  --> Writing configuration files to output directory...
```

Note the warnings during the validation step since I ran the above from my local machine.

The corresponding log file for the above run looks like this:

```
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.main] Started configuration process.
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_environment] Validating working environment...
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.parse_yaml_config] Loading template configuration file...
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.setup_jinja] Setting-up templating environment...
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.translate_templates] Translating templates...
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.translate_templates] Translating "example/templates/vhosts.d/bar.conf" into "/tmp/mkconf/vhosts.d/bar.conf"...
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.translate_templates] Translating "example/templates/vhosts.d/foo.conf" into "/tmp/mkconf/vhosts.d/foo.conf"...
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.translate_templates] Translating "example/templates/conf/filemaps.conf" into "/tmp/mkconf/conf/filemaps.conf"...
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.translate_templates] Translating "example/templates/vhosts.d/baz.conf" into "/tmp/mkconf/vhosts.d/baz.conf"...
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.translate_templates] Translating "example/templates/conf/modules.conf" into "/tmp/mkconf/conf/modules.conf"...
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.translate_templates] Translating "example/templates/conf/httpd.conf" into "/tmp/mkconf/conf/httpd.conf"...
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_configs] Validating configuration files...
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_configs] Validating "vhosts.d/bar.conf"...
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 19) DocumentRoot directive specifies potentially non-existent directory "/www/sites/www.devel.bar.example.com/htdocs".
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 22) ServerName directive specification "www.devel.bar.example.com" does not resolve in DNS.
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_configs] Relevant ServerName resolution exception: [Errno -2] Name or service not known
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 25) ServerAlias directive specification "sub1.devel.bar.example.com" does not resolve in DNS.
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_configs] Relevant ServerAlias resolution exception: [Errno -2] Name or service not known
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 26) ServerAlias directive specification "sub2.devel.bar.example.com" does not resolve in DNS.
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_configs] Relevant ServerAlias resolution exception: [Errno -2] Name or service not known
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 27) ServerAlias directive specification "sub3.devel.bar.example.com" does not resolve in DNS.
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_configs] Relevant ServerAlias resolution exception: [Errno -2] Name or service not known
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 28) ServerAlias directive specification "sub4.devel.bar.example.com" does not resolve in DNS.
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_configs] Relevant ServerAlias resolution exception: [Errno -2] Name or service not known
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 31) ServerAlias directive specification "alternative.devel.example.com" does not resolve in DNS.
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_configs] Relevant ServerAlias resolution exception: [Errno -2] Name or service not known
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 35) TransferLog directive specifies potentially non-existent parent directory "/www/logs/www.devel.bar.example.com".
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 36) ErrorLog directive specifies potentially non-existent parent directory "/www/logs/www.devel.bar.example.com".
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_configs] Validating "vhosts.d/foo.conf"...
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 19) DocumentRoot directive specifies potentially non-existent directory "/www/sites/www.devel.example.com/htdocs".
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 22) ServerName directive specification "www.devel.example.com" does not resolve in DNS.
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_configs] Relevant ServerName resolution exception: [Errno -2] Name or service not known
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 25) ServerAlias directive specification "sub1.devel.example.com" does not resolve in DNS.
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_configs] Relevant ServerAlias resolution exception: [Errno -2] Name or service not known
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 26) ServerAlias directive specification "sub2.devel.example.com" does not resolve in DNS.
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_configs] Relevant ServerAlias resolution exception: [Errno -2] Name or service not known
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 27) ServerAlias directive specification "sub3.devel.example.com" does not resolve in DNS.
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_configs] Relevant ServerAlias resolution exception: [Errno -2] Name or service not known
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 28) ServerAlias directive specification "sub4.devel.example.com" does not resolve in DNS.
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_configs] Relevant ServerAlias resolution exception: [Errno -2] Name or service not known
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 33) TransferLog directive specifies potentially non-existent parent directory "/www/logs/www.devel.example.com".
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 34) ErrorLog directive specifies potentially non-existent parent directory "/www/logs/www.devel.example.com".
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_configs] Validating "conf/filemaps.conf"...
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_configs] Validating "vhosts.d/baz.conf"...
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 25) DocumentRoot directive specifies potentially non-existent directory "/www/sites/baz.devel.example.com/htdocs".
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 28) ServerName directive specification "baz.devel.example.com" does not resolve in DNS.
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_configs] Relevant ServerName resolution exception: [Errno -2] Name or service not known
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 34) TransferLog directive specifies potentially non-existent parent directory "/www/logs/baz.devel.example.com".
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 35) ErrorLog directive specifies potentially non-existent parent directory "/www/logs/baz.devel.example.com".
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_configs] Validating "conf/modules.conf"...
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.validate_configs] Validating "conf/httpd.conf"...
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 16) DocumentRoot directive specifies potentially non-existent directory "/www/sites/harrisontlxlap.localdomain/htdocs".
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 19) ErrorLog directive specifies potentially non-existent parent directory "/www/logs/harrisontlxlap.localdomain".
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 24) PidFile directive specifies potentially non-existent parent directory "/etc/httpd/run".
[WAR] [09/11/2018 02:54:05 PM] [25799] [mkconf.validation_issue] (Line 30) ServerRoot directive specifies potentially non-existent directory "/etc/httpd".
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.write_output] Finalizing configuration process...
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.write_output] Creating backup of existing configuration...
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.write_output] Writing configuration files to output directory...
[INF] [09/11/2018 02:54:05 PM] [25799] [mkconf.main] Configuration process complete.
```
Notice that the log file follows the following format:

```
[LOG_LEVEL] [TIMESTAMP] [PROCESS_ID] [MODULE.FUNCTION] MESSAGE
```

## Exit Codes

The script not only returns non-zero exit codes on fatal errors, but even broadly categorizes them:

| Code | Description                                                                                                              |
|------|--------------------------------------------------------------------------------------------------------------------------|
| 0    | Script successfully ran, although perhaps with warnings.                                                                 |
| 1    | Generic issue prior to the environment validation step (invalid arguments, import exceptions, etc).                      |
| 2    | Issue during the environment validation step.                                                                            |
| 3    | Issue with loading, parsing, or validating the template configuration file.                                              |
| 4    | Issue with instantiating or configuring the Jinja templating environment.                                                |
| 5    | Issue with translating the source templates or saving them to the working directory.                                     |
| 6    | Issue with reading or validating the newly-generated configuration files within the working directory.                   |
| 7    | Issue with performing the backup of the existing configuration or writing the new configuration to the output directory. |
| 8    | Issue with reloading or restarting the Apache Web Server service (if `-r` or `--restart-httpd` is supplied).             |
| 9    | Issue occuring during a `--revert` operation.                                                                            |
| 100  | Script was interrupted via CTRL+C or CTRL+D.                                                                             |

## Environment Variables

The default behavior of the `mkconf` script may also be configured via environment variables. Each environment variable has an associated command line argument, as described in the table below:

| Environment Variable       | Corresponding CLI Argument |
|----------------------------|----------------------------|
| `MKCONF_BLOCK_END_STR`     | `--block-end-string`       |
| `MKCONF_BLOCK_START_STR`   | `--block-start-string`     |
| `MKCONF_COMMENT_END_STR`   | `--comment-end-string`     |
| `MKCONF_COMMENT_START_STR` | `--comment-start-string`   |
| `MKCONF_EMAIL_LVL`         | `--email-level`            |
| `MKCONF_EMAIL_TO`          | `--email-to`               |
| `MKCONF_EXCLUDE`           | `--exclude`                |
| `MKCONF_LOG_FILE`          | `--log-file`               |
| `MKCONF_LOG_LVL`           | `--log-level`              |
| `MKCONF_LOG_MODE`          | `--log-mode`               |
| `MKCONF_OUTPUT`            | `--output`                 |
| `MKCONF_RSYNC_PATH`        | `--rsync-executable`       |
| `MKCONF_VAR_END_STR`       | `--variable-end-string`    |
| `MKCONF_VAR_START_STR`     | `--variable-start-string`  |
| `MKCONF_WORKING_DIR`       | `--working-directory`      |

As an example, if `MKCONF_WORKING_DIR` was set to `/tmp/foo` but `-w /tmp/bar` was passed to `mkconf` via command-line, then `mkconf` would use the value of `/tmp/bar` for the working directory.

## Handling Failure Cases

In the case that a falure occurs prior to the finalization step (in other words, a failure case that does not produce an exit code of `7` or `8`), it can be assumed that the existing Apache Web Server configuration has not been corrupted and thus no restoration of a backup is necessary. However in the event that `mkconf` encounters a critical issue during the finalization step, the user may quickly revert any changes by restoring the dedicated backup directory by executing:

```
$ mkconf --revert
```

Note that it is often the case that the Apache Web Server service should be reloaded/restarted after the restoration is complete. This can be accomplished by simply adding the `-r` or `--restart-httpd` flags along with the `--revert` option.

In additon to restoring the configuration from the dedicated backup directory, the `--revert` option can also be specified in conjection with a specific source directory. The following example would restore the configuration from `/foo/bar/2018.backup` and restart the Apache Web Server service on completion:

```
$ mkconf --revert /foo/bar/2018.backup --restart-httpd
```

----
# Templating Layout

## Provided Jinja Expressions

`mkconf` provides several useful additional variables and functions that are available to all configuration templates.

### Provided Variables

| Variable             | Description                                                      | Example                                                                                                                        |
|----------------------|------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| `file`               | The name of the current template file.                           | `httpd.conf`                                                                                                                   |
| `fqdn`               | The fully-qualified domain name of the machine running `mkconf`. | `foo.example.com`                                                                                                              |
| `hostname`           | The hostname of the machine running `mkconf`.                    | `foo`                                                                                                                          |
| `network_interfaces` | A dictionary of network interfaces (see below).                  | `{'eth0': {'ip_address': '10.10.0.1', 'netmask': '255.255.224.0'}, 'lo': {'ip_address': '127.0.0.1', 'netmask': '255.0.0.0'}}` |

In regards to `network_interfaces`, this value is populated during the initial stages of the script's runtime. Note that for this dictionary to be populated, the machine running `mkconf` must be a GNU/Linux machine with `ifconfig` or `ip` installed. This value is a dictionary of dictionaries with keys that correspond to the name of various network interfaces. Each entry has the following corresponding key-value pairs:

| Key          | Description                                                    |
|--------------|----------------------------------------------------------------|
| `ip_address` | The IPv4 address bound to the corresponding network interface. |
| `netmask`    | The netmask bound to the corresponding network interface.      |

For virtual host configuration templates, the `vhost` variable is set to the relevant `virtual_hosts` list item (see _Template Configuration File_). As a side note, since `vhost.file` has the same value as `file`, it's probably shorter to just use `file` everywhere.

### Provided Functions

| Function      | Description                                                                                                    | Example Usage                                                     |
|---------------|----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| `get_host`    | Returns the reverse record for the specified IP address.                                                       | `get_host('8.8.8.8')`                                             |
| `get_ip`      | Returns the IPv4 address associated with the specified host string.                                            | `get_ip('www.google.com')`                                        |
| `domain_join` | Joins each argument as part of a domain. Calls `return '.'.join([x.strip('.') for x in args])` under the hood. | `domain_join('www', root_directory)`                              |
| `path_join`   | Joins each argument as a file path. Literally a subcall to Python's `os.path.join`.                            | `path_join('/www/sites/', site_directory, 'htdocs')`              |
| `print`       | Prints the specified string during the templating process.                                                     | `print('The site directory for this file was set to' + site_dir)` |
| `raise`       | Raises an exception during the templating process with the specified string.                                   | `raise('Invalid value for foo variable: ' + foo)`                 |
| `require`     | Raises an exception if any of the arguments are not present in the template configuration file.                | `require('log_dir', 'sites_dir')`                                 |

Note that each of the above functions will _only work_ in a Jinja _variable specification_ (`{{ }}`), and not within a Jinja _block specification_ (`{% %}`).

## Template Configuration File

The template configuration file (`example/mkconf.yaml`) is a YAML file which contains the global variables accessible to the various configuration templates. In addition to being utilized by the Jinja templates themselves, `mkconf` also directly acts on certain required variables within the file. The current required variables are the `templates` variable and the `virtual_hosts` variable.

The `templates` variable defines a list of "basic" configuration files within the template directory to be translated and written to the output directory. These configuration files are specified with paths relative to the specified template directory. Note that this list _does not_ include the list of virtual host templates to include (see `virtual_hosts`). By default, the output file relative path and name is identical to the input template file, however this translation by be overwritten by specifying `template` alongside `file`. An example `templates` specification is given below:

```
templates:
    # Include all ".conf" files within the "conf" subdirectory of the specified
    # template directory.
    - file: "conf/*.conf"
    
    # Import foo/baz.conf and save it as foo/bar.conf
    - file: "foo/bar.conf"
      template: "foo/baz.conf"
```

The `virtual_hosts` variable defines a list of dictionaries specifying virtual host configuration files to include in the output directory, relative to the `vhosts.d` subdirectory of the specified configuration template directory as the value of the `file` key. Additional variables that are specific to each virtual host should also be specified here. In the example included within this repo, each virtual host specification includes the relevant root domain, primary subdomain, additional subdomains, and any additional alias domains. `mkconf` automatically sets the value of `vhost` to the relevant entry in `virtual_hosts` so that one may simply refer to variables specific to that virtual host via `vhost.foo` or `vhost['foo']`. Below is an example specification:

```
virtual_hosts:
  - file: "a.conf"
    foo: true
    
  - file: "b.conf"
    foo: false
```

In the above example, the value of `vhost.foo` would be `true` in `a.conf` but `false` in `b.conf`. Just like with the `templates` variable, you can also distinguish betwen input and output file names by specifying an additioal `template` key. This allows you to re-use the same template for different output files, like so:

```
virtual_hosts:
   - file: "foo.conf"
     template: "example.template"
     baz: true
     
   - file: "bar.conf"
     template: "example.template"
     baz: false
```

----
# Validation Features

`mkconf` supports an extensive list of validation checks to ensure that most issues with the configuration process or the generated configuration are found accurately and early. Since `mkconf` is designed for either running directly on the target machine or elsewhere (and then copying the configurations to the target machine), the script supports validating configs both during the configuration process and on-demand via the `-v` (`--validate`) option.

Validation in `mkconf` is broken into four separate types/stages: _argument validation_, _environment validation_, _template configuration validation_, and _generated configuration validation_. The subsections below encapsulate what checks are performed during each step:

## Argument Validation

The vast majority of the argument validation step is handled via Python's built-in `argparse` module, however some additional checks on arguments are made prior to initializing the logging system:

* Verify that `--email-to` is explicitly specified if `--email-level` is set to something other than `never`.
* Verify that environment variables corresponding with a finite set of possible choices are valid.
* Verify that it is not the case that both `--reload-httpd` and `--restart-httpd` are specified.
* Verify that it is not the case that both `--revert` and `--validate` are specified.
* Verify that `--validate` is not specified with `--reload-httpd` or `--restart-httpd`.

## Environment Validation

After logging is set-up and the hostname of the machine is established, the script will validate the working environment, to include:

* Verifying the existence of the specified template source directory, and `conf` and `vhosts.d` subdirectories.
* Verifying the existence of `conf/httpd.conf` (the only _required_ configuration file).
* Verifying the existence of the specified template configuration file.
* Creation of the specified working directory if it does not exist, or cleaning of the working directory if it does exist.
* Verifying the existence of the specified `rsync` executable path.

## Template Configuration Validation

After the template configuration file has been correctly parsed, the following validation checks are made against the resulting data structure:

* The existence and data type of the required `templates` variable.
* The existence of each template file specified within the `templates` variable.
* The existence and data type of the required `virtual_hosts` variable.
* The existence of each virtual host template specified within the `virtual_hosts` variable.

## Generated Configuration Validation

After the templates have been translated into the specified working directory, each of the newly generated configuration files is re-read with the following validation checks:

* That "group directives" (`<foo bar>`) are followed by a corresponding closing statement (`</foo>`).
* That _absolute-path_ `DocumentRoot` directives point to an existing directory. Note that `mkconf` currently will not validate _relative-path_ `DocumentRoot` directives.
* That _absolute-path_ `ErrorLog` directives point to a location with an existing parent directory. Note that `mkconf` currently will not validate _relative-path_ `ErrorLog` directives.
* That _absolute-path_ `Include` directives point to an existing location relative to the _current_ or _future_ filesystem layout. In other words, some logic is used to discern if any configuration files in current working directory will end-up with an absolute path that matches the one specified by the include. Of course, it will also check to see if the specified file already exists there.
* That _relative-path_ `Include` directives point to an existing location relative to the current working directory. Note that although `Include` directives support wildcarded paths, `mkconf` currently will not validate these wildcards.
* That _absolute-path_ `PidFile` directives point to a location with an existing parent directory. Note that `mkconf` currently will not validate _relative-path_ `PidFile` directives.
* That `ServerName` and `ServerAlias` directives have a valid host string and resolve in DNS.
* That _absolute-path_ `ServerRoot` directives point to an existing directory. Note that `mkconf` currently will not validate _relative-path_ `ServerRoot` directives.
* That _absolute-path_ `SSLCACertificateFile` directives point to an existing file. Note that `mkconf` currently will not validate _relative-path_ `SSLCACertificateFile` directives.
* That _absolute-path_ `SSLCertificateFile` directives point to an existing file. Note that `mkconf` currently will not validate _relative-path_ `SSLCertificateFile` directives.
* That _absolute-path_ `SSLCertificateKeyFile` directives point to an existing file. Note that `mkconf` currently will not validate _relative-path_ `SSLCertificateKeyFile` directives.
* That _absolute-path_ `TransferLog` directives point to a location with an existing parent directory. Note that `mkconf` currently will not validate _relative-path_ `TransferLog` directives.

Note that the majority of the above validations will only fail at a _warning_ level. This is to allow configuration files to be generated on a different machine and then moved to their final destination.

This step can also be executed on-demand via the `-v` (`--validate`) option. This is useful for running validation on target servers that don't have the PyYAML or Jinja2 system requirements, since this option automatically disables the loading of those modules. By default, this option will validate the default output directory (which may be overloaded via the `MKCONF_OUTPUT` environment variable), but a specific directory may also be supplied like so:

```
$ mkconf --validate /foo/bar
```
