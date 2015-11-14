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
  * `effective_user`, `effective_group` — optional attributes. Configures a *local* user to use for a 'priviledged' command. <br/> 
    Implementation uses `sudo -u $user`, so `sudo` should be installed and configured as required.<br/> 
    See [Command Invocation](#Command Invocation) below for additional details. If attributes are omitted — current user and group are used;
  * `ssh_key` — optional attribute. If defined, ssh is used with `-i $ssh_key` option.<br/>
    Systems ssh client is used, so ssh configuration is honoured, ssh-agent may be used for auth.
* `service` — describes a single service. Usually it's a single init script;
  * attributes:
    * `id` — obligatory. The display and internal name of a service;
    * `host` — obligatory. A host which runs the service. Should be ssh accessible;
    * `start`, `stop`, `status` — corresponding commands. Any of these may be omitted. See **Command Invocation** below;
  * subtags:
    * `requires` — lists a required service. Can be repeated as needed.<br/> 
       To be precise: `requires` is a tag to establish start-stop order. See **Command invocation** below;
      *  `id` attribute — obligatory, contains `service` tag `id` attribute;
      *  `host` attribute — obligatory, contains required `service` tag `host` attribute;
      *  `strong` attribute — if `false` or `0` then a
* `group` — describes arbitary service grouping. Used for group commads.
  * `service` — obligatory subtag. May be repeated several times if needed.
    * `id` — corresponding  

## Command invocation
## Initial help

```
Usage:
    clinit [command] [options] [selector] ...

Arguments:
    All command-line arguments my come in any order. Each selector is a
    colon separated pair host:service. Where each host or service can be an
    unique identifier, a wildcard or an empty string (in which case it is
    considered as '*').

  Commands:
    start
        Start cluster services

    stop
        Stop cluster services

    status
        Query status of cluster services

    restart
        Stop and then start selected and stopped services

    condrestart
        Stop selected services and then start only stopped services

    list, ls
        Display list of cluster services

    grouplist, gl
        Display list of service groups

    tree
        Display services dependency tree

    keyscan
        Outputs the result of ssh-keyscan for selected hosts.

  Default Configuration File and Environment Variables:
    The default configuration is /etc/clinit/services.xml

    CLINIT_CONFIG environment variable can be used to override the location
    of the default configuration file.

    See configuration file syntax below.

  Options:
    All options require a command to be specified except for --help.

    -S, --services=services.xml file
        Change loaded services descriptor file (default is services.xml)

    -j, --jobs=jobs
        Specifies the number of jobs (commands) to run simultaneously. Use
        -j0 to disable jobs. Default value -j8.

    --no-deps, --nodeps
        Start only services selected by selectors, don't figure out
        dependencies.

    --force-deps
        Check each service dependencies even if it is running.

    --selected-deps
        Resolve dependencies in bounds of selected services. Command(s) will
        affect ONLY selected services. Dependencies determines services
        start/stop order.

        This switch is useful to gracefully stop or start services on
        specified host. Complete ignoring dependencies (--no-deps) may cause
        startup errors.

    --no-remote=HOST
        Execute commands for specified host(s) locally.

        To disable remote execution on several hosts it is possible to use
        option multiple times or specify comma or space separated list:
        --no-remote=localhost --no-remote="local,127.0.0.1 myhost"

    -p, --probe-count=N
        Sets probe count to N before probe procedure considered to be
        failed. Defaults to 10. --probe-count=0 disables probing procedure
        in which case action is considerd to be successful always.

    --probe-delay=N
        Sets delay to N seconds between probe attempts. Default 3 sec.

    -n, --dry-run
        Do not execute commands

    -l, --long-list
        Print more information for *list commands

    -a, --all
        Match hidden services and groups. Without this option hidden
        services and groups matched only it's full names.

    --no-colors, --nocolors
        Disable colors in output.

    -v, --verbose (incremental)
        Print verbose messages about running commands and other debugging
        information.

    -q, --quiet
        Suppress all informational messages from startup system. Not affect
        list and tree commands.

    -h, --help
        Print brief help message and exit.

  Selectors:
    Command line parameters not matched any command or option and NOT
    started with '-' treat as selectors. Selector may include wildcards '*'
    and '?'. Each selector matched against hosts, groups and full service
    names and determines list of services, groups and hosts to apply
    command(s). Selectors with ':' symbol are parsed separately and have the
    special meaning - [host]:[service]. Both host and service parts is
    optional and replaced by '*' if omitted.

    Default selector is 'all' - services from group 'all' or all services if
    no such group in services.xml.

    Selector may be followed by '-' sign which turns it to NEGATIVE selector
    (exclusion) or '+' which is implied by default.

  Configuration File:
    The configuration is a generic XML. The required configuration must be
    enclosed in the services element. Here is a sample:

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

  Configuration File Syntax Description:
    services element has optional attributes:

    effective_user , effective_group
          Defines a user account and group to perform all operation. sudo is
          used to impersonate to the required account.

    ssh_key
          Path to ssh private key to access hosts.

    A service defined by service element.

    service element must have attributes:

    id    The service identifier, will be shown in all output.

    host  The host to execute commands.

    stop, start, status
          Define commands to start, stop and get the status of a service

    requires is optional element to define dependency to other service
    referenced by id and host attributes.

    strong attribute defines whether service can start if dependent on
    service failed. If strong="true" the service will not try to start.

    group is optional element.
          group defines a list of services. The group name attribute can be
          used as argument for clinit commands.
```
## See also

See also <A HREF=example>example</A>

