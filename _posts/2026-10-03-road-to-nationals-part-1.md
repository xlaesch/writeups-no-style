---
layout: post
title: "Road to Nationals, Part 1: Building Our Practice Lab"
date: 2026-10-03 00:00:00 -0400
author: Alex Schneider
categories: [road-to-nationals]
tags: [ccdc, blue-team, windows, proxmox, homelab]
series: Road to Nationals
series_part: 1
series_url: /road-to-nationals.html
permalink: /road-to-nationals-part-1.html
description: Building Cornell Cyber's CCDC practice lab with GOAD-Light, Proxmox, and a scoring engine.
---

## Starting out

I'm Alex, Cornell Cyber's president for Fall 2026, taking over from Raman. This year, I want us to publish useful tools and get more involved in vulnerability research. I also want us to make it to CCDC nationals for the first time, though we have to qualify for regionals first.

We're also trying to get Cornell project team status. The emphasis on project-based work has made CCDC a harder sell. I find that frustrating given how much work goes into preparing for it, including building this lab.

We bought our first server for Cybear Day 2026, our first public event. We'd planned our own red-vs-blue competition, but we hadn't prepared enough, and it didn't really work out. I still enjoyed seeing a group of students spend the afternoon together because they were interested in security.

The server barely got used during the event, so afterward we decided to turn it into a practice environment. Members could set up the services they'd eventually defend and get some experience managing the infrastructure themselves.

That server and the PCs donated by CCRA make up our setup at the Radio Club. Thanks to CCRA for the machines and the Radio Club for giving us somewhere to put them.

## The Windows side

Bill, our vulnerability research lead, and I handle most of the Windows work. Most of the team prefers Linux. I do too, but someone has to look after Active Directory.

Last year, we used AI-generated hardening scripts with plenty of prompts and elaborate terminal interfaces. We hadn't tested them thoroughly. Some changes disabled compatibility features that services relied on, and we lost availability points as a result.

This year, I want to know what each change does before we deploy it. We'll run the scripts against a domain and check the services afterward. We also need to be able to undo whatever breaks.

I settled on [GOAD-Light](https://github.com/Orange-Cyberdefense/GOAD), a deliberately vulnerable Active Directory lab with three Windows machines. It's small enough for our hardware and gives us a domain with misconfigurations to fix. It also supports Proxmox, which our server was already running.

## Getting it running

### Templates and clones

Remote access was limited at first, so I used the Proxmox API to stage the installer ISOs directly on the host, then added an SSH forward for shell access.

Packer installed Windows Server 2019 into a template. Terraform then created linked clones, and Ansible configured the domains and services.

The template took about 72 minutes to build on our 1.6 GHz Xeons. Once it was ready, Terraform's plan was more encouraging:

```text
Plan: 3 to add, 0 to change, 0 to destroy.
```

The resulting lab looked like this:

| VM | Hostname | Lab IP |
| --- | --- | --- |
| DC01 | kingslanding | 192.168.56.10 |
| DC02 | winterfell | 192.168.56.11 |
| SRV02 | castelblack | 192.168.56.22 |

### Waiting for Windows

The first Ansible run failed with:

```text
No route to host
```

Provisioning had started about two minutes after the clones booted. On our hardware, Windows took roughly fifteen minutes to apply its network configuration. I needed to wait for networking and WinRM to become reachable before starting Ansible.

Later, SQL Server Management Studio sat installing for two and a half hours. After I disabled Defender's real-time protection on that lab machine, the installation finished within minutes. GOAD's configuration called for it to be disabled anyway, but I hadn't caught that before waiting on the installer.

### The lab network

I put the VMs on `vmbr1`, a bridge with no physical ports. A small container at `192.168.56.1` provides DNS, DHCP, and NAT for internet access. The vulnerable machines stay off the physical LAN.

## Checking the services

Once the domain was running, I deployed the [CCDC ScoringEngine](https://github.com/ScoringEngine/ScoringEngine) in another container. We have ten checks covering DNS, LDAP, RDP, MSSQL, WinRM, and IIS, running every 180 seconds.

Getting those checks green took some debugging too. The LDAP checks failed until I used the accounts' CN-based distinguished names in the bind configuration. Another failure came from an inline YAML mapping: an unquoted comma cut off the base DN in `DC=sevenkingdoms,DC=local`.

Reading the individual check output helped separate scoring configuration errors from problems on the Windows machines. Eventually, all ten checks passed.

I saved the working VMs with a snapshot named `working-state`. To reset a VM between sessions:

```bash
qm rollback <vmid> working-state
```

We now have a baseline to test against. In the next post, I'll run our hardening scripts, track which checks fail, and work through the changes that caused them.
