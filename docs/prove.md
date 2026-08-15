---
title: Prove
layout: default
nav_order: 6
---

# Prove
{: .no_toc }

Checks that the stack is alive. This is not a click-through of every GUI dialog.
{: .fs-5 .fw-300 }

## On this page
{: .no_toc .text-delta }

- TOC
{:toc}

---

Run as `sysop` after `seiscomp print env` is in `.bashrc` (`sudo -iu sysop`).

## Processes

```bash
seiscomp check
seiscomp list enabled
```

Expect enabled: `scmaster`, `seedlink`, `slarchive`, `scautopick`, `scautoloc`, `scamp`, `scmag`, `scevent`, `scalert`, `fdsnws`, `scqc`, `scevtlog`.

Listeners:

| Bind              | Role                                 |
| ----------------- | ------------------------------------ |
| `127.0.0.1:18180` | scmaster                             |
| `127.0.0.1:3306`  | MariaDB                              |
| `127.0.0.1:8080`  | fdsnws                               |
| `0.0.0.0:18000`   | seedlink (SG does not allow inbound) |
| `0.0.0.0:3389`    | xrdp                                 |

## SeedLink

```bash
slinktool -Q localhost
```

Expect BHZ/BHN/BHE for `GE.MORC`, `GE.RGN`, `GE.STU`, `GE.WLF` with end times a few seconds in the past.

## FDSNWS

```bash
curl -sS http://127.0.0.1:8080/fdsnws/station/1/version
curl -sS 'http://127.0.0.1:8080/fdsnws/station/1/query?net=GE&level=station&format=text'
```

Dataselect should return HTTP 200 and miniSEED once `slarchive` has data (wait a few minutes after first start).

## GUIs

On XFCE, double-click the desktop launchers. Under Xvfb (no display), a 12 second `timeout` with exit 124 means the process stayed up.

| Tool          | Expect                                |
| ------------- | ------------------------------------- |
| scconfig      | stays up                              |
| scmv / scmvx  | map                                   |
| scrttv        | traces if SeedLink is live            |
| scolv / scesv | catalog may be empty until a location |
| scheli        | needs `~/seiscomp/etc/scheli.cfg`     |
| scqcv         | needs `scqc` running                  |
| scmm          | talks to scmaster                     |

## Desktop toast

After you are logged into XFCE:

```bash
/home/sysop/bin/sc-toast-event "test toast"
```

Look top-right. Real `scalert` toasts need an event; four stations means that can be rare.
