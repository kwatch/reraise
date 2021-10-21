<!-- # -*- coding: utf-8 -*- -->

Reraise README
==============

Reraise is a service process supervisor tool, like runit, dameontools, Supervisor (python), or God (ruby).

* runit <http://smarden.org/runit/>
* daemontools <https://cr.yp.to/daemontools.html>
* Supervisor (python) <http://supervisord.org/>
* God (ruby) <http://godrb.com/>

Compared with them, Reraise has the following features:

* No need root-user priviledge.
* Written in shell script.
* Small memory footprint (thanks to shell script).
* Intuitive CLI interface.

Reraise requires:

* UNIX-like platform (Linux, BSD, or macOS).
* `ps` command (you may install `procps` package in Ubuntu).
* (optional) `sudo` command (if you want run service process in other user priviledge)



Install and Setup
-----------------

```terminal
### download `reraise` and `auto-reraise` scripts
$ curl -o reraise      -L http://bit.ly/reraise_
$ curl -o auto-reraise -L http://bit.ly/auto-reraise_

### install scripts
$ chmod a+x reraise auto-reraise
$ mkdir -p ~/bin
$ mv reraise auto-reraise ~/bin
$ export PATH=$PATH:~/bin
$ which reraise auto-reraise
/home/<yourname>/bin/reraise
/home/<yourname>/bin/auto-reraise

### create directories
$ mkdir ~/reraise.d
$ mkdir /var/tmp/reraise
# or: sudo mkdir /var/run/reraise; chown `whoami` /var/run/reraise

### set environment variables
$ export RERAISE_DIR="$HOME/reraise.d"
$ export RERAISE_PIDDIR="/var/tmp/reraise"
```

If `ps` command not found in Ubuntu, install 'procps' package.

```terminal
$ which ps         # not found
$ sudo apt install procps
$ which ps
/usr/bin/ps
```



Quick Tutorial
--------------

```terminal
### see help message
$ reraise --help

### create service starter script
$ reraise skeleton > $RERAISE_DIR/http-8000
$ vi $RERAISE_DIR/http-8000
$ chmod +x $RERAISE_DIR/http-8000     # !!important!!

### start/restart/stop service process
$ reraise status
$ reraise start http-8000      # or: reraise start 'http-*'
$ reraise restart http-8000    # or: reraise restart '*'
$ reraise stop http-8000       # or: reraise stop '*'
```

If you want to run service process in other user and group,
change owner and group of service starter script.
In this case, `sudo` command required.

```terminal
$ sudo useradd app1user
$ sudo -u app1user whoami
app1user
$ chown app1user:app1user $RERAISE_DIR/http-8000
$ chown app1user:app1user $RERAISE_PIDDIR
$ reraise start http-8000       # service process owner is app1user
```



Configuration
-------------

Line `config_xxxx` in service starter script will affect to behavior of `auto-reraise`.


### `config_start_sec`

Line `config_start_sec=10` in service starter script is important.

* If service process exited within 10 seconds, `auto-reraise` regards it
  as 'service proces failed to start'.
* If service process failed to start over 10 seconds, `auto-reraise` regards it
  as 'service proces started successfully, but exited in error'.

Default value of `config_start_sec` is `10`. Change it according to the followings:

* If service process starts up quickly (like Go), specify small value to `config_start_sec`.
  For example: `config_start_sec=5`.
* If service process takes long time to start up (like Java), specify large value to `config_start_sec`.
  For example: `config_start_sec=30`.


### `config_max_retry`

Line `config_max_retry=2` in service starter script controls retry times.

* `reraise start <service>` will NOT retry even if service process failed to start.
  This is an intended specification.
* After service process started successfully, `auto-reraise` will retry to start
  service process 2 times when service process exited.

Default value of `config_max_retry` is `2`.
Normally it is no need to change this value, but you can change it to large value if you want.



Tips
----

* `reraise clean` removes remained PID files in `$RERAISE_PIDDIR`.
* One service starter script can invoke one service process.
  If you want invoke multiple service processes, create multple starter scripts.
* Notice that `reraise restart` doesn't start not-running service process.



Memory Footprints
-----------------

* Reraise consumes only 1.6MB memory per service process. Thanks to shell script.

  ```terminal
  $ reraise start http-8000
  $ ps aux | awk 'NR==1||/rerais[e]/'
  USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
  ubuntu    263236  0.0  0.1   2608  1688 ?        S    10:51   0:00 /bin/sh /usr/local/bin/auto-reraise http-8000
  ```

* 'runit' and 'daemontools' consume less than 1MB memroy, because they are implemented in C language.

  ```terminal
  $ ps aux | awk 'NR==1||/runs[v]/'
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root      278656  0.0  0.0   2372   672 ?        Ss   15:30   0:00 runsv app1
root      277348  0.0  0.0   2524   676 ?        Ss   15:17   0:00 runsvdir -P /etc/service log: ...
  ```

* 'Supervisor' (python) consumes over 23MB, because it is implemented in Python.

  ```terminal
  $ sudo apt install supervisor
  $ ps aux | awk 'NR==1||/superviso[r]/'
  USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
  root      274693  0.1  2.3  31264 23380 ?        Ss   15:04   0:00 /usr/bin/python3 /usr/bin/supervisord -n -c /etc/supervisor/supervisord.conf
  ```

* 'God' (ruby) consumes over 33MB, because it is implemented in Ruby.

  ```terminal
  $ sudo apt install supervisor
  $ god
  $ ps aux | awk 'NR==1||/go[d]/'
  USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
  ubuntu    277066  0.3  3.3 289684 33408 pts/0    Sl   15:15   0:00 /usr/bin/ruby /usr/bin/god
  ```



Architecture
------------

### How `reraise start` command works

1. `reraise start <service>` invokes `auto-reraise <service>` command (and exits).
2. `auto-reraise <service>` invokes `$RERAISE_DIR/<service>` script (and not-exit).
3. `$RERAISE_DIR/<service>` script executes service process.
4. `auto-reraise` process creates `$RERAISE_PIDDIR/<service>` file (pid file).
5. `auto-reraise` process waits for service process to exit with `wait` command.
6. If service process exited, `auto-reraise` process invokes service process again automatically.
7. Goto 4.

(`auto-reraise` process itself is not supervised. This may be a weak point of Reraise compared to runit or daemontools.)


### How `reraise stop` command works

1. `reraise stop <service>` finds `$RERAISE_PIDDIR/<service>` file (pid file).
2. `reraise` command gets `auto-reraise` process id (ppid) from that file.
3. `reraise` command sends TERM signal to `auto-reraise` process (`kill -TERM $ppid`).
4. `auto-reraise` process sends TERM signal to service process.
5. `auto-reraise` process exits after removing `$RERAISE_PIDDIR/<service>` file.


### How `reraise restart` command works

1. `reraise restart <service>` finds `$RERAISE_PIDDIR/<service>` file (pid file).
2. `reraise` command gets service process id (pid) from that file.
3. `reraise` command sends TERM signal to service process (`kill -TERM $pid`).
4. `auto-reraise` process invokes service process again automatically.
<!--
2. `reraise` command gets `auto-reraise` process id (ppid) from that file.
3. `reraise` command sends HUP signal to `auto-reraise` process (`kill -HUP $ppid`).
4. `auto-reraise` process sends TERM signal to service process.
5. `auto-reraise` process invokes service process again automatically.
-->


### How `auto-reraise` recognizes configuration in service starter script?

1. `auto-reraise` executes `grep '^config_' $RERAISE_DIR/<service>`.
2. `auto-reraise` evaluates the result with `eval`.



FAQ
---

<dl>

<dt>[Q] What is the origin of names ('reraise' and 'auto-reraise') ?</dt>
<dd>
[A] From Final Fantasy.

* <https://finalfantasy.fandom.com/wiki/Reraise_(ability)>
* <https://finalfantasy.fandom.com/wiki/Reraise_(status)>
* <https://ffxi.allakhazam.com/wiki/Auto-Reraise>
</dd>

</dl>



License and Copyright
---------------------

* $License: Public Domain $
* $Copyright: 2021 kuwata-lab.com all rights reserved $
