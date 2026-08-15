---
title: Troubleshooting
layout: default
nav_order: 7
---

# Troubleshooting
{: .no_toc }

Failures from the first build of this lab.
{: .fs-5 .fw-300 }

## On this page
{: .no_toc .text-delta }

- TOC
{:toc}

---

## SSM RunShellScript is dash

`AWS-RunShellScript` is `sh` (dash on Ubuntu), not bash. `set -o pipefail` and bash arrays fail. Use `aws ssm start-session` and a bash shell, or wrap as `echo BASE64 | base64 -d | bash`.

## gsm and apt prompt

`gsm install` without `-y` waits for confirmation. `seiscomp install-deps` calls `apt install` without `-y`. Use `gsm install -y` and wrap `apt` as in the [walkthrough]({{ site.baseurl }}/walkthrough/#5-distro-deps-and-mariadb).

## No outbound internet

The instance must have a public IPv4 (subnet map-public-ip plus `--associate-public-ip-address`, or an Elastic IP). There is no NAT. Without it, apt, gsm, SSM extras, and GEOFON all fail.

## SSM not online

The instance profile must exist **before** launch. Wait about a minute after boot for `PingStatus=Online`.

## seiscomp as root

`seiscomp` refuses root unless you pass `--asroot`. Run as `sysop`.

## update-config deletes station keys

Inventory must exist before `station_GE_*` files. Missing stations (for example `GE.BFO` not in the FDSN response) are dropped.

## xrdp drops after password

Globals `port=3389` is the listen port. `[Xorg]` must be `port=-1`. A blanket `sed` on `port=` sets Xorg to 3389; xrdp then connects to itself.

Also check `sesman.ini` `ListenAddress=0.0.0.0` and `Xwrapper` `allowed_users=anybody`.

## scheli exits immediately

`ERROR: no streams given` — set `heli.streams` in `~/seiscomp/etc/scheli.cfg`.

## notify-send does nothing

Use `/home/sysop/bin/sc-toast-event` (gdbus against xfce4-notifyd). `notify-send` often hits xdg-portal and never shows.

## RDP from Windows fails

Confirm your current public IP is the `/32` on the security group. Choose session **Xorg**. Do not RDP to a Linux hop first.

PowerShell SSM port forwarding: use `portNumber=3389,localPortNumber=13389`, not JSON.

## Elastic IP after stop

An EIP associated with a **stopped** instance still bills. Release it or keep the instance running.
