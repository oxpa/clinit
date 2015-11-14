# ClInit
A lightweight and simple to use cluster management tool.

A name was derived from "**cl**uster **init**". But you can read that as "**cl**oud **init**" because everything is in clouds nowadays.

### What it is
 It is a cluster management tool:
 * runs `start`, `stop` and other management commands you used to see in init scripts;
 * runs commands on a group of hosts simultaneously using **ssh** and honoring dependencies;
 * shows you service status, visualize dependency tree for you and helps you with clusters

### What it is not
* Not a clustering tool: won't detect node failure for you
* Not a deployment tool. While it is possible to install software with clinit, it is not intended usage

## Example usage
```
~]$ clinit grouplist
all
hadoop
hbase
impala
zookeeper-server
~]$ clinit status impala
hat0:impala-catalog                 is started
hat0:impala-state-store             is started
hat1:impala-server                  is started
hat2:impala-server                  is started
hat3:impala-server                  is started
~]$ clinit tree impala
hat0:impala-catalog─┬─hat0:hive-metastore───hat0:mysql-meta
                    └─hat0:impala-state-store
hat2:impala-server─┬─hat0:hdfs-namenode
                   └─hat0:impala-state-store
hat3:impala-server─┬─hat0:hdfs-namenode
                   └─hat0:impala-state-store

```
## Configuration
Default configuration is read from `/etc/clinit/services.xml` . As you might guess, it's a generic XML file. 
```xml
          <?xml version="1.0" encoding="utf-8"?>

          <services xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                    effective_user="root"
                    effective_group="root"
                    ssh_key="/root/.ssh/id_rsa">

               <service id="mariadb" host="localhost"
                        start="service mariadb start"
                        stop="service mariadb  stop"
                        status="service mariadb status">
               </service>

             <service id="httpd" host="localhost"
                      start="service httpd start"
                      stop="service httpd  stop"
                      status="service httpd status">

                  <requires id="mariadb" host="localhost" strong="true"/>
             </service>

            <group name="httpd">
              <service id="httpd" host="localhost"/>
            </group>

            <group name="mariadb">
              <service id="mariadb" host="localhost"/>
            </group>

          </services>
```
* `services` — configuration root tag.
  * `effective_user`, `effective_group` — optional attributes. Configures a *local* user to use for a 'priviledged' command.<br/> 
    Implementation uses `sudo -u $user`, so `sudo` should be installed and configured as required.<br/> 
    See [Command Invocation](#command-invocation) below for additional details. If attributes are omitted — current user and group are used;
  * `ssh_key` — optional attribute. If defined, ssh is used with `-i $ssh_key` option.<br/>
    Systems ssh client is used, so ssh configuration is honoured, ssh-agent may be used for auth.
* `service` — describes a single service. Usually it's a single init script;
  * attributes:
    * `id` — obligatory. The display and internal name of a service;
    * `host` — obligatory. A host which runs the service. Should be ssh accessible;
    * `start`, `stop`, `status` — corresponding commands. Any of these may be omitted. See [Command Invocation](#command-invocation) below;
  * subtags:
    * `requires` — lists a required service. Can be repeated as needed.<br/> 
       To be precise: `requires` is a tag to establish start-stop order. See [Command Invocation](#command-invocation) below;
      *  `id` attribute — obligatory, contains `service` tag `id` attribute;
      *  `host` attribute — obligatory, contains required `service` tag `host` attribute;
      *  `strong` attribute — if `false` or `0` then required service will be attempted to start, but on a fail start sequence will continue. By default it is set to 'true', so dependencies are obligatory.
* `group` — describes arbitary service grouping. Used for group commads.
  * `service` — obligatory subtag. May be repeated several times if needed.
    * `id` — corresponding `service` tag `id` attribute
    * `host — corresponding `service` tag `host` attribute

## Command invocation
All command-line arguments my come in any order.
```
clinit [command] [options] [selectors]
```
### Commands 
Commands are hardcoded. A command marked with `*` is priviledged. It will have `sudo -u $effective_user` issued before the command if the `effective_user` is configured. Following commands possible:
* `start` * — starts selected services;
* `stop` * — stops selected serivces;
* `status` * — queries for selected services status;
* `restart` * — stops and then starts selected services;
* `condrestart` * — stops and then starts only serives it previously stopped;
* `list`, `ls` — display list of services in the configuration;
* `grouplist`, `gl` — dislay list of groups in the configuration;
* `keyscan` — outputs the result of `ssh-keyscan` for selected hosts;
* `tree` — displays services dependency tree

### Default behaviour
This behaviour can be configured with command line options. See below.<br/>
Each time a command is invoked a dependency tree is built and then each service is queried for the status. If a service lacks `status` command - it will be invoked directly, without status queries. If a service is running it's dependencies are not checked<br/> 
If status is not matched - an action is invoked. If there is no corresponding action (`start` or `stop`) then the service is silently skipped.

You may use the latter property to build a pre-start, pre-stop, post-start, post-stop scripts. For example, you may checkout service configuration or run a configuration management utility.

### Selectors
Anything which is not a command or an option and is not started with `-` sign is treated as a selector.

Selectors are:
* built-in `all` keyword. This is the default selector if none is used
* a host name;
* a service name;
* a group name;
* A colon separated `host`:`id` pair, possibly with either part omitted;
* a glob pattern of the above entity with `*` or `?` replacing any part except for a `:` sign in a pair.

If a `host` or `id` is omitted from a pair, missing part is treated as a `*` sign.<br/>
`all` keyword is used for `all` group or all services in a configuration if there is no `all` group.<br/>
Any selector may have `-` (minus sign) suffix to indicate negative selector. Or a `+` sign which is implied. A good example would be:
```
clinit stop all slave-
```
Which should stop everything except for the `slave` group, which will be kept in the state it is. If a service inside `slave` group depends on any other outside of the group — it will still be stopped too by dependency, unless `--selected-deps` is stated

## Options

All options require a command to be specified except for `--help`.

* `-S`, `--services=services.xml` — change loaded services descriptor file (default is services.xml);

* `-j`, `--jobs=N` — specifies the number of jobs (commands) to run simultaneously. Use `-j0` to disable jobs. Default value `-j8`;

* `--no-deps`, `--nodeps` start only services selected by selectors, don't figure out dependencies;

* `--force-deps` check each service dependencies even if it is running;

* `--selected-deps` — resolve dependencies in bounds of selected services. Command(s) will affect ONLY selected services. <br/>
While dependencies determines services start/stop order. This switch is useful to gracefully stop or start services on specified host. Complete ignoring dependencies (`--no-deps`) may cause startup errors.

* `--no-remote=HOST` — Execute commands for specified host(s) locally.<br/>
To disable remote execution on several hosts it is possible to use
option multiple times or specify comma or space separated list:
`--no-remote=localhost --no-remote="local,127.0.0.1 myhost"`

* `-p`, `--probe-count=N` — sets probe count to N before probe procedure considered to be failed. Defaults to 10. `--probe-count=0` disables probing procedure in which case action is considerd to be successful always.

* `--probe-delay=N` — sets delay to N seconds between probe attempts. Default 3 sec.

* `-n`, `--dry-run` — do not execute commands

* `-l`, `--long-list` — print more information for list commands

* `--no-colors`, `--nocolors` — disable colors in output.

* `-v`, `--verbose` (incremental) — print verbose messages about running commands and other debugging information.

* `-q`, `--quiet` — suppress all informational messages from startup system. Doesn't affect list and tree commands.

* `-h`, `--help` — print brief help message and exit.

## Environment
`CLINIT_CONFIG` environment variable can be used to override the location of the default configuration file

## See also

See also <A HREF=example>example</A>


