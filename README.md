# Home Lab Infrastructure Build

## Project Overview

I've been working on building out a home lab using retired computers donated by my workplace. The original idea was to put together an environment where I could eventually run an Active Directory domain and practice penetration testing. Rather than buying new hardware, I wanted to see what I could build with the equipment I already had.

The project has involved quite a bit more than connecting a few computers to a switch. I've been working through network segmentation, virtualized routing, firewall configurations, Windows imaging, and troubleshooting some of the donated hardware.

My goal is to have an environment that I can expand over time, experiment with, and use to practice different aspects of information security. I also wanted to build the network in a way that would let me isolate systems and control how they communicate rather than having everything on the same subnet.

## Project Goals

The initial objectives for this project are:

- Build a segmented home network using VLANs.
- Configure pfSense to handle routing and firewall rules between network segments.
- Establish separate networks for management, trusted workstations, IoT devices, and lab systems.
- Create reusable Windows images to simplify deployment and rebuilding.
- Establish a reliable management and recovery path for network equipment.
- Prepare the environment for a Windows Server 2025 and Active Directory deployment.
- Build a foundation for future penetration testing and security research.

## Hardware Inventory

Most of the equipment came from retired workplace hardware. The current setup consists of:

| Role | Hardware | RAM | Storage | OS |
|---|---|---:|---:|---|
| Router | AMT system, Intel i3 | 8 GB | 500 GB | pfSense (virtualized) |
| Node 01 | Dell Precision 3440, Intel i3 | 8 GB | 500 GB | Windows 10 |
| Node 02 | AMT system, Intel i3 | 8 GB | 500 GB | Windows 10 |
| Node 03 | AMT system, Intel i3 | 8 GB | 500 GB | Windows 10 |
| Node 04 | AMT system, Intel i3 | 8 GB | 500 GB | Windows 10 |
| Storage Node | AMT system, Intel i3 | 8 GB | 3 TB | Decommissioned |
| Switch | HP V1905-24 | — | — | Managed switch |

## Network Architecture

Before getting too far into deploying the computers, I wanted to establish a network that would allow me to separate different types of devices and control how they communicate.

I'm using an HP V1905-24 managed switch and pfSense running in a virtualized environment. The network is divided into several VLANs:

| VLAN ID | Name | Gateway | Subnet | Purpose |
|---|---|---|---|---|
| 1 | Default / MGMT | 192.168.1.1 | /24 | Emergency access |
| 10 | Management | 192.168.10.1 | /24 | Infrastructure management |
| 20 | Trusted | 192.168.20.1 | /24 | Primary workstations |
| 30 | IoT | 192.168.30.1 | /24 | Lower-trust devices |
| 40 | DMZ | 192.168.40.1 | /24 | Lab systems and external services |

The idea is to keep the different systems separated rather than having everything connected to the same network. As I build out the lab, this will give me more control over which devices can communicate with each other.

### Switch Configuration

The HP switch is configured with the following port assignments:

| Ports | Configuration |
|---|---|
| 1 | Trunk to pfSense, tagged VLANs 10, 20, 30, and 40 |
| 2–6 | Access VLAN 20 (Trusted) |
| 7–11 | Access VLAN 30 (IoT) |
| 13–17 | Access VLAN 40 (DMZ) |
| 25 | Access VLAN 1 (Emergency management access) |
| Unused | Administratively disabled |

The unused ports are disabled, and the other ports are assigned to specific VLANs based on their intended use.

### Management and Recovery Access

One of the things I learned early on was the importance of having a way to access network equipment if something goes wrong with the network configuration.

I originally moved my admin station into a different VLAN before configuring the router to handle traffic between the segments. As soon as I made the change, I lost access to the network equipment.

To avoid repeating that mistake, I set aside Port 25 on my HP switch for access through VLAN 1. This gives me a recovery path if I break something while working on the other VLANs.

This is not a true out-of-band management setup since it still depends on the switch and its configuration, but it does provide a separate access path from the VLANs I'm actively experimenting with.

## Deployment and Configuration

### Virtualization Host

pfSense is running in a VirtualBox virtualized environment on a Linux Mint (v7.0.0) host. The host firewall also needs to account for traffic moving between the virtualized environment and the physical network.

I added targeted UFW rules on the Linux Mint host to allow traffic from the lab subnets:

- 192.168.10.0/24
- 192.168.20.0/24
- 192.168.30.0/24
- 192.168.40.0/24

This was necessary to prevent the host firewall from interfering with traffic going to the virtualized environment without having to disable the firewall entirely.

### Windows Imaging

I wanted to avoid having to manually install and configure Windows on every computer. To make deployment easier, I put together a repeatable imaging process using Clonezilla.

The process looks like this:

1. Perform a fresh Windows installation.
2. Install the necessary drivers and Chrome.
3. Run Sysprep using `sysprep /generalize /oobe /shutdown`.
4. Boot into Clonezilla and capture the installation.
5. Store the compressed image on an external SSD.
6. Restore the image to the target computer using Clonezilla.

I initially tried using an image from the Dell Precision on the AMT machines, but ran into boot issues. After troubleshooting UEFI, Legacy BIOS, CSM, SATA modes, and attempting bootloader repairs with `bcdboot`, I decided to maintain separate images for the two hardware configurations.

This made more sense than continuing to troubleshoot compatibility issues between different systems.

Sysprep generalizes the Windows installation by removing system-specific information and preparing it for deployment to another computer. When the restored image is first booted, Windows goes through the out-of-box experience and generates new machine-specific information.

I stored the images on an external SSD so I could reuse them when deploying Windows to the other computers.

Before restoring an image, I also used `diskpart clean` to remove existing partitions, including any leftover organizational partitions from the previous installations.

Having separate images for the Dell and AMT systems should make it easier to rebuild the machines if needed.

### Endpoint Firewall Configuration

I also ran into issues with Windows Defender Firewall blocking ICMP requests from other VLANs.

Rather than disabling the firewall, I created custom inbound rules allowing ICMPv4 traffic from the Management and Trusted subnets.

This allows me to test connectivity between the systems while keeping the endpoint firewalls enabled.

## Challenges and Troubleshooting

### Getting the Computers to Boot

One of the bigger issues I ran into was trying to deploy a Windows image from the Dell Precision to the AMT machines. The image worked fine on the Dell, but the other machines wouldn't boot from it.

I spent quite a bit of time troubleshooting this, going through UEFI and Legacy BIOS settings, CSM, SATA modes, and trying to repair the bootloader with `bcdboot`.

Eventually, I decided that trying to get the same image working across different hardware was probably not the best approach. Instead, I created an image specifically for the AMT machines.

It was a good lesson in how much hardware differences can complicate something that seems like it should be relatively simple. Having a working image doesn't necessarily mean it's going to work on every machine you want to deploy it to.

### Troubleshooting a Problem Computer

One of the donated computers gave me quite a bit of trouble. It would turn on, flash a red LED near the motherboard power connection, and shut itself off shortly afterward.

I initially suspected a RAM issue, so I reseated the modules and tried again. That didn't change anything. After taking a closer look, I noticed a considerable amount of dust between the fan and heatsink.

Cleaning that out got the computer to start up, but it still wasn't stable enough to use.

At this point, I suspect the power supply is failing. I haven't ruled out other hardware issues, but the instability continued even after cleaning the system and checking the RAM.

I was particularly disappointed to lose this machine because it had a 3 TB drive. I originally planned to use it as a database server or for other network storage purposes. Having a dedicated storage system would have been useful as I started adding more services to the lab.

For now, I've decommissioned the machine rather than spend more time trying to get it stable. I may still repurpose the hard drives for another project down the road. I've considered building a NAS, which would give me a way to put that storage capacity to use without relying on the original computer.

## Current Status and Next Steps

The basic infrastructure is now in place. The network is segmented into separate VLANs, the switch is configured to handle the different network segments, and pfSense is handling routing between them.

I've also established a recovery path for network management, created separate Windows images for the different hardware configurations, and configured the host and endpoint firewalls to allow the necessary lab traffic.

There are still things I want to improve as I continue building out the lab, but the foundation is there.

The next step is getting Windows Server 2025 installed and setting up an Active Directory environment. From there, I'll have a foundation to start building out the lab I originally had in mind, including additional systems and the ability to practice penetration testing in an environment I control.

I'll also need to revisit the operating systems currently running on the workstations. These machines are currently using Windows 10, which reached the end of standard support in October 2025. I'll need to account for that as I move toward a more permanent lab configuration.

Longer term, I'd also like to revisit the storage side of things. If I can repurpose the existing drives into a NAS, that could become another part of the lab and provide centralized storage for future projects.

I'll continue updating this repository as the lab develops.