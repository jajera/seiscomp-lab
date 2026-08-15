---
title: Overview
layout: default
nav_order: 1
---

<div class="conduit-hero">
  <p class="conduit-kicker">jajera · seiscomp-lab</p>
  <h1>SeisComP lab</h1>
  <p class="conduit-lede">
    A single-host SeisComP learning lab on Ubuntu 24.04 in AWS: public gsm packages,
    local MariaDB, GEOFON SeedLink, and an XFCE desktop. No containers, no commercial
    gempa modules.
  </p>
  <div class="conduit-actions">
    <a class="conduit-btn conduit-btn--primary" href="{{ site.baseurl }}/walkthrough/">Deploy the lab</a>
    <a class="conduit-btn conduit-btn--ghost" href="{{ site.baseurl }}/architecture/">See architecture</a>
  </div>
</div>

## What you build

One EC2 in a public subnet. The host pulls BH streams from GEOFON, runs automatic picking and location, stores a catalog in MariaDB, and serves FDSNWS on localhost. You reach the box with SSM. Optional RDP opens XFCE for `scconfig`, `scrttv`, `scolv`, and the other public GUIs.

This follows gempa’s supported path: **Linux LTS + gsm**, not Docker. Only the public repository is used (AGPL SeisComP).

## Read in this order

<div class="nav-grid">
  <a class="nav-card" href="{{ site.baseurl }}/architecture/">
    <strong>1. Architecture</strong>
    <span>VPC, security group, processes on the host</span>
  </a>
  <a class="nav-card" href="{{ site.baseurl }}/seiscomp/">
    <strong>2. SeisComP</strong>
    <span>gsm, MariaDB, inventory, bindings, modules</span>
  </a>
  <a class="nav-card" href="{{ site.baseurl }}/desktop/">
    <strong>3. Desktop</strong>
    <span>XFCE, xrdp, Windows RDP, toasts</span>
  </a>
  <a class="nav-card" href="{{ site.baseurl }}/walkthrough/">
    <strong>4. Deploy and destroy</strong>
    <span>Manual CLI from VPC to prove to teardown</span>
  </a>
  <a class="nav-card" href="{{ site.baseurl }}/prove/">
    <strong>5. Prove</strong>
    <span>SeedLink, FDSNWS, GUIs</span>
  </a>
  <a class="nav-card" href="{{ site.baseurl }}/troubleshooting/">
    <strong>Troubleshooting</strong>
    <span>SSM dash, xrdp, gsm prompts</span>
  </a>
  <a class="nav-card" href="{{ site.baseurl }}/reference/">
    <strong>Reference</strong>
    <span>File map and example inventory</span>
  </a>
</div>

## Scope

| In | Out |
|---|---|
| Ubuntu 24.04, SeisComP 7.3.1 public, `world-minimal` | gempa private/commercial gsm |
| MariaDB on localhost | RDS, multi-AZ |
| Four GEOFON BH stations | Publishing SeedLink to the internet |
| XFCE + xrdp | Amazon DCV |
| SSM for shell access | Inbound SSH |

{: .cost }
> `t3.xlarge` is not cheap to leave on. Stop it when you are not using the desktop. See [Walkthrough]({{ site.baseurl }}/walkthrough/#destroy).
