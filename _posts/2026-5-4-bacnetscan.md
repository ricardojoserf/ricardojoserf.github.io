---
layout: post
title: BACnet-scan - Tool for BACnet/IP and BACnet/SC discovery
excerpt_separator: <!--more-->
---

A command-line tool for discovering and fingerprinting BACnet devices on a network supporting BACnet/IP (UDP Who-Is probes) and BACnet/SC (TCP + TLS detection), with concurrent subnet scanning and a combined summary report.

<!--more-->


<br>


---

## Introduction

BACnet-scan was born at the CYD Hackathon hosted by Armasuisse in April 2026, a fast-paced event focused on cyber defence research where teams get a week to dig into real-world security problems.

BACnet/IP has been around for decades, but BACnet/SC (the newer TLS-based variant) is increasingly showing up in modern deployments and is not well covered by existing open-source tooling. That gap turned into this script.

The result is a command-line scanner that handles both protocol variants, works on single hosts or full subnets, and provides enough detail (vendor ID, TLS version, mTLS enforcement) to quickly characterise what is actually running on the network. Built in a couple of days using Claude, but hopefully useful beyond it!

<br>


---

## Features

- **BACnet/IP**: raw UDP Who-Is probe, no external dependencies required

- **BACnet/SC** three-step check: TCP reachability → TLS handshake → mTLS detection, plus optional WebSocket upgrade probe

- Single IP, broadcast address, CIDR subnet (`/24`, `/16`, …), or comma-separated target list

- Flexible port specification: single port, comma list, range, or a mix

- Concurrent scanning via a configurable thread pool

- Run either protocol independently or both together in one pass

- Built-in vendor ID registry lookup

<br>


---

## Installation

```
git clone https://github.com/ricardojoserf/BACnet-scan
cd BACnet-scan
python BACnet-scan.py --help
```

<br>


---

## Usage

At least one of `--port-ip` or `--port-sc` must be supplied.

- `--port-ip` only: BACnet/IP probed only

- `--port-sc` only: BACnet/SC probed only

- Both: Both probed, combined summary shown

```
python bacnet_check.py --target TARGET --port-ip PORT [--port-sc PORT] [options]
```

<br>


---

### Options

- `--target`: Single IP, CIDR subnet, or comma-separated IPs. Default: `192.168.1.255`

- `--port-ip`: UDP port(s) for BACnet/IP *(required to probe IP)*

- `--port-sc`: TCP port(s) for BACnet/SC *(required to probe SC)*

- `--timeout`: Seconds to wait per probe. Default: `2`

- `--workers`: Concurrent threads for subnet scans. Default: `30`

- `--list-vendors`: Print vendor ID registry and exit. 

- `--lookup-vendor ID`: Look up a single vendor ID and exit.

![Help message](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/bacnet/Screenshot_1.png)


<br>


---

## Examples

#### BACnet/IP scan only

![BACnet/IP scan](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/bacnet/Screenshot_2.png)

Devices that respond to the UDP Who-Is probe are listed with their object type, instance number, vendor ID, max APDU size, and segmentation support.

<br>

#### BACnet/SC scan only

![BACnet/SC scan](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/bacnet/Screenshot_3.png)

Each host is classified as one of:

- `CONFIRMED`: TCP ✓  TLS ✓  mTLS required — full BACnet/SC endpoint

- `POTENTIAL`: TCP ✓  TLS ✓  mTLS not detected at handshake level

- `NO TLS`: TCP reachable but TLS not supported

- `CLOSED`: Port unreachable or filtered

<br>

#### Both protocols

![Combined scan](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/bacnet/Screenshot_4.png)

When both `--port-ip` and `--port-sc` are supplied the tool runs both probes concurrently and prints a unified summary separating BACnet/IP and BACnet/SC results.

<br>

#### BACnet/SC detail in single host

When a single target and port are given for `--port-sc`, the tool prints a step-by-step diagnostic instead of a summary table:

```
==============================================================
   BACnet-SC capability check  172.21.10.2:47821
==============================================================

  [1/3] TCP connect
        [OK]   Port 47821 is reachable.

  [2/3] TLS handshake
        [OK]   TLS handshake succeeded.
               Version : TLSv1.3
               Cipher  : TLS_AES_256_GCM_SHA384

  [3/3] Mutual TLS (mTLS) detection
        [OK]   Server sent CERTIFICATE_REQUIRED.
               mTLS is enforced - client certificate required.

  ------------------------------------------------------------
  RESULT: [BACNET-SC CONFIRMED]
          TCP [OK]  TLS [OK]  mTLS required [OK]
          To connect you need the client certificate
          issued by the hub's CA (see hub config directory).
==============================================================
```

<br>
