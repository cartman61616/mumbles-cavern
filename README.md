# Mumbles Cavern - Dell OptiPlex Homelab Cluster

A comprehensive deployment guide for building a Proxmox-based homelab cluster using Dell OptiPlex micro nodes with UniFi networking and shared storage.

## 🏠 Project Overview

This repository contains detailed documentation for deploying a complete homelab infrastructure called "Mumbles Cavern" - a multi-node Proxmox cluster built with Dell OptiPlex hardware, featuring advanced networking, shared storage, and gaming optimization.

## 🖥️ Hardware Specifications

- **Compute Nodes**: 4x Dell OptiPlex (3x 7050 Micro, 1x 6500T)
- **CPU**: Intel i5 processors (6th/7th gen)
- **Memory**: 32GB RAM per node
- **Networking**: UniFi Dream Machine Pro + Flex Mini switch
- **Storage**: ASUSTOR TrueNAS (shared) + local NVMe/SSD per node
- **Power**: Anker Prime USB-C charger with barrel jack adapters

## 📚 Documentation Structure

### Core Deployment Guides

| File | Purpose | Estimated Time |
|------|---------|----------------|
| [`master_deployment_guide.md`](master_deployment_guide.md) | Complete deployment overview and phase coordination | N/A |
| [`bios_optimization.md`](bios_optimization.md) | BIOS updates and hardware optimization | 1-2 days |
| [`network_setup.md`](network_setup.md) | VLAN architecture and UniFi configuration | 4-6 hours |
| [`primary_node_setup.md`](primary_node_setup.md) | First Proxmox node installation and setup | 2-3 hours |
| [`cluster_setup.md`](cluster_setup.md) | Multi-node cluster formation | 2-4 hours |

### Infrastructure Components

| File | Purpose |
|------|---------|
| [`storage_setup.md`](storage_setup.md) | Shared storage configuration |
| [`essential_services.md`](essential_services.md) | Core homelab services deployment |
| [`external_access.md`](external_access.md) | Remote access and security setup |
| [`gaming_setup.md`](gaming_setup.md) | Gaming optimization and GPU passthrough |
| [`shared_infrastructure.md`](shared_infrastructure.md) | Common infrastructure components |
| [`cavern_learnings.md`](cavern_learnings.md) | Lessons learned, troubleshooting, and optimization insights |
| [`transition_plan.md`](transition_plan.md) | Comprehensive roadmap from current state to final Snorlax Edition |


## 🚀 Quick Start

1. **Start Here**: Read [`master_deployment_guide.md`](master_deployment_guide.md) for complete overview
2. **Hardware Prep**: Follow [`bios_optimization.md`](bios_optimization.md) to prepare Dell nodes
3. **Network Setup**: Configure UniFi infrastructure using [`network_setup.md`](network_setup.md)
4. **First Node**: Deploy primary Proxmox node with [`primary_node_setup.md`](primary_node_setup.md)
5. **Scale Up**: Add remaining nodes using [`cluster_setup.md`](cluster_setup.md)

## 🎯 Key Features

- **Proxmox VE Cluster**: High-availability virtualization platform
- **Advanced Networking**: VLAN segmentation with gaming optimization
- **Shared Storage**: Centralized TrueNAS with Proxmox integration
- **Gaming Ready**: GPU passthrough and network QoS for gaming VMs
- **Remote Access**: Secure external connectivity options
- **Power Efficient**: Optimized for low-power Dell micro form factor

## ⚠️ Prerequisites

- Basic Linux command line experience
- Dell OptiPlex hardware (or similar micro PC)
- UniFi networking equipment
- Password manager for credential storage
- Reliable internet connection for downloads

## 🔧 Deployment Phases

The deployment is structured in logical phases to minimize complexity:

1. **Hardware Preparation** - BIOS optimization and validation
2. **Network Infrastructure** - VLAN and security configuration  
3. **Primary Node** - First Proxmox installation
4. **Cluster Formation** - Multi-node setup
5. **VPN Access** - Secure remote connectivity (priority moved up)
6. **Storage Integration** - Shared storage configuration
7. **Service Deployment** - Essential homelab services
8. **Gaming Optimization** - Performance tuning and GPU setup

## 📊 Progress Report

Track your deployment progress using this checklist:

### Phase Status
- [x] **Hardware Preparation** - BIOS updates and optimization complete
- [x] **Network Infrastructure** - VLANs, QoS, and switch configuration complete! 🚀 ([Progress](network_setup_progress.md))
- [x] **Primary Node** - Proxmox installed and inter-VLAN routing working! 🚀
- [x] **Cluster Formation** - 2-node cluster created and operational! 🚀
- [ ] **VPN Access** - Secure remote connectivity established (moved up in priority)
- [ ] **Storage Integration** - TrueNAS and local storage configured
  - 🔄 **Currently migrating data to extra drives for ASUSTOR NAS reset**
- [ ] **Essential Services** - Core homelab services deployed
- [ ] **External Access** - Additional remote connectivity options
- [ ] **Gaming Setup** - GPU passthrough and optimization complete

### Infrastructure Status
- [x] **Hardware Validated** - All Dell nodes tested and operational
- [x] **Network Segmentation** - VLANs properly configured and tested
- [ ] **High Availability** - Cluster failover tested
- [ ] **Backup Strategy** - Data protection implemented
- [ ] **Monitoring** - System health monitoring active
- [ ] **Documentation** - Infrastructure as Code documented

### Next Steps
- [ ] Performance benchmarking and optimization
- [ ] Advanced service deployment (CI/CD, development environments)
- [ ] Disaster recovery testing
- [ ] Capacity planning and scaling preparation

## 🔗 Infrastructure Repositories

As your homelab grows, consider organizing infrastructure code in dedicated repositories:

### Planned Repository Structure
```
mumbles-cavern/
├── mumbles-cavern-docs/          # This repository - Documentation
├── mumbles-cavern-ansible/       # Ansible playbooks and automation
├── mumbles-cavern-terraform/     # Infrastructure as Code (IaC)
├── mumbles-cavern-kubernetes/    # K8s manifests and configs
├── mumbles-cavern-monitoring/    # Prometheus, Grafana, alerting
└── mumbles-cavern-services/      # Application deployments and configs
```

### Repository Links
> **Note**: These repositories will be created as the infrastructure grows and requires code management.

- **📖 Documentation**: `mumbles-cavern-docs` (This repository)
- **🤖 Automation**: `mumbles-cavern-ansible` *(Coming Soon)*
- **🏗️ Infrastructure as Code**: `mumbles-cavern-terraform` *(Coming Soon)*
- **☸️ Kubernetes**: `mumbles-cavern-kubernetes` *(Coming Soon)*
- **📈 Monitoring**: `mumbles-cavern-monitoring` *(Coming Soon)*
- **🚀 Services**: `mumbles-cavern-services` *(Coming Soon)*

### Integration Strategy
- **GitOps Workflow**: Use Git repositories as the source of truth
- **CI/CD Pipeline**: Automated testing and deployment
- **Infrastructure as Code**: Version-controlled infrastructure definitions
- **Configuration Management**: Ansible for system configuration
- **Service Deployment**: Kubernetes or Docker Compose manifests

## 📝 Notes

- Each guide includes estimated completion times
- Corrections files contain updated procedures when original guides needed revision
- Focus on Dell OptiPlex micro form factor for space and power efficiency
- Designed for home environment with gaming and productivity workloads

## 🤝 Contributing

This is a personal homelab documentation project. Feel free to adapt the procedures for your own hardware and requirements.

## 📜 License

Documentation provided as-is for educational and personal use.