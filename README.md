# RKE2 Kubernetes Setup Guide

Welcome to the **RKE2 Kubernetes Setup Guide**! This repository provides a comprehensive, step-by-step guide for deploying an RKE2 Kubernetes cluster. It covers everything from installing the RKE2 server and worker nodes to setting up tools like K9s, MetalLB, Uptime Kuma, NGINX Ingress, and deploying a Flask demo application using Helm. Additionally, it includes instructions for pushing Docker images to Docker Hub and resetting the cluster.

Whether you're a beginner or an experienced Kubernetes user, this guide is designed to help you build a robust cluster with practical examples and configurations.

## Features
- **RKE2 Cluster Setup**: Install and configure RKE2 server and worker nodes.
- **Cluster Management**: Use K9s for an intuitive terminal-based UI.
- **Load Balancing**: Implement MetalLB for bare-metal environments.
- **Monitoring**: Deploy Uptime Kuma with persistent storage.
- **Ingress Routing**: Set up NGINX Ingress for advanced traffic management.
- **Application Deployment**: Build, push, and deploy a Flask app using Helm.
- **Cluster Reset**: Instructions for safely resetting the RKE2 cluster.


## Prerequisites
- A Linux-based system (Ubuntu recommended).
- Root or sudo access for installing dependencies.
- Basic knowledge of Kubernetes and Docker.

## Usage
The guide is organized into sections for easy navigation:
- [Install RKE2 Server](#install-rke2-server)
- [Install Worker Nodes](#install-rke2-worker-node)
- [Deploy Tools and Apps](#install-metallb)
- [Helm Deployment](#deploy-flask-demo-app-with-helm)
- [Cluster Reset](#reset-rke2-cluster)

Refer to the detailed instructions in the [setup guide](docs/setup-guide.md) for each step.

## Contributing
Contributions are welcome! Feel free to open issues or submit pull requests to improve the guide or add new features.

## License
This project is licensed under the [MIT License](LICENSE).

## References
- [RKE2 Documentation](https://docs.rke2.io/)
- [MetalLB Documentation](https://metallb.universe.tf/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [Helm Documentation](https://helm.sh/docs/)
