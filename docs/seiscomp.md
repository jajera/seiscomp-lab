---
title: SeisComP
layout: default
nav_order: 3
---

# SeisComP
{: .no_toc }

Public gsm install, local MariaDB, GEOFON inventory, and the modules this lab enables.
{: .fs-5 .fw-300 }

## On this page
{: .no_toc .text-delta }

- TOC
{:toc}

---

## Install model

gempa’s supported path is **Linux LTS + gsm**, not a container.

`gsm setup` with **no** `--user` / `--password` uses `https://data.gempa.de/packages/Public`.

| Package          | Role                                       |
| ---------------- | ------------------------------------------ |
| `seiscomp` 7.3.1 | Core, acquisition, processing, public GUIs |
| `world-minimal`  | Maps for `scmv` / `scesv`                  |

Do not point gsm at a private gempa URL unless you have a paid repository.

Install under `/home/sysop/seiscomp`. Data under `/home/sysop/data`. Run `seiscomp` as `sysop`, not root (`--asroot` is not the lab path).

## Database

MariaDB on **127.0.0.1:3306** only.

|          |                                           |
| -------- | ----------------------------------------- |
| Database | `seiscomp`                                |
| User     | `sysop` @ `localhost`                     |
| Password | `sysop` (SeisComP default; lab only)      |
| Schema   | `/home/sysop/seiscomp/share/db/mysql.sql` |

`innodb_buffer_pool_size = 512M` is enough for this catalog size.

## Agency and streams

| Key                         | Value                     |
| --------------------------- | ------------------------- |
| `agencyID` / `datacenterID` | `LEARN`                   |
| `recordstream`              | `slink://localhost:18000` |
| `connection.server`         | `localhost/production`    |
| scmaster bind               | `127.0.0.1:18180`         |

Stations (GEOFON BH via `geofon.gfz.de:18000`): `GE.MORC`, `GE.RGN`, `GE.STU`, `GE.WLF`.

Inventory is imported from GEOFON FDSNWS station XML. `GE.BFO` was requested in the first lab and was **not** in the response; `update-config` drops station keys that have no inventory.

## Bindings

Files under `/home/sysop/seiscomp/etc/key`:

| Profile / key                   | Meaning                                                       |
| ------------------------------- | ------------------------------------------------------------- |
| `global/profile_BH`             | `detecStream=BH`                                              |
| `scautopick/profile_default`    | default picker                                                |
| `seedlink/profile_geofon`       | chain plugin → `geofon.gfz.de:18000`, selectors `BH?.D`       |
| `slarchive/profile_week`        | keep 7 days                                                   |
| `station_GE_{WLF,STU,MORC,RGN}` | `global:BH scautopick:default seedlink:geofon slarchive:week` |

Create inventory **before** station key files. Otherwise `update-config` deletes keys for missing stations.

## Enabled modules

```text
scmaster seedlink slarchive
scautopick scautoloc scamp scmag scevent
scalert fdsnws scqc scevtlog
```

`scmaster` is a kernel module and is enabled with the rest.

`fdsnws` listens on `127.0.0.1:8080` only. Do not open 8080 on the security group.

## Intentionally not installed

Public extras left off for a first lab: `sed-eew`, `scrtdd`, `elevation`, `slabs-usgs`, `dlbase` / `dltools`, `seiscomp-debug`.

Modules in the SeisComP package left disabled: `scimport`, `ql2sc`, `ew2sc`, `scimex`, `scvoice`, `scwfparam`, `screloc`, `slmon`, `access`, `diskmon`, iLoc tables.
