---
title: My Homelab in 2026 — How It Started and What I'm Running
date: 2026-06-03
draft: false
tags:
  - homelab
  - proxmox
  - docker
  - self-hosting
  - networking
description: How a dead router, a tax return, and too many YouTube videos turned into a full homelab stack.
---

Like a lot of people, it started with YouTube and TikTok. I kept seeing videos about homelabs, what people were running, what it could do, why they built one, and it slowly pulled me in.

For me it came down to three things.

First, security. I have a cybersecurity background, currently working toward a BS in cybersecurity, and my day job leans heavily on the sysadmin side with a security emphasis. But the work can get stagnant. Limited scope, slow technology adoption, not a lot of room to experiment. A homelab fixes that. It gives me a sandbox where I can actually try things without waiting for an approval process.

Second, learning. I wanted hands-on experience with tools and technologies I wasn't getting exposure to at work. Reading about something and actually running it are very different things.

Third, and honestly this one might be the most relatable, subscriptions. Everything is a subscription now. Cloud storage, media streaming, VPN, backup services. It adds up fast and you own nothing. I wanted to change that.

There was also a fourth reason, and it came out of nowhere. My router/modem combo decided to die around the same time I was going down the homelab rabbit hole. I called customer service and was told my device was outdated and that I needed to either rent their router or buy a compatible one. The modem would be free as long as I stayed with them, but the router was a rental. That didn't sit right with me. If I was going to replace my networking gear anyway, I figured I might as well invest in something proper instead of paying monthly for equipment I'd never own. That decision spiraled pretty quickly into everything else you're about to read.

The only thing standing between me and getting started was money. I didn't want to spend a bunch upfront before I even knew if I'd stick with it. Then tax season came through. I know I'm overpaying, don't bug me about it. I like to think of my tax return as a nice surprise once a year. Some of that money went toward the homelab, and you'll see what I mean once I introduce you to the setup and what it cost to get here.

---

## The Hardware and Why I Chose Each Piece

Once I decided I was going to replace my networking gear anyway, the question became what to replace it with. I was already familiar with TP-Link from my old router/modem combo, so I started looking into what else they made. That's when I found Omada.

Omada is TP-Link's software-defined networking lineup, and once I saw what it offered I was sold. A built-in VPN server with client-to-site support, meaning I can access my home network securely from any device anywhere, centralized management of all network devices from one interface, and the ability to expand the setup as needed. The learning curve is steep if you're new to networking. I have decent experience but had never touched SDNs before, so that was part of the appeal too.

The other thing that drove a lot of my decisions was the modem Spectrum provided. It has a 2.5 Gbps ethernet port, and once I saw that I wanted everything downstream to match. No point having a fast modem if your network gear is going to bottleneck it.

Here's what I ended up with on the networking side:

**TP-Link ER707-M2** — the router. Two 2.5 Gbps WAN ports plus six additional WAN/LAN ports. Handles routing, firewall, and the VPN server.

| Spec | Detail |
|---|---|
| WAN Ports | 2x 2.5 Gbps |
| LAN Ports | 6x 1 Gbps WAN/LAN |
| VPN | WireGuard, OpenVPN, IPSec |
| Management | Omada SDN |
| Other | SPI Firewall, Load Balancing, Lightning Protection |

**SG2210XMP-M2** — the switch. Eight 2.5 Gbps PoE+ ports and two SFP+ ports for future upgrades. The PoE+ is important, more on that in a second.

| Spec | Detail |
|---|---|
| PoE+ Ports | 8x 2.5 Gbps |
| Uplink Ports | 2x 10G SFP+ |
| PoE Budget | 240W |
| Management | Omada SDN |

**TP-Link EAP770** — WiFi 7 access point with a 2.5 Gbps uplink, managed through Omada like everything else.

| Spec | Detail |
|---|---|
| WiFi Standard | WiFi 7 (802.11be) |
| Max Speed | BE11000 Tri-band |
| Uplink | 2.5 Gbps |
| Power | PoE+ |
| Management | Omada SDN |

I'd also be doing this section a disservice if I didn't mention the device that started it all before the tax return came through. An old Acer Predator laptop, i7-7700HQ, 8 GB of RAM. Nothing special by today's standards but it was what I had. Before I bought a single piece of equipment, that laptop was my testing ground. I installed Pi-hole on it, tried out different operating systems, broke things, fixed things, and figured out what I actually wanted to build. It was messy and unorganized but that's kind of the point when you're learning.

It's still in the mix today. Once I got the Mini PC up and running Proxmox, I installed Proxmox on the laptop too and joined it to the same cluster. It now serves as a failover node. Not the most powerful machine in the stack, but it doesn't need to be. If the Mini PC goes down for maintenance or something unexpected happens, the laptop is there to keep things running.

| Spec | Detail |
|---|---|
| Model | Acer Predator Helios 300 G3-571 |
| CPU | Intel Core i7-7700HQ @ 2.80GHz (8 threads) |
| RAM | 8 GB |
| Role | Proxmox failover node |

For the server side, I wanted something small. I didn't want a full rack or a loud tower sitting somewhere. I went with a Mini PC that punches well above its weight, 48 GB of RAM, dual 2.5 Gbps NICs, enough cores to run a bunch of VMs simultaneously, and a CPU with a low enough power draw that leaving it on 24/7 doesn't hurt. It's running Proxmox as the hypervisor, which lets me spin up and tear down virtual machines without touching the host.

| Spec | Detail |
|---|---|
| Model | GMKtec K15 AI |
| CPU | Intel Core Ultra 5 125U |
| RAM | 48 GB DDR5 |
| Storage | 1 TB NVMe SSD |
| Networking | Dual 2.5 Gbps NICs |
| Other | OCuLink, USB4, HDMI 2.1 |
| Role | Proxmox hypervisor / primary compute |

I also picked up two Raspberry Pi 5s, 16 GB RAM each. Raspis are great for containerization and I wanted to learn Docker Swarm. They're small, efficient, and more than capable of running a solid number of containers. Combined with the GeeekPi PoE+ NVMe hats I grabbed for each one, they're powered directly from the switch over ethernet. No separate power bricks, no extra cables, just one ethernet cable each. That's the PoE+ I mentioned.

| Spec | Detail |
|---|---|
| Model | Raspberry Pi 5 x2 |
| CPU | Broadcom BCM2712, 4 cores @ 2.4GHz |
| RAM | 16 GB |
| Storage | SD card (NVMe upgrade planned) |
| Power | PoE+ via GeeekPi P33 hat |
| Role | Docker Swarm manager nodes |

| Spec | Detail |
|---|---|
| Model | GeeekPi P33 PoE+ NVMe Hat |
| Power | PoE+ (802.3at) |
| Storage | M.2 NVMe slot (2230/2242/2260/2280) |
| Cooling | Official Pi 5 active cooler mount |

For storage, I went with the UGREEN NASync DH4300 Plus. It was on discount, which was honestly the deciding factor. It supports up to 128 TB, has a 2.5 Gbps ethernet port, and runs its own OS with Docker support. I didn't have the budget to buy proper NAS drives right away, so I'm currently running whatever spare hard drives I had laying around the house. It works, but it's janky and I know it. Four 16 TB drives are on the list when the budget allows.

| Spec           | Detail                    |
| -------------- | ------------------------- |
| Model          | UGREEN NASync DH4300 Plus |
| Bays           | 4                         |
| Max Capacity   | 128 TB                    |
| RAM            | 8 GB LPDDR4X              |
| Networking     | 2.5 Gbps ethernet         |
| OS             | UGOS Pro                  |
| Current Drives | 4x 16 TB                  |

Honestly, if I'm being transparent, most of my decision making came back to one thing. The modem had a 2.5 Gbps port and I wanted everything to match. Future-proofing by making it uniform throughout. Everything else kind of fell into place around that.

---

## The Physical Setup (Honest Edition)

If you were expecting a clean rack setup with velcro cable ties and perfect airflow, I'm sorry to disappoint. Everything lives in my TV stand.

It's cluttered. I'll be the first to admit it. I made an attempt at cable management and got about halfway there. The cables behind the TV stand are somewhat under control, but the top shelf is a different story. It is what it is for now.

The bigger concern heading into summer is heat. Everything is packed into an enclosed TV stand with limited airflow and I genuinely don't know how the equipment is going to handle the warmer months. That's something I need to figure out before it becomes a problem.

There's also the matter of power. Right now everything is plugged into a single outlet. No UPS, no surge protection worth mentioning. One bad power fluctuation and I could be looking at fried equipment or corrupted NAS drives. A UPS is high on the list of things to address. It's not a matter of if, it's a matter of when.

Basically, the physical setup works but it's held together with good intentions and a little bit of luck. There's room for improvement and I know it.

---

## What's Actually Running on It

This is where things get interesting. The homelab isn't just sitting there looking pretty, it's actually doing things.

On the virtualization side, Proxmox is running several VMs on the Mini PC. The first is an RHEL IDM server, which handles identity management for the lab. Think centralized user accounts and authentication, the same kind of setup you'd find in an enterprise environment. A redundant IDM server is in the works so there's a failover if the first one goes down.

There are two VMs running Pi-hole with Unbound. Pi-hole handles network-wide ad blocking and Unbound sits behind it as a recursive DNS resolver, meaning DNS queries don't get forwarded to a third party like Google or Cloudflare. They resolve directly from authoritative nameservers. Two instances means if one goes down for maintenance, the other keeps the network running without skipping a beat. It's been working great.

One VM is dedicated to hosting the Omada SDN controller application. You can buy a physical Omada controller, but why spend the money when you can just spin up a VM and install the software? Same functionality, no extra hardware.

There's also a Grafana VM for monitoring. Still a work in progress, but the goal is a unified dashboard pulling metrics from everything in the lab. A VM running OpenVAS Community Edition handles vulnerability scanning, which ties back to the security side of why I built this in the first place. It scans the network for known vulnerabilities and misconfigurations so I know what needs attention.

Finally there's an Ubuntu VM that's part of a Docker Swarm cluster along with the two Raspberry Pi 5s. The Swarm is running a few things worth mentioning. STIG Manager for system hardening and security compliance, the arr stack for media self hosting (Jellyfin, Sonarr, Radarr, and Prowlarr if those mean anything to you), and Portainer as a management UI for the containers.

One more thing. There's a VM running a Ghost and Nginx Proxy Manager Docker stack that's hosting this very website. So if you're reading this, you're being served content straight from my TV stand. Make of that what you will.

---

## Where It's Going

The dream is a proper enterprise grade server rack. Clean, organized, everything mounted and labeled. That's not happening anytime soon though. I rent, so a full rack setup isn't really an option right now. Maybe one day.

In the more immediate future, the priority is a UPS. Everything is still plugged into a single outlet and it's been that way for about three months now. The 4x16 TB drives just arrived and I've already migrated the NAS. Still have zero power protection during the whole process. Not my brightest moment, but it worked out. A UPS is still happening though. The drives are in, the data is there, and one bad power event could ruin all of it. It's next on the list.

On the hardware side, I want to build out a proper 4 node Raspberry Pi cluster using one of those DeskPi rack cases. Get a small switch that fits inside it, mount the Pis properly, and finally replace the SD cards with NVMe SSDs so the OS isn't running on hardware that could fail without warning. I'd also like to get the Mini PC and NAS mounted in there and maybe pick up a second Mini PC so everything is uniform.

Longer term, I want to build a dedicated server for running a local LLM at home. Paired with OpenClaw, a free and open source autonomous AI agent that runs on top of LLMs and uses messaging platforms as its main interface, the idea is to have a self hosted AI setup that doesn't rely on any cloud services. Is it a need? Absolutely not. But it sounds like a fun project and that's reason enough.

The homelab is a living thing. It grows, it changes, and it occasionally breaks at the worst possible time. That's kind of the point. More updates to come.

---

*— d3vilsec*