# Zimaboard HA Solution

## Overview

This repository contains a comprehensive High Availability (HA) self-hosted solution designed for Zimaboard devices. It leverages Docker Compose to orchestrate a suite of services for media consumption (music, movies, audiobooks) and personal cloud storage, with a focus on resilience, security, and ease of management. The architecture supports two Zimaboard devices, one acting as a primary node and the other as a secondary/backup node, ensuring service continuity and data protection.

## Features

*   **Operating System:** ZimaOS (natively Dockerized)
*   **Containerization:** Docker and Docker Compose
*   **Reverse Proxy & Load Balancer:** Traefik (with automatic SSL/TLS certificate management)
*   **Dynamic DNS (DDNS):** Client for external access with dynamic IP addresses
*   **Secure Remote Access:** WireGuard VPN server
*   **Single Sign-On (SSO):** Authelia for centralized authentication and enhanced security
*   **Cloud Storage:** Nextcloud
*   **Music Server:** Navidrome
*   **Media Server:** Jellyfin
*   **Audiobook Server:** Audiobookshelf
*   **Database:** PostgreSQL (with streaming replication for HA)
*   **Monitoring:** Netdata (real-time system and application monitoring)
*   **Backup:** Duplicati (for incremental, encrypted backups to various destinations)
*   **High Availability (HA):** Active/passive setup for stateful services, active/active for stateless services, managed via Keepalived for Virtual IP (VIP) failover.

## Architecture Overview

The solution is designed to run across two Zimaboard devices:

*   **Node 1 (Primary):** Zimaboard with Intel N150, 16GB RAM. Hosts primary services and acts as the master for stateful services.
*   **Node 2 (Secondary/Backup):** Zimaboard with Intel N3450, 8GB RAM. Acts as a replica for stateful services (e.g., PostgreSQL) and can take over primary roles in case of Node 1 failure. Stateless services like Traefik can run in an active/active configuration.

Shared storage (e.g., NFS) is utilized to ensure data consistency and accessibility for stateful services across both nodes, facilitating failover and recovery.

## Getting Started

To deploy this solution, you will need two Zimaboard devices, external shared storage, and a domain name configured with a DDNS provider. The installation process involves configuring the base operating system, setting up shared storage, deploying Docker containers using `docker-compose`, and configuring HA mechanisms.

### Prerequisites

*   Two Zimaboard devices (recommended specs as above)
*   External Network Attached Storage (NAS) or a shared drive accessible via NFS/SMB
*   A registered domain name and access to a DDNS provider
*   Basic understanding of Linux command line, Docker, and networking.

### Installation

For detailed step-by-step installation and configuration instructions, please refer to the `docs/Installation_Maintenance.md` file in this repository. This document covers:

1.  **Initial Setup:** ZimaOS installation, network configuration, NFS client setup.
2.  **Docker Compose Configuration:** Customizing `docker-compose-node1.yml` and `docker-compose-node2.yml` for your environment.
3.  **Traefik Setup:** Configuration for reverse proxy, SSL, and HA.
4.  **DDNS Client:** Setting up the dynamic DNS client.
5.  **WireGuard VPN:** Deploying the VPN server for secure remote access.
6.  **Authelia SSO:** Integrating single sign-on for your web services.
7.  **Duplicati Backup:** Configuring your backup strategy.
8.  **Netdata Monitoring:** Setting up real-time monitoring.
9.  **HA Management:** Implementing Keepalived for VIP failover and service orchestration.

## Usage

Once installed, services will be accessible via their configured domain names (e.g., `nextcloud.your-domain.com`, `jellyfin.your-domain.com`). Remote access is secured via WireGuard VPN, and web services are protected by Traefik and Authelia SSO.

## Backup and Recovery

The solution includes Duplicati for automated, encrypted backups of critical data to a chosen destination. Disaster recovery procedures are outlined in the `docs/Installation_Maintenance.md` to guide you through restoring services and data in case of a failure.

## Monitoring

Netdata provides real-time performance monitoring for both Zimaboard nodes and all running Docker containers, offering insights into system health and resource utilization.

## Security Considerations

*   **WireGuard VPN:** Recommended for secure remote access to your home network.
*   **Authelia SSO:** Centralizes authentication for web services, supporting 2FA.
*   **Traefik:** Manages SSL/TLS certificates (Let's Encrypt) for encrypted web traffic.
*   **Firewall:** Ensure proper firewall rules (e.g., UFW on Ubuntu) are in place.

## Contributing

Feel free to fork this repository, open issues, or submit pull requests to improve the solution.

## License

This project is licensed under the MIT License - see the `LICENSE` file for details. (Note: A `LICENSE` file is not currently present in the repository, this is a placeholder.)

