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

### Corrections and Updates

- [`corrected_network_setup.md`](corrected_network_setup.md) - Updated networking procedures
- [`corrected_storage_setup.md`](corrected_storage_setup.md) - Revised storage configuration

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
5. **Storage Integration** - Shared storage configuration
6. **Service Deployment** - Essential homelab services
7. **Gaming Optimization** - Performance tuning and GPU setup

## 📝 Notes

- Each guide includes estimated completion times
- Corrections files contain updated procedures when original guides needed revision
- Focus on Dell OptiPlex micro form factor for space and power efficiency
- Designed for home environment with gaming and productivity workloads

## 🤝 Contributing

This is a personal homelab documentation project. Feel free to adapt the procedures for your own hardware and requirements.

## 📜 License

Documentation provided as-is for educational and personal use.