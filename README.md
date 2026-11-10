Home Lab Power Control

<img width="1470" height="956" alt="Screenshot 2026-11-10 at 16 06 06" src="https://github.com/user-attachments/assets/7c6db912-1b23-4867-983d-96f80f230f34" />


Secure, self-hosted power management and orchestration for a multi-device home laboratory.

A lightweight web-based control system designed to remotely power on, gracefully shut down, and monitor physical homelab infrastructure through a single interface.

Built around a Raspberry Pi, the system integrates multiple hardware management protocols while keeping remote access restricted to a private network.

Project type: Personal infrastructure engineering project
Status: Deployed and operational
Source code: Private — architecture and implementation overview only

⸻

1. Project Overview

My home laboratory consists of multiple physical systems running different operating systems and services, including virtualization, network security, monitoring, and local AI workloads.

Managing these devices individually created several challenges:

* Different hardware required different power management methods.
* Some devices supported Wake-on-LAN, while others required IPMI.
* Graceful shutdown needed to be handled differently across operating systems.
* Access to the laboratory needed to remain available even when the main firewall was powered off.
* Power operations needed safeguards to prevent accidental shutdowns or conflicting commands.

To address these challenges, I developed Home Lab Power Control, a centralized, self-hosted management interface.

The goal was not simply to create power buttons, but to build a controlled orchestration layer with authentication, device-specific operations, operational logging, and network-failure resilience.

2. System Architecture

The application runs on a Raspberry Pi 3B+ and provides a secure HTTPS dashboard accessible through Tailscale.

           Remote Administrator
                    |
              Tailscale VPN
                    |
             HTTPS Dashboard
                    |
             Raspberry Pi 3B+
                    |
          Flask / Gunicorn Backend
                    |
        Authentication & Authorization
                    |
          Power Control Orchestrator
                    |
       +------------+-------------+
       |            |             |
      IPMI          SSH        Wake-on-LAN
       |            |             |
   Supermicro    Linux Hosts    OPNsense
   BMC/IPMI      / Shutdown     WOL Relay
                                    |
                              Network Devices

The controller selects the appropriate management method for each device based on its configured capabilities.

Core Technologies

Component	Technology
Controller	Raspberry Pi 3B+
Operating system	Linux
Backend	Python, Flask, Gunicorn
Frontend	HTML, CSS, JavaScript
Remote access	Tailscale
Transport security	HTTPS / TLS
Hardware power management	IPMI
Remote startup	Wake-on-LAN
Graceful shutdown	Restricted SSH
Network integration	OPNsense API
Service management	systemd
Operational state	Persistent local journal

3. Key Features

Centralized Device Management

The dashboard provides a unified interface for managing multiple physical devices without requiring direct access to each host’s administrative interface.

Supported operations depend on the hardware and configured management capabilities.

Multiple Power Management Methods

Wake-on-LAN

Used to initiate startup of supported physical systems.

For devices on separate network segments, the controller can request Wake-on-LAN through an OPNsense relay rather than relying on local broadcast traffic.

IPMI

Provides out-of-band hardware power management for supported server hardware.

This allows the controller to query chassis power state and issue authorized power operations without relying on the server operating system.

Restricted SSH Shutdown

Provides graceful shutdown for supported Linux hosts.

Instead of granting unrestricted remote shell access, the system uses a constrained command execution model for approved shutdown operations.

Bulk Power Operations

The dashboard supports selecting multiple eligible devices and executing coordinated power operations.

Bulk operations are serialized to avoid uncontrolled concurrent actions.

Critical infrastructure is handled separately where required by the network topology.

Device Locks and Safety Controls

The application implements safeguards to reduce the risk of unintended operations:

* Device-level locks.
* Operation eligibility checks.
* Explicit confirmations.
* Rate limiting.
* Duplicate-operation prevention.
* Persistent operation leases.
* Serialized bulk execution.
* Conservative handling of unknown device states.

A successful command dispatch is not automatically treated as proof that the target device has completed the requested power transition.

Operational Journal

Power operations are recorded in a persistent local journal.

The journal helps track requested actions, dispatch results, and available confirmation evidence.

This provides traceability for troubleshooting and prevents the dashboard from presenting unverified device states as authoritative.

4. Security Architecture

Security was a core design requirement because the application can control physical infrastructure.

Private Remote Access

The dashboard is exposed through Tailscale rather than a public internet-facing endpoint.

Access is restricted to the private overlay network.

Authentication and Session Protection

The application implements:

* Authenticated web sessions.
* Secure and HttpOnly session cookies.
* SameSite cookie restrictions.
* CSRF protection.
* Backend authorization checks.
* Request validation and rate limiting.

Frontend confirmation dialogs are usability safeguards; security decisions are enforced independently by the backend.

Credential Management

Credentials are stored in protected local files with restricted access.

The application does not require exposing credentials in frontend JavaScript or public repositories.

Restricted Administrative Operations

Power management integrations are designed around narrowly scoped capabilities.

Examples include:

* Dedicated shutdown commands rather than unrestricted SSH execution.
* Device-specific management adapters.
* Explicitly configured target devices.
* Restricted power-control operations.
* TLS certificate validation for management API connections.

Service Isolation

The application runs as a managed Linux service with non-root execution and systemd sandboxing protections.

5. Network Resilience

One of the most important engineering challenges was ensuring that the management controller remained reachable when the main firewall was unavailable.

The Raspberry Pi participates in the management network while maintaining an alternative connectivity path.

The design incorporates:

* A dedicated Ethernet management connection.
* Static addressing for local management access.
* Alternative Wi-Fi connectivity.
* Tailscale remote access.
* Route monitoring and failover behavior.

This was particularly important for out-of-band management.

An earlier network configuration depended on DHCP services provided by the firewall being managed. When the firewall was powered off, the controller could lose the addressing needed to reach its BMC.

The issue was resolved by establishing persistent local management addressing independent of the firewall’s DHCP availability.

This allows the controller to communicate with the server’s management controller even when the primary firewall operating system is offline, provided the underlying management network remains available.

6. Engineering Challenges

Challenge 1 — Different Hardware Management Methods

Problem: Not all devices support the same startup and shutdown mechanisms.

Solution: Implemented device-specific control adapters and a common orchestration interface.

Challenge 2 — Managing Devices Across VLANs

Problem: Wake-on-LAN broadcasts do not automatically traverse routed network boundaries.

Solution: Integrated the OPNsense API to initiate Wake-on-LAN from the appropriate network interface.

Challenge 3 — Maintaining Access During Firewall Downtime

Problem: The controller initially depended on network services provided by infrastructure it was supposed to manage.

Solution: Introduced independent management addressing and alternative connectivity, reducing the dependency on the firewall’s operating state.

Challenge 4 — Avoiding Conflicting Power Operations

Problem: Repeated clicks, concurrent requests, and bulk operations could produce inconsistent results.

Solution: Implemented eligibility validation, operation leases, device locks, serialization, and persistent journaling.

Challenge 5 — Distinguishing Command Acceptance From Actual Power State

Problem: An API accepting a Wake-on-LAN request does not guarantee that a device has started.

Solution: Separated command dispatch results from observed or authoritative power-state confirmation.

7. Validation and Operational Experience

The system has been deployed in a real multi-device home laboratory.

Validated workflows include:

* Remote access to the management dashboard.
* Individual power management through supported adapters.
* Coordinated bulk shutdown of multiple systems.
* Wake-on-LAN startup for supported hosts.
* IPMI-based server power management.
* Protected access to OPNsense management APIs.
* Controller connectivity during firewall downtime.
* Persistent operation tracking and safety enforcement.

The implementation was refined through practical end-to-end testing, including network outages, routing issues, power-state inconsistencies, and service startup failures.

Known Limitations

Wake-on-LAN reliability depends on hardware, firmware, NIC power management, and network delivery.

One legacy Mac mini has exhibited inconsistent wake behavior following shutdown. The application successfully dispatches requests through the OPNsense API, but packet delivery and hardware wake behavior have not yet been conclusively verified.

For devices without authoritative out-of-band power-state reporting, the dashboard deliberately avoids treating network reachability as definitive proof of chassis power state.

8. Project Outcomes

Home Lab Power Control provides a single management interface for a heterogeneous physical laboratory.

The project demonstrates practical experience with:

* Linux systems administration.
* Python backend development.
* REST API integration.
* Network segmentation and routing.
* Secure remote administration.
* Hardware management protocols.
* Authentication and access controls.
* Operational state management.
* Failure analysis and troubleshooting.
* Infrastructure automation.

The system was developed to solve real operational problems in an actively used home laboratory rather than as a purely theoretical demonstration.

9. Project Documentation

This repository documents the architecture, design decisions, security controls, and engineering lessons learned.

The application source code, deployment configurations, credentials, internal network addressing, and sensitive infrastructure details are intentionally not published.

10. Related Projects

NetSec-HomeLab
Network security laboratory, OPNsense, network segmentation, traffic analysis, and infrastructure experiments.

Lab Assistant
A separate local AI assistant project focused on controlled interaction with homelab infrastructure.

GETHRA
A security intelligence platform exploring telemetry ingestion, deterministic detection, correlation, investigation context, and evidence-driven security analysis.

⸻

Designed, implemented, deployed, and maintained as an independent personal engineering project.
