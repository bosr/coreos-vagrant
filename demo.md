# 1. `etcd` native
Let's run `etcd` and play with it outside CoreOS. Note: we're talking only on `etcd 2.x` here; make no confusion with `etcd 0.4.x`.

For this part, please `cd` into the directory named `demo/1.etcd_native`

## run `etcd` locally in static mode
Resource: [CoreOS Clustering in etcd docs](https://coreos.com/etcd/docs/latest/clustering.html)

There are [3 modes for bootstrapping](https://coreos.com/etcd/docs/latest/clustering.html) `etcd`: static, etcd discovery, dns discovery. In order to cut down on complexity, let's run etcd locally first, and bootstrap the cluster in static mode.

Normally, we would launch 3 instances of `etcd` (but we'll do it in an easier way):

    etcd -name infra0 -initial-advertise-peer-urls http://10.0.1.10:2380 \
      -listen-peer-urls http://10.0.1.10:2380 \
      -listen-client-urls http://10.0.1.10:2379,http://127.0.0.1:2379 \
      -advertise-client-urls http://10.0.1.10:2379 \
      -initial-cluster-token etcd-cluster-1 \
      -initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
      -initial-cluster-state new

    $ etcd -name infra1 -initial-advertise-peer-urls http://10.0.1.11:2380 \
      -listen-peer-urls http://10.0.1.11:2380 \
      -listen-client-urls http://10.0.1.11:2379,http://127.0.0.1:2379 \
      -advertise-client-urls http://10.0.1.11:2379 \
      -initial-cluster-token etcd-cluster-1 \
      -initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
      -initial-cluster-state new

    $ etcd -name infra2 -initial-advertise-peer-urls http://10.0.1.12:2380 \
      -listen-peer-urls http://10.0.1.12:2380 \
      -listen-client-urls http://10.0.1.12:2379,http://127.0.0.1:2379 \
      -advertise-client-urls http://10.0.1.12:2379 \
      -initial-cluster-token etcd-cluster-1 \
      -initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
      -initial-cluster-state new

Instead, we'll use a `Procfile`, which basically is a Makefile for `goreman` (the reason being I saw Brandon Philips do it and it was cool). The content of `Procfile` is simply:

    etcd1: etcd -name infra1 -listen-client-urls http://localhost:4001 -advertise-client-urls http://localhost:4001 -listen-peer-urls http://localhost:7001 -initial-advertise-peer-urls http://localhost:7001 -initial-cluster-token etcd-cluster-1 -initial-cluster 'infra1=http://localhost:7001,infra2=http://localhost:7002,infra3=http://localhost:7003' -initial-cluster-state new

    etcd2: etcd -name infra2 -listen-client-urls http://localhost:4002 -advertise-client-urls http://localhost:4002 -listen-peer-urls http://localhost:7002 -initial-advertise-peer-urls http://localhost:7002 -initial-cluster-token etcd-cluster-1 -initial-cluster 'infra1=http://localhost:7001,infra2=http://localhost:7002,infra3=http://localhost:7003' -initial-cluster-state new

    etcd3: etcd -name infra3 -listen-client-urls http://localhost:4003 -advertise-client-urls http://localhost:4003 -listen-peer-urls http://localhost:7003 -initial-advertise-peer-urls http://localhost:7003 -initial-cluster-token etcd-cluster-1 -initial-cluster 'infra1=http://localhost:7001,infra2=http://localhost:7002,infra3=http://localhost:7003' -initial-cluster-state new


In order to run this file, we'll need a Go installation somewhere and a `$GOPATH` properly defined, and possibly added to our `$PATH`. Assuming this is done, let's install `goreman`.

    go get github.com/mattn/goreman

Then simply launch our 3 `etcd` instances

    goreman start

The stdout/stderr of each of the 3 processes is logged into the terminal.

In another terminal, check that 3 etcd processes are running (using `ps` for instance), and check the cluster health

    etcdctl cluster-health

So, the cluster is healthy.

## Play a bit with `etcd`
Resource: [Getting started with etcd on CoreOS](https://coreos.com/etcd/docs/latest/getting-started-with-etcd.html)

### curl interface

`etcd` is available at `127.0.0.1`. Stores keys in a hierarchy.

#### basics

Set a key and value `{"username": "romain"}` under `/app01`

    http put http://127.0.0.1:4001/v2/keys/app01/username value==romain

List the directory "app01"

    http http://127.0.0.1:4001/v2/keys/app01

Get the value for key "username"

    http http://127.0.0.1:4001/v2/keys/app01/username

#### Watch for a change on a value

    http http://127.0.0.1:4001/v2/keys/app01/username\?wait=true
    curl http://127.0.0.1:4001/v2/keys/app01/username\?wait=true

And from another process

    http put http://127.0.0.1:4001/v2/keys/app01/username value==romain

watch for a change on a directory (recursively in the directory)

    curl http://127.0.0.1:4001/v2/keys/app01\?wait=true\&recursive=true

And again from another process

    http put http://127.0.0.1:4001/v2/keys/app01/username value==alex


### `etcdctl` client

Makes it easy to `set`, `get`, `ls`, `watch`, get `cluster-health`, etc.

    etcdctl ls /app01
    etcdctl get /app01/username
    etcdctl watch /app01 --forever --recursive
    # and someone makes
    etcdctl set /app01/username alex

#### watching the directory and triggering an executable

    etcdctl exec-watch --recursive /foo-service -- sh -c 'echo "\"$ETCD_WATCH_KEY\" key was updated to \"$ETCD_WATCH_VALUE\" value by \"$ETCD_WATCH_ACTION\" action"'

and from another process:

    etcdctl set /foo-service/container3 localhost:2222

#### Test and set

having a watch

    etcdctl exec-watch --recursive /foo-service -- sh -c 'echo "\"$ETCD_WATCH_KEY\" key was updated to \"$ETCD_WATCH_VALUE\" value by \"$ETCD_WATCH_ACTION\" action"'

have another process do this twice

    etcdctl set /foo-service/container3 "localhost:3333" --swap-with-value "localhost:2222"
    etcdctl set /foo-service/container3 "localhost:3333" --swap-with-value "localhost:2222"

The first time the value is updated from "localhost:2222" to "localhost:3333", and a watch event is triggered, but the 2nd time, the value is already "localhost:3333", so the `swap-with-value` does not check, an error is returned to the client, and no watch event is triggered.


## run `etcd` in discovery mode
This requires reading the docs on [discovery mode](https://coreos.com/etcd/docs/latest/clustering.html#etcd-discovery), or else you will waste time.

The idea is that Discovery uses an existing cluster to boostrap itself.

### Custom etcd Discovery Service

If you already have a running `etcd` cluster somewhere, then you can create a URL like so

    curl -X PUT https://myetcd.local/v3/keys/discovery/6c007a14875d53d9bf0ef5a6fc0257c817f0fb83/_config/size -d value=3

By setting the size key to the URL, you create a discovery URL with an expected cluster size of 3.

If you bootstrap an etcd cluster using discovery service with more than the expected number of etcd members, the extra etcd processes will fall back to being proxies by default.

The URL you will use in this case will be `https://myetcd.local/v2/keys/discovery/6c007a14875d53d9bf0ef5a6fc0257c817f0fb83` and the etcd members will use the `https://myetcd.local/v2/keys/discovery/6c007a14875d53d9bf0ef5a6fc0257c817f0fb83` directory for registration as they start.

This **is** important.


### Public etcd Discovery Service

f you do not have access to an existing cluster, you can use the public discovery service hosted at discovery.etcd.io. You can create a private discovery URL using the "new" endpoint like so:

    $ curl https://discovery.etcd.io/new?size=3
    https://discovery.etcd.io/3e86b59982e49066c5d813af1c2e2579cbf573de

This will create the cluster with an initial expected size of 3 members. If you do not specify a size, a default of 3 will be used.

If you bootstrap an etcd cluster using discovery service with more than the expected number of etcd members, the extra etcd processes will fall back to being proxies by default.

### Resizing a cluster
All this is covered in detail in the [Runtime Reconfiguration docs](https://coreos.com/etcd/docs/latest/runtime-configuration.html). Please also read the doc they suggest to read (the [Design behind Runtime Reconfiguration](https://coreos.com/etcd/docs/latest/runtime-configuration.html))

Basically,

1. inform cluster of new configuration
2. start new member (need to specify the correct `initial-cluster` and set `initial-cluster-state` to `existing`)

#### Example - add a member (static mode)

Let's do this with the static configuration we currently have (the one defined in the Procfile).

As in [Brandon Philips' `etcd2` demo](https://www.youtube.com/watch?v=z6tjawXZ71E) on Youtube, I have prepared a second `Procfile` in the `newmember` directory. This `Procfile` runs a fourth instance of `etcd` as a member of the cluster.

First make sure the cluster is still running, and inform it of its new member (name and uri of new member):

    etcdctl member list
    etcdctl member add infra4 http://127.0.0.1:7004  # this is the peer port, not the client (4004)

And then run the `newmember/Procfile`

    cd newmember
    goreman start

Looking at the logs, we can see the new member, and

    etcdctl member list

shows it.

#### Example - removing a member

This is simpler

    etcdctl member list
    etcdctl member remove <UUID-of-fourth-member>

and looking at the logs and member list, we can see that the new member is gone. Furthermore, the `etcd` process of the fourth member automatically exits.

Now the memberid is in the permanent removal list. If we wanted to reinstantiate it we should remove its `data-dir`.


### Notes
#### on protection

If we were to launch another instance that pretented to be a member (configured like the fourth member above), the cluster would reject it because it was not declared in the first place.

#### on security

There also exists trusted clustered (ssl, authentication). The docs are on the main `etcd` doc page:

- [Authentication Guide](https://coreos.com/etcd/docs/latest/authentication.html)
- [Auth & Security API](https://coreos.com/etcd/docs/latest/auth_api.html)
- [Security Model](https://coreos.com/etcd/docs/latest/security.html)

#### on disaster recovery

This is also covered in the [Administration Guide](https://coreos.com/etcd/docs/latest/admin_guide.html)

### on `data-dir`
Here and there in the docs, you will see `data-dir` mentionned. Although it is critical, you won't easily find in the docs where it is located. I wasted a nice amount of time on this, and it is very simple.

#### Default location
As in our previous examples, where we launched `etcd` in native mode locally without specifying `data-dir`, the `data-dir` is by default in the current directory

        ~/D/T/c/etcd ❮❮❮ tree infra3.etcd
        infra3.etcd
        └── member
            ├── snap
            └── wal
                └── 0000000000000000-0000000000000000.wal

#### in-CoreOS (systemd) default location of `data-dir`

In a few minutes, we will launch CoreOS. `etcd` will be launched automatically by `systemd`, then the default location of `data-dir` will be `/var/lib/etcd2`.



# 2. CoreOS
You can `cd` out to the repo root.

## Get a new `etcd` cluster id

We need to prepare a new `etcd` cluster id before we boot the machines:

    $ http https://discovery.etcd.io/new\?size\=3
    https://discovery.etcd.io/813b7818ebd757986ec5d5e452714b95

Please copy the `user-data.sample` to `user-data` and add the following line where indicated in it

    discovery: https://discovery.etcd.io/813b7818ebd757986ec5d5e452714b95

and finally boot the 3 instances

    vagrant up

then for the rest of this part `cd` to `demo/2.coreos_fleet`

## Prepare terminal, add files

In order to have a well behaved terminal inside coreos, from the host, set the `TERM` as follows:

    the_key=~/.vagrant.d/insecure_private_key
    for port in 2222 2200 2201;
    do
        infocmp | ssh -T -p $port -i $the_key core@localhost 'cat > "$TERM.info" && tic "$TERM.info"'
    done

The ports are the ones given by vagrant/virtualbox to each host's ssh server, so that we can ssh into them.

## CoreOS Tour

How to install my app, where is the package manager? Stop. By design, CoreOS is designed to be mnml lnx (thanks Brian), and run containers, have nice networking stack (flannel), nice orchestration (systemd + fleet), docker or rkt for managing containers. So nothing prevents you from installing from source or installing binaries, but there is no package manager.

In the best demos I've seen, the [aim was to not log in](https://www.youtube.com/watch?v=9W-ngbpBSMM)!

Where are the configuration files? The `systemd` config files are located as usual in `/run/systemd/system` and `/etc/systemd/system`

The guys at CoreOS use `cloud-init`. The config files are in `/usr/share/oem/cloud-config.yml`, and there are good docs that explain the [`cloud-init` mechanism and config files for CoreOS](https://coreos.com/os/docs/latest/cloud-config.html).


Ping each other

    ping 172.17.8.101
    ping 172.17.8.102
    ping 172.17.8.103

Check systemd status

    systemctl status

Check etcd, fleetd, flanneld are running

    systemctl status etcd2.service
    systemctl status fleet.service
    systemctl status flanneld.service
    systemctl status docker.service    # is not running
    systemctl status docker-tcp.socket # is running


etcd is accessible from `127.0.0.1` or the `172.17.8.101`, on port 2379 by default, so we could do exactly the same demo than before on `etcd` usage.

## Booting: from disk or iPXE

CoreOS can be installed to and booted from disk. We will do this here using the [coreos-vagrant](https://github.com/coreos/coreos-vagrant/)  and in my [fork](https://github.com/bosr/coreos-vagrant/) which also contains some unit files.

CoreOS can also be booted on diskless machines, via [iPXE](https://coreos.com/os/docs/latest/booting-with-ipxe.html). Kelsey Hightower has it all explained on [coreos-ipxe-server](https://github.com/kelseyhightower/coreos-ipxe-server/blob/master/docs/getting_started.md).


## Playing with fleet

This is based on the [CoreOS tutorial](https://www.digitalocean.com/community/tutorials/an-introduction-to-coreos-system-components) from Digital Ocean.

and on [Scott Lowe's CoreOS, fleet, and Docker tutorial](http://blog.scottlowe.org/2014/08/20/coreos-continued-fleet-and-docker/)

and also on [CoreOS doc 'using the client'](https://coreos.com/fleet/docs/latest/using-the-client.html)

## copy the unit-files

First copy unit files on the fleet master

    tar -c unit-files | ssh -C -p 2222 -i $the_key core@localhost "tar -x"

Then log in to the coreos fleetmanager

    ssh -p 2222 -i $the_key core@localhost

## check that `fleet` is running

    core@core-01 ~ $ fleetctl list-machines
    MACHINE         IP              METADATA
    4f4a4289...     172.17.8.101    -
    ca541338...     172.17.8.102    -
    f91cbb3a...     172.17.8.103    -

Here are the commands that fleetctl understands:

    core@core-01 ~ $ fleetctl
    NAME:
            fleetctl - fleetctl is a command-line interface to fleet, the cluster-wide CoreOS init system.

    USAGE:
            fleetctl [global options] <command> [command options] [arguments...]

    VERSION:
            0.11.5

    COMMANDS:
            cat             Output the contents of a submitted unit
            destroy         Destroy one or more units in the cluster
            fd-forward      Proxy stdin and stdout to a unix domain socket
            help            Show a list of commands or help for one command
            journal         Print the journal of a unit in the cluster to stdout
            list-machines   Enumerate the current hosts in the cluster
            list-unit-files List the units that exist in the cluster.
            list-units      List the current state of units in the cluster
            load            Schedule one or more units in the cluster, first submitting them if necessary.
            ssh             Open interactive shell on a machine in the cluster
            start           Instruct systemd to start one or more units in the cluster, first submitting and loading if necessary.
            status          Output the status of one or more units in the cluster
            stop            Instruct systemd to stop one or more units in the cluster.
            submit          Upload one or more units to the cluster without starting them
            unload          Unschedule one or more units in the cluster.
            verify          DEPRECATED - No longer works
            version         Print the version and exit

    [... skip global options and etcd endpoints ...]

    Run "fleetctl help <command>" for more details on a specific command.

## Takeaway

The basic takeaway is that `fleet` makes a distinction between those unit-file states

- submitted: the `unit-file` is read from filesystem and registered with `fleet` in `etcd`
- loaded: the `unit-file` is copied onto the targeted worker (we can set constraints), read to be started
- running: you may have guessed


## examine the unit-files

    tree unit-files
    unit-files
    ├── instances
    │   ├── nginx-discovery@5555.service -> ../templates/nginx-discovery@.service
    │   ├── nginx-discovery@6666.service -> ../templates/nginx-discovery@.service
    │   ├── nginx-discovery@7777.service -> ../templates/nginx-discovery@.service
    │   ├── nginx@5555.service -> ../templates/nginx@.service
    │   ├── nginx@6666.service -> ../templates/nginx@.service
    │   └── nginx@7777.service -> ../templates/nginx@.service
    ├── static
    │   └── hello.service
    └── templates
        ├── nginx-discovery@.service
        └── nginx@.service


### Basic static unit-file

This basically echoes 'Hello World' forever, every second:

    $ cat unit-files/static/hello.service

    [Unit]
    Description=My Service
    After=docker.service

    [Service]
    TimeoutStartSec=0
    KillMode=none
    ExecStartPre=-/usr/bin/docker kill hello
    ExecStartPre=-/usr/bin/docker rm hello
    ExecStartPre=/usr/bin/docker pull busybox
    ExecStart=/usr/bin/docker run --name hello busybox /bin/sh -c "while true; do echo Hello World; sleep 1; done"
    ExecStop=/usr/bin/docker stop hello

### Other unit-files are more complex

We'll check them out later.


## Running hello-world

    core@core-01 ~ $ fleetctl load unit-files/static/hello.service
    Unit hello.service inactive
    Unit hello.service loaded on 4f4a4289.../172.17.8.101

fleetctl has

1. submitted the file (registered it in `etcd`): we could have done separately with `fleetctl submit unit-files/static/hello.service`
2. then given the constraints (none in this simple example) it has elected the worker `172.17.8.101`
3. then it has copied the unit-file onto `172.17.8.101` (it could have been elsewhere), but not yet started it

And now let's check all this.

Is the unit-file submitted?

    core@core-01 ~ $ fleetctl list-unit-files
    UNIT            HASH    DSTATE  STATE   TARGET
    hello.service   4286752 loaded  loaded  4f4a4289.../172.17.8.101

Yes. Is the unit loaded onto the target worker?

    core@core-01 ~ $ fleetctl list-units
    UNIT            MACHINE                         ACTIVE          SUB
    hello.service   4f4a4289.../172.17.8.101        inactive        dead

Yes. Concretely, what does it mean for the `unit` to be loaded?

    core@core-01 ~ $ ls -al /run/systemd/system
    total 12
    drwxr-xr-x  5 root root 180 Nov  1 21:50 .
    drwxr-xr-x 16 root root 380 Nov  1 21:50 ..
    -rw-r--r--  1 root root  71 Nov  1 11:15 coreos-cloudinit-vagrant-mkdir.service
    -rw-r--r--  1 root root 146 Nov  1 11:15 coreos-cloudinit-vagrant-user.path
    -rw-r--r--  1 root root  17 Nov  1 11:16 docker-3a0c96192c64e51ffab498a748d9468e52cece3c36fb1a8980196e585118aa21.scope
    drwxr-xr-x  2 root root 160 Nov  1 11:16 docker-3a0c96192c64e51ffab498a748d9468e52cece3c36fb1a8980196e585118aa21.scope.d
    drwxr-xr-x  2 root root  60 Nov  1 11:15 etcd2.service.d
    drwxr-xr-x  2 root root  60 Nov  1 11:15 fleet.service.d
    lrwxrwxrwx  1 root root  30 Nov  1 21:50 hello.service -> /run/fleet/units/hello.service

Can you see the last line? Loaded/scheduled means the unit-file is actually copied to the worker, and linked into a place that the local `systemd` can see.

And docker is still not started on `172.17.8.101`, because we haven't yet started the service.

    core@core-01 ~ $ systemctl status docker.service
    ● docker.service - Docker Application Container Engine
       Loaded: loaded (/usr/lib64/systemd/system/docker.service; disabled; vendor preset: disabled)
       Active: inactive (dead)
         Docs: http://docs.docker.com

Let's not yet start it, but continue to play with those concepts.

### Unload the unit-file

    core@core-01 ~ $ fleetctl unload hello.service
    Unit hello.service inactive

The `unit` is not listed anymore (I wrote `unit`, not `unit-file`...)

    core@core-01 ~ $ fleetctl list-units
    UNIT    MACHINE ACTIVE  SUB

But the `unit-file` is still registered with `fleetctl`, but it is not 'loaded'/scheduled on one cluster member.

    core@core-01 ~ $ fleetctl list-unit-files
    UNIT            HASH    DSTATE          STATE           TARGET
    hello.service   4286752 inactive        inactive        -

Let's check whether the `unit-file` is still in place in `/run/systemd/system`

    core@core-01 ~ $ ls -al /run/systemd/system
    total 12
    drwxr-xr-x  5 root root 160 Nov  1 21:54 .
    drwxr-xr-x 16 root root 380 Nov  1 21:54 ..
    -rw-r--r--  1 root root  71 Nov  1 11:15 coreos-cloudinit-vagrant-mkdir.service
    -rw-r--r--  1 root root 146 Nov  1 11:15 coreos-cloudinit-vagrant-user.path
    -rw-r--r--  1 root root  17 Nov  1 11:16 docker-3a0c96192c64e51ffab498a748d9468e52cece3c36fb1a8980196e585118aa21.scope
    drwxr-xr-x  2 root root 160 Nov  1 11:16 docker-3a0c96192c64e51ffab498a748d9468e52cece3c36fb1a8980196e585118aa21.scope.d
    drwxr-xr-x  2 root root  60 Nov  1 11:15 etcd2.service.d
    drwxr-xr-x  2 root root  60 Nov  1 11:15 fleet.service.d

    core@core-01 ~ $ ls -l /run/fleet/units/hello.service
    ls: cannot access /run/fleet/units/hello.service: No such file or directory

The file has been deleted from the worker.

But `fleet` still has it registered in `etcd`

    core@core-01 ~ $ fleetctl list-unit-files
    UNIT            HASH    DSTATE          STATE           TARGET
    hello.service   4286752 inactive        inactive        -

Can we remove it from `etcd`? Yes, using the `fleetctl destroy` command. Beware that destroy also destroys any running container for that `unit`.

### Let's run that service, and monitor it

For the moment the `hello.service` is not loaded (and even less started)

    core@core-01 ~ $ fleetctl status hello.service
    ● hello.service - My Service
       Loaded: loaded (/run/fleet/units/hello.service; linked-runtime; vendor preset: disabled)
       Active: inactive (dead)

    Nov 01 21:37:24 core-01 systemd[1]: Stopped My Service.
    Nov 01 21:54:26 core-01 systemd[1]: Stopped My Service.

Because it has been scheduled to run on `172.17.8.101`, for fun, I am going to run the following commands from another member of the cluster. I'll tell you when I come back to `172.17.8.101` (core-01).

    core@core-03 ~ $ fleetctl start hello.service
    Unit hello.service launched on 4f4a4289.../172.17.8.101

    core@core-03 ~ $ fleetctl list-units
    UNIT            MACHINE                         ACTIVE  SUB
    hello.service   4f4a4289.../172.17.8.101        active  running


In the mean time, from `172.17.8.101` (core-01), I check the status

    core@core-01 ~ $ fleetctl status hello.service
    ● hello.service - My Service
       Loaded: loaded (/run/fleet/units/hello.service; linked-runtime; vendor preset: disabled)
       Active: active (running) since Sun 2015-11-01 22:21:56 UTC; 534ms ago
      Process: 6327 ExecStartPre=/usr/bin/docker pull busybox (code=exited, status=0/SUCCESS)
      Process: 6322 ExecStartPre=/usr/bin/docker rm hello (code=exited, status=1/FAILURE)
      Process: 6267 ExecStartPre=/usr/bin/docker kill hello (code=exited, status=1/FAILURE)
     Main PID: 6350 (docker)
       Memory: 4.4M
          CPU: 49ms
       CGroup: /system.slice/hello.service
               └─6350 /usr/bin/docker run --name hello busybox /bin/sh -c while true; do echo Hello World; sleep 1; done

    Nov 01 22:21:56 core-01 docker[6327]: c51f86c28340: Verifying Checksum
    Nov 01 22:21:56 core-01 docker[6327]: c51f86c28340: Download complete
    Nov 01 22:21:56 core-01 docker[6327]: 039b63dd2cba: Verifying Checksum
    Nov 01 22:21:56 core-01 docker[6327]: 039b63dd2cba: Download complete
    Nov 01 22:21:56 core-01 docker[6327]: 039b63dd2cba: Pull complete
    Nov 01 22:21:56 core-01 docker[6327]: c51f86c28340: Pull complete
    Nov 01 22:21:56 core-01 docker[6327]: Digest: sha256:87fcdf79b696560b61905297f3be7759e01130a4befdfe2cc9ece9234bbbab6f
    Nov 01 22:21:56 core-01 docker[6327]: Status: Downloaded newer image for busybox:latest
    Nov 01 22:21:56 core-01 systemd[1]: Started My Service.
    Nov 01 22:21:56 core-01 docker[6350]: Hello World

And the service is started.

To stop the service from any cluster member

    core@core-03 ~ $ fleetctl stop hello.service
    Unit hello.service loaded on 4f4a4289.../172.17.8.101

    core@core-03 ~ $ fleetctl list-units
    UNIT            MACHINE                         ACTIVE          SUB
    hello.service   4f4a4289.../172.17.8.101        deactivating    stop

    core@core-03 ~ $ fleetctl list-units
    UNIT            MACHINE                         ACTIVE          SUB
    hello.service   4f4a4289.../172.17.8.101        inactive        dead



## More complex units

The other unit-files I have provided in the `unit-files` directory have some constraints. They also use 'templates'. This template mechanism is brought by `systemd`, not by `fleet`. `fleet` knows how to handle it of course.

We are going to use `docker` to run 3 `nginx` instances, and have a discovery service attached to each `nginx` instance. This discovery service checks the health of `nginx` and reports it in `etcd`.

Let's destroy the `hello.service` for cleaner outputs.

### Examine the unit-files

Again, here are the unit-files

    tree unit-files
    unit-files
    ├── instances
    │   ├── nginx-discovery@5555.service -> ../templates/nginx-discovery@.service
    │   ├── nginx-discovery@6666.service -> ../templates/nginx-discovery@.service
    │   ├── nginx-discovery@7777.service -> ../templates/nginx-discovery@.service
    │   ├── nginx@5555.service -> ../templates/nginx@.service
    │   ├── nginx@6666.service -> ../templates/nginx@.service
    │   └── nginx@7777.service -> ../templates/nginx@.service
    ├── static
    │   └── hello.service
    └── templates
        ├── nginx-discovery@.service
        └── nginx@.service

    3 directories, 9 files

### nginx@.service
This template file, if you take the time to read it, is pretty obvious: it abstracts the port `nginx` will listen on as `%i`. This makes it possible to reuse the template file on various ports. The idiomatic way with `systemd` to specify the port is to link the template from `nginx@5555.service` for instance.

If one starts that unit, one has nginx listening on port 5555.

If one schedules that unit with `fleet`, one has `nginx` listening on port 5555 somewhere in the cluster.

    [Unit]
    Description=nginx web server service on port %i

    # Requirements
    Requires=etcd2.service
    Requires=docker.service
    Requires=nginx-discovery@%i.service

    # Dependency ordering
    After=etcd2.service
    After=docker.service
    Before=nginx-discovery@%i.service

    [Service]
    # Let processes take awhile to start up (for first run Docker containers)
    TimeoutStartSec=0

    # Change killmode from "control-group" to "none" to let Docker remove
    # work correctly.
    KillMode=none

    # Get CoreOS environmental variables
    EnvironmentFile=/etc/environment

    # Pre-start and Start
    ## Directives with "=-" are allowed to fail without consequence
    ExecStartPre=-/usr/bin/docker kill nginx.%i
    ExecStartPre=-/usr/bin/docker rm nginx.%i
    ExecStartPre=/usr/bin/docker pull nginx
    ExecStart=/usr/bin/docker run --name nginx.%i -p ${COREOS_PUBLIC_IPV4}:%i:80 nginx

    # Stop
    ExecStop=/usr/bin/docker stop nginx.%i

    [X-Fleet]
    # Don't schedule on the same machine as other nginx instances
    Conflicts=nginx@*.service

In the Requirements and Dependency ordering, the ordering constraints are explicited. They are pretty obvious, not very interesting here, except that it requires `etcd2`, not `etcd` (which corresponds to the old 0.4.x version).

The last part of the unit-file indicates the scheduling constraint: we want `fleet` to avoid running two `nginx` instances on the same host.

I wrote that the `nginx` will be running 'somewhere in the cluster', but where exactly. That's the reason for the `nginx-discovery` service.

### nginx-discovery@.service

This template also abstracts the port and also declares a scheduling constraint

    [Unit]
    Description=nginx web server on port %i etcd registration

    # Requirements
    Requires=etcd2.service
    Requires=nginx@%i.service

    # Dependency ordering and binding
    After=etcd2.service
    After=nginx@%i.service
    BindsTo=nginx@%i.service

    [Service]

    # Get CoreOS environmental variables
    EnvironmentFile=/etc/environment

    # Start
    ## Test whether service is accessible and then register useful information
    ExecStart=/bin/bash -c '\
      while true; do \
        curl -f ${COREOS_PUBLIC_IPV4}:%i; \
        if [ $? -eq 0 ]; then \
          etcdctl set /services/nginx/${COREOS_PUBLIC_IPV4} \'{"host": "%H", "ipv4_addr": ${COREOS_PUBLIC_IPV4}, "port": %i}\' --ttl 30; \
        else \
          etcdctl rm /services/nginx/${COREOS_PUBLIC_IPV4}; \
        fi; \
        sleep 20; \
      done'

    # Stop
    ExecStop=/usr/bin/etcdctl rm /services/nginx/${COREOS_PUBLIC_IPV4}

    [X-Fleet]
    # Schedule on the same machine as the associated nginx service
    MachineOf=nginx@%i.service

The Requirements and Dependency ordering and binding are explicited. The `BindsTo` means that this unit follows the behavior of `nginx@%i.service`: whenever `nginx` is restarted/stopped/started, this `nginx-discovery@%i.service` does the same. This is enforced by `systemd`, not `fleet`.

The last part explicitates the scheduling constraint that the discovery service (once the template is interpolated) must run on the same machine as the (interpolated) nginx instance. Concretely, we ask `fleet` to schedule `nginx@5555.service` and `nginx-discovery@5555.service` on the same host. They will run in different containers though.

What does this discovery service do?

1. curl the nginx instance
2. if good, report with a ttl of 30 sec to `etcd` (`/services/nginx/172.17.8.x`) that the `nginx` instance is healthy and available at the right ip.
3. else, remove the `nginx` instance from `/services/nginx/172.17.8.x`) in `etcd`.

Please note the timing: sleep for 20 sec, set the key with ttl 30 sec. In case of failure, that service registration will disappear within 30 sec maximum.


### DJ, spin that sh\*t

From the coreos fleetmanager:

    fleetctl start unit-files/instances/*

1. The unit-files are going to be registered into `etcd` for `fleet` to use them
2. `fleet` will solve the scheduling constraints (find targets for units) and copy the unit-files on them
3. `fleet` will have `systemd` start those units in the right order.

Let's check this

    fleetctl list-units

    core@core-01 ~ $ fleetctl start unit-files/instances/*
    Unit nginx-discovery@5555.service inactive
    Unit nginx-discovery@6666.service inactive
    Unit nginx-discovery@7777.service inactive
    Unit nginx@5555.service inactive
    Unit nginx@6666.service inactive
    Unit nginx@7777.service inactive
    Unit nginx@5555.service launched on e2d9ac47.../172.17.8.103
    Unit nginx@6666.service launched on 605c0ca8.../172.17.8.101
    Unit nginx-discovery@6666.service launched on 605c0ca8.../172.17.8.101
    Unit nginx@7777.service launched on 83ebb758.../172.17.8.102
    Unit nginx-discovery@7777.service launched on 83ebb758.../172.17.8.102
    Unit nginx-discovery@5555.service launched on e2d9ac47.../172.17.8.103

Now the units should be running on the cluster

    fleetctl list-units

    etcdctl watch /services/nginx/ --recursive --forever

From a separate, let's stop one of the units

    fleetctl stop nginx@5555.service

And then restart it

    fleetctl start nginx@5555.service


### DJ, more flexibility please

Destroy all units and start over in a more flexible manner at runtime.

    core@core-01 ~ $ fleetctl destroy nginx-discovery@5555.service nginx@5555.service \
    nginx-discovery@6666.service nginx@6666.service \
    nginx-discovery@7777.service nginx@7777.service

    Destroyed nginx-discovery@5555.service
    Destroyed nginx@5555.service
    Destroyed nginx-discovery@6666.service
    Destroyed nginx@6666.service
    Destroyed nginx-discovery@7777.service
    Destroyed nginx@7777.service

We can submit only the units templates and load/start an instanciated unit on any port we want (I'll pick `4444`):

    $ fleetctl submit unit-files/templates/*
    $ fleetctl start nginx@4444.service nginx-discovery@4444.service

    Unit nginx-discovery@4444.service loaded on 605c0ca8.../172.17.8.101
    Unit nginx@4444.service loaded on 605c0ca8.../172.17.8.101
    Unit nginx-discovery@4444.service launched on 605c0ca8.../172.17.8.101
    Unit nginx@4444.service launched on 605c0ca8.../172.17.8.101

And we'll see that unit popping out somewhere in the cluster (if the watch is still active, we'll see that unit reporting that it is healthy).

    [set] /services/nginx/172.17.8.101
    {"host": "core-01", "ipv4_addr": 172.17.8.101, "port": 4444}

Good!

## One last thing

We can also use `fleet` to log into a worker

    fleetctl ssh 605c0ca8

or

    fleetctl ssh nginx@6666.service

in order to log in the worker that runs that service.

We can also do that from outside CoreOS, but this requires doing some ssh configuration. Due to lack of time, I engage the reader to read the [nice docs on that subject](https://coreos.com/fleet/docs/latest/using-the-client.html#ssh-dynamically-to-host).

# 3. Runtime cluster reconfiguration

For this part, please `cd` to `demo/3.coreos_dynamic`.

This part of the demo is based on [How to use confd and etcd to dynamically reconfigure services in CoreOS](https://www.digitalocean.com/community/tutorials/how-to-use-confd-and-etcd-to-dynamically-reconfigure-services-in-coreos), the 7th part of [Digital Ocean's CoreOS tutorial series](https://www.digitalocean.com/community/tutorial_series/getting-started-with-coreos-2).

I have already built the Docker image and named it `nginx-lb` on [my Docker Hub repository](), so we'll go fast and demonstrate a small architecture for serving web pages:

- two `nginx` webservers that serve a webpage (a static one, but whatever).
- one `nginx` service to load balance the requests

The actual tutorial series uses `apache` webservers to serve the pages, but I took the liberty to change this with `nginx` so as to minimize the web payload during the demo.

## copy the unit-files onto the fleet master

The unit-files are in the `unit-files3` directory in `demo/3.coreos_dynamic`. Before we copy them onto the fleet master
- we should destroy the units currently registered/running in `fleet`
- delete the `unit-files` directory on the fleet master.

When done,

    tar -c unit-files3 | ssh -C -p 2222 -i $the_key core@localhost "tar -x"


## unit-files
### webservers template

The webservers unit-files are **almost exactly** the same as before. They should have `{COREOS_PRIVATE_IPV4}` instead of `{COREOS_PUBLIC_IPV4}` in order to simulate a real scenario where the workers are not visible from the outside world, but we here have a flat network in Vagrant for this demo.

### discovery service template

The unit-files are **almost exactly** the same as before, but instead of passing a JSON object, we are setting a simple IP address + port combination. This way, we can read this value directly to find the connection information necessary to get to this service.

### nginx-lb unit-file

No more time sorry. TBContinued

