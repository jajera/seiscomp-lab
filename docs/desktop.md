---
title: Desktop
layout: default
nav_order: 4
---

# Desktop
{: .no_toc }

XFCE and xrdp on the same host. Windows can use Remote Desktop with no AWS CLI if the security group allows your public IP.
{: .fs-5 .fw-300 }

## On this page
{: .no_toc .text-delta }

- TOC
{:toc}

---

## Packages

`xfce4`, `xfce4-notifyd`, `xrdp`, `xorgxrdp`, `libnotify-bin`, Mesa. LightDM stays **disabled**. xrdp is the login path.

## xrdp that actually works
{: .warning }
> Do not replace every `port=` in `/etc/xrdp/xrdp.ini`. Only the **Globals** listen port should be `3389`. The **Xorg** session must stay `port=-1` so sesman starts X. Setting Xorg to `3389` makes xrdp connect to itself and drop after the password.

Also set:

- `/etc/xrdp/sesman.ini` — `ListenAddress=0.0.0.0`
- `/etc/X11/Xwrapper.config` — `allowed_users=anybody`

For Windows `mstsc` to the Elastic IP, Globals should listen on `0.0.0.0:3389`. The security group is the access control (your `/32` only).

For SSM port forwarding only, you can bind xrdp to `127.0.0.1:3389` and skip inbound 3389 on the security group.

## Password

Generate a password on the instance, set it on `sysop`, and store it in SSM Parameter Store:

`/seiscomp-lab/sysop-rdp-password` (SecureString)

Do not commit the password.

Retrieve:

```bash
export AWS_PROFILE=sandbox AWS_DEFAULT_REGION=ap-southeast-2
aws ssm get-parameter --name /seiscomp-lab/sysop-rdp-password \
  --with-decryption --query Parameter.Value --output text
```

## Connect from Windows (no AWS CLI)

1. Open Remote Desktop Connection (`mstsc`).
2. Computer: the instance Elastic IP.
3. User: `sysop`.
4. Session type: **Xorg** (if offered).
5. Accept the self-signed certificate warning.

You should get XFCE. If your public IP changes, update the security group.

## Connect with an SSM tunnel

Leave this running:

```bash
export AWS_PROFILE=sandbox AWS_DEFAULT_REGION=ap-southeast-2
aws ssm start-session \
  --target "$IID" \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["3389"],"localPortNumber":["13389"]}'
```

Then connect to `127.0.0.1:13389` as `sysop`.

Windows PowerShell: use `key=value` parameters. JSON `{"portNumber":...}` is mangled by PowerShell.

```powershell
$env:AWS_PROFILE = "sandbox"
$env:AWS_DEFAULT_REGION = "ap-southeast-2"
aws ssm start-session `
  --target $env:IID `
  --document-name AWS-StartPortForwardingSession `
  --parameters "portNumber=3389,localPortNumber=13389"
```

Linux client example: `xfreerdp /v:127.0.0.1:13389 /u:sysop /dynamic-resolution`.

## Launchers

Icons on `~/Desktop` and in the application menu under Science. Wrappers source `SEISCOMP_ROOT` and set `QT_QPA_PLATFORM=xcb`.

| Tool         | What it does                                              |
| ------------ | --------------------------------------------------------- |
| scconfig     | Modules, stations, bindings                               |
| scmv / scmvx | Map view                                                  |
| scrttv       | Real-time traces and picks                                |
| scolv        | Event analysis                                            |
| scesv        | Event summary                                             |
| scheli       | Helicorder (needs streams in `~/seiscomp/etc/scheli.cfg`) |
| scqcv        | Quality control                                           |
| scmm         | Messaging monitor                                         |

`scheli` without `heli.streams` exits with `ERROR: no streams given`. Lab config:

```text
heli.streams = GE.WLF..BHZ,GE.STU..BHZ,GE.MORC..BHZ,GE.RGN..BHZ
```

## Toasts

`scalert` calls `/home/sysop/bin/sc-toast-event`. `notify-send` often fails against xdg-portal on this desktop. The working path is **gdbus** `Notify` on the `xfce4-notifyd` session bus.

Look at the **top-right** of XFCE. Real earthquake toasts are uncommon with four stations.

Test from a terminal **on the desktop**:

```bash
/home/sysop/bin/sc-toast-event "test toast"
```
