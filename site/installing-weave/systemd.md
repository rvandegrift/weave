---
title: Using Weave with Systemd
menu_order: 130
---


Having installed Weave Net as per [Installing Weave](/site/installing-weave.md), you might find it convenient to configure the
init daemon to start Weave on boot. Most recent Linux distribution releases ship with [systemd](http://www.freedesktop.org/wiki/Software/systemd/). The information below provides you with some initial guidance on getting a Weave Net service configured on a systemd-based OS.

## Single Weave Net Service Unit and Configuration

A regular service unit definition for Weave Net is shown below. This file is
normally placed in `/etc/systemd/system/weave.service`.

    [Unit]
    Description=Weave Network
    Documentation=http://docs.weave.works/weave/latest_release/
    Requires=docker.service
    After=docker.service
    [Service]
    EnvironmentFile=-/etc/sysconfig/weave
    ExecStartPre=/usr/local/bin/weave launch --no-restart $PEERS
    ExecStart=/usr/bin/docker attach weave
    ExecStop=/usr/local/bin/weave stop
    [Install]
    WantedBy=multi-user.target


To specify the addresses or names of other Weave hosts to join the network, 
create the `/etc/sysconfig/weave` environment file using the following format:

    PEERS="HOST1 HOST2 .. HOSTn"

You can also use the [`weave connect`](/site/using-weave/finding-adding-hosts-dynamically.md) command to add participating hosts dynamically.

Additionally, if you want to enable [encryption](/site/using-weave/security-untrusted-networks.md) specify a
password using `WEAVE_PASSWORD="wfvAwt7sj"` in the `/etc/sysconfig/weave` environment file, and it will get picked up by
Weave Net on launch. Recommendations for choosing a suitably strong password can be found [here](/site/using-weave/security-untrusted-networks.md).

You can now launch Weave Net using

    sudo systemctl start weave

To ensure Weave Net launches after reboot, run:

    sudo systemctl enable weave

For more information on systemd, please refer to the documentation supplied
with your distribution of Linux.

## Per-component Weave Net Service Units

The above is sufficient get weave working under systemd but it has
some shortcomings.  In particular, it doesn't permit systemd to
monitor the health of the weave processes.  To fix this each component
should be started in the foreground so systemd can monitor it.  This
requires a unit for each component.  As before these are normally
placed in `/etc/systemd/system`.

weave-router.service:

    [Unit]
    Description=Weave Router
    Documentation=http://docs.weave.works/weave/latest_release/
    Requires=docker.service
    After=docker.service
    [Service]
    EnvironmentFile=-/etc/sysconfig/weave
    ExecStart=/usr/local/bin/weave launch-router --no-restart --no-detach $PEERS
    ExecStop=/usr/local/bin/weave stop-router
    [Install]
    WantedBy=multi-user.target
PEERS=
weave-proxy.socket:

    [Unit]
    Description=Weave Proxy
    Documentation=http://docs.weave.works/weave/latest_release/
    PartOf=weave-proxy.service
    [Socket]
    ListenStream=/var/run/weave/weave.sock
    SocketMode=0640
    SocketUser=root
    SocketGroup=docker
    [Install]
    WantedBy=sockets.target

weave-proxy.service:

    [Unit]
    Description=Weave Proxy
    Documentation=http://docs.weave.works/weave/latest_release/
    Requires=weave-proxy.socket
    After=weave-router.service weave-proxy.socket
    [Service]
    EnvironmentFile=-/etc/sysconfig/weave
    ExecStart=/usr/local/bin/weave launch-proxy --no-restart --no-detach
    ExecStop=/usr/local/bin/weave stop-proxy
    [Install]
    WantedBy=multi-user.target

As before, you can customize weave's behavior through the environment
file or by supplying additional command-line arguments.

You can now launch Weave using

    sudo systemctl start weave-router
    sudo systemctl start weave-proxy

To ensure Weave Net launches after reboot, run:

    sudo systemctl enable weave-router
    sudo systemctl enable weave-proxy

## SELinux Tweaks

If your OS has SELinux enabled and you want to run Weave Net as a systemd unit,
then follow the instructions below. These instructions apply to
CentOS and RHEL as of 7.0. On Fedora 21, there is no need to do this.

Once `weave` is installed in `/usr/local/bin`, set its execution
context with the commands shown below. You will need to have the
`policycoreutils-python` package installed.

    sudo semanage fcontext -a -t unconfined_exec_t -f f /usr/local/bin/weave
    sudo restorecon /usr/local/bin/weave

**See Also**

 * [Using Weave Net](/site/using-weave.md)
 * [Getting Started Guides](http://www.weave.works/guides/)
 * [Features](/site/features.md)
 * [Troubleshooting](/site/troubleshooting.md)
