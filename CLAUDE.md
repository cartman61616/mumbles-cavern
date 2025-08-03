# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **documentation repository** for "Mumbles Cavern" - a Dell OptiPlex homelab cluster deployment guide. This is NOT a software project with code to build/test/lint, but rather a comprehensive documentation project for deploying Proxmox-based infrastructure.

## Repository Structure

This repository contains detailed deployment guides organized in phases:

### Core Documentation Files
- `master_deployment_guide.md` - Main deployment orchestration and phase overview
- `README.md` - Project overview and quick start guide
- `bios_optimization.md` - Hardware preparation and BIOS optimization
- `network_setup.md` - VLAN architecture and UniFi configuration
- `primary_node_setup.md` - First Proxmox node installation
- `cluster_setup.md` - Multi-node cluster formation
- `storage_setup.md` - Shared storage configuration
- `essential_services.md` - Core homelab services
- `external_access.md` - Remote access setup
- `gaming_setup.md` - Gaming optimization and GPU passthrough
- `shared_infrastructure.md` - Common infrastructure components

### Progress Tracking Files
- `network_setup_progress.md` - Network configuration status updates

## Project Architecture

### Hardware Infrastructure
- **4x Dell OptiPlex nodes** (3x 7050 Micro, 1x 6500T)
- **UniFi networking** (Dream Machine Pro + Flex Mini switch)
- **Proxmox VE cluster** as virtualization platform
- **VLAN segmentation** for network security and performance
- **Shared TrueNAS storage** plus local storage per node

### Network Architecture
- **VLAN 1** (Default): Initial deployment and user devices
- **VLAN 10** (Management): Infrastructure (Proxmox, NAS)
- **VLAN 20** (Services): Homelab services (Plex, *arr stack)
- **VLAN 80** (Gaming): Gaming devices and services

### Deployment Phases
1. **Hardware Preparation** - BIOS optimization
2. **Network Infrastructure** - VLAN and security setup
3. **Primary Node** - First Proxmox installation
4. **Cluster Formation** - Multi-node setup
5. **Storage Integration** - Shared storage configuration
6. **Service Deployment** - Essential homelab services
7. **Gaming Optimization** - Performance tuning

## Working with This Repository

### Key IP Ranges
- **Management VLAN**: 192.168.10.x
- **Services VLAN**: 192.168.20.x  
- **Gaming VLAN**: 192.168.80.x
- **Default/User VLAN**: 192.168.1.x

### Current Deployment Status
The project tracks progress through checkbox lists in `README.md` and dedicated progress files. Phase completion status should be updated as deployment progresses.

### Documentation Standards
- Each phase guide includes estimated completion times
- Guides reference prerequisite phases and next steps
- Hardware-specific procedures focus on Dell OptiPlex micro form factor
- Network procedures are UniFi-specific but adaptable

### Future Infrastructure Repositories
The documentation references planned code repositories for automation:
- `mumbles-cavern-ansible` - Ansible playbooks
- `mumbles-cavern-terraform` - Infrastructure as Code
- `mumbles-cavern-kubernetes` - K8s manifests
- `mumbles-cavern-monitoring` - Monitoring stack
- `mumbles-cavern-services` - Application deployments

## Commands and Operations

This is a documentation-only repository. There are no build, test, or lint commands. The primary operations are:

### Documentation Updates
- Update progress tracking in `README.md`
- Create correction files when procedures need revision
- Update IP addresses and configuration details as infrastructure evolves

### Content Validation
- Verify command syntax in documentation blocks
- Ensure cross-references between guides remain accurate
- Validate network diagrams and IP allocations

## Notes for Development

- This repository contains **homelab infrastructure documentation**, not application code
- Focus on **Dell OptiPlex** hardware specifics and **UniFi** networking
- Procedures are designed for **home environment** with gaming and productivity workloads
- Documentation emphasizes **phase-based deployment** to reduce complexity