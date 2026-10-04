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

I'm Alex, Cornell Cyber's president for Fall 2026, taking over from Raman. This year, I set a couple of goals for the club: I wanted us to publish blue team tools that anyone could use, and I hoped we could get some CVEs under our belt. I also wanted us to continue placing well in competitions—we won BSidesNYC last year and jumped seven spots in ISTS—including making it to CCDC nationals for the first time. We have to qualify for regionals first, though.

We're also trying to get Cornell project team status. Last year, our application was denied in favor of CSalt, a team focused on "marine energy solutions." I still have a hard time understanding that decision, given that we're Cornell's only cybersecurity organization. The emphasis on project-based work has made CCDC a harder sell. I find that frustrating given how much work goes into preparing for it, including building this lab.

We bought our first server for Cybear Day 2026, our first public event. We'd planned our own red-vs-blue (RvB) competition, but we hadn't prepared enough or recruited enough participants, so it didn't really work out. I will say, I still enjoyed seeing a group of students spend the afternoon together because they were interested in security.

The server barely got used during the event, so afterward we decided to turn it into a practice environment. Members could set up the services they'd eventually defend and get some experience managing the infrastructure themselves.

That server and the PCs donated by [CCRA](https://cornellcomputerreuse.org/) make up our entire infrastructure setup at the Radio Club tower. Thanks to CCRA for the PCs and to the Radio Club for giving us somewhere to put them.

[IMAGE: the racks at the Radio Club]

## The Windows Side

Bill, our vulnerability research and CTF lead, and I handle all of the Windows work. Most of the team works on Linux. I'd prefer working on Linux as well—everything feels so much more intuitive—but someone has to handle Active Directory (AD), and there weren't many people willing to touch Windows at all.

Last year, I barely had any experience hardening Windows, especially in the rhythm of an RvB competition. My experience centered on some red teaming I'd done at a past job, which I really enjoyed. At our last CCDC, we made the mistake of using heavily AI-generated hardening scripts with cool interfaces and stuff. We barely tested them, and they ended up shooting us in the foot. Some changes disabled compatibility features that services relied on, and we lost availability points as a result. While I don't think using AI to write scripts is necessarily wrong, a player should be deeply familiar with how their scripts work and should test them thoroughly in a running lab environment.

With that in mind, this year I want to know what each change does before we deploy it. We'll run the scripts against a domain and check the services afterward. We also need to be able to undo whatever breaks.

I decided on [GOAD-Light](https://github.com/Orange-Cyberdefense/GOAD), a vulnerable Active Directory lab with three Windows machines. It's kind of similar to what we see in a real CCDC environment, except without the command-and-control implants (C2s) and registry madness. It's small enough for our hardware and gives us a domain with misconfigurations to fix. It also supports Proxmox, which our server was already running.

## Running It

Remote access was limited at first, so I used the Proxmox API to stage the installer ISOs directly on the host, then added an SSH forward for shell access.

Packer installed Windows Server 2019 into a template. Terraform then created linked clones, and Ansible configured the domains and services. To be honest, I was expecting a pretty painful process of fixing every Ansible step. A previous attempt to run GOAD on my own machine had been very frustrating.

The template took about 72 minutes to build on our 1.6 GHz Xeons. Once it was ready, Terraform gave me this plan:

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

It wasn't all sunshine and rainbows, so I'll sprinkle in some of the issues I faced. The setup was far more involved than this shows, but I'd rather not bore you with every detail. The first Ansible run failed with:

```text
No route to host
```

Provisioning had started about two minutes after the clones booted. On our hardware, Windows took roughly fifteen minutes to apply its network configuration. I needed to wait for networking and WinRM to become reachable before starting Ansible.

Later, SQL Server Management Studio sat installing for close to five hours. I wasn't sure whether it was just our really slow hardware or something else interfering with the installation. I realized Defender's real-time protection was scanning every new file on that lab machine, leading to crazy overhead. Once I disabled it, the installation finished within minutes. GOAD's configuration called for it to be disabled anyway, but I hadn't caught that before waiting on the installer.

I also wanted to follow proper homelabbing practices, so I put the VMs on `vmbr1`, a bridge with no physical ports. A small container at `192.168.56.1` provides DNS, DHCP, and NAT for internet access. The vulnerable machines stay off the physical LAN.

[![Proxmox dashboard listing the GOAD Windows VMs, scoring container, and Windows Server template.]({{ '/assets/images/road-to-nationals/proxmox-dashboard.png' | relative_url }})]({{ '/assets/images/road-to-nationals/proxmox-dashboard.png' | relative_url }})

*The lab VMs and scoring container in Proxmox.*

## Checking the Services

Once the domain was fully up and running, I wanted to make the lab feel more like a competition environment, so I looked into scoring engines. I know most RvB competitions use Quotient for scoring, but I found another option that seemed more actively maintained: [CCDC ScoringEngine](https://github.com/ScoringEngine/ScoringEngine). I deployed it in another container. We have ten checks covering DNS, LDAP, RDP, MSSQL, WinRM, IIS (HTTP), and ICMP, running every 180 seconds.

Getting those checks green took some debugging too. The LDAP checks failed until I used the accounts' CN-based distinguished names in the bind configuration. Another failure came from an inline YAML mapping: an unquoted comma cut off the base DN in `DC=sevenkingdoms,DC=local`.

Reading the individual check output helped separate scoring configuration errors from problems on the Windows machines. Eventually, all ten checks passed.

[![CCDC ScoringEngine overview showing the GOAD Lab with ten services up, zero down, and a green check for every service.]({{ '/assets/images/road-to-nationals/scoring-overview.png' | relative_url }})]({{ '/assets/images/road-to-nationals/scoring-overview.png' | relative_url }})

*All ten checks passing in the scoring dashboard.*



I saved the working VMs with a snapshot named `working-state`. To reset a VM between sessions:

```bash
qm rollback <vmid> working-state
```

We now have a baseline to test our scripts against. In the next post, I'll discuss what went into creating our Windows hardening scripts.
