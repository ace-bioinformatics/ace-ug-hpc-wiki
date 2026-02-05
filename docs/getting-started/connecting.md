---
id: connecting
title: Connecting to the Cluster
---

This guide explains how to connect to the ACE HPC Cluster using SSH.

### Prerequisites
- An active ACE HPC account.
- An SSH client (e.g., OpenSSH on Linux/Mac, PuTTY on Windows).

### Login Nodes
- Primary login Node: `biocompace.ace.ac.ug`
- Old Biocompace1 server: `biocompace1.ace.ac.ug`
- Old Biocompace2 server: `biocompace2.ace.ac.ug`

### Connecting via SSH
1. Open a terminal or SSH client.
2. Use the following command (replace `username` with your actual username):
- Primary login Node:
   ```bash
   ssh <username>@biocompace.ace.ac.ug
- Old Biocompace1 server:
   ```bash
   ssh <username>@biocompace1.ace.ac.ug
- Old Biocompace2 server:
   ```bash
   ssh <username>@biocompace2.ace.ac.ug
3. Enter your password when prompted.


## Connecting to the ACE HPC Cluster off-site
This guide provides instructions for users to connect to the African Centre of Excellence in Bioinformatics and Data Intensive Sciences (ACE) High Performance Computing (HPC) Cluster from off-site. Follow the steps below to establish a secure connection.

### Prerequisites
- An active ACE HPC account with a username and password.
- A computer with an SSH client installed (e.g., OpenSSH on Linux/Mac, or PuTTY on Windows).
- EduVPN client installed on your machine from the App Store for Mac and from the [EduVPN site](https://www.eduvpn.org/client-apps/) for Windows and Linux users.

### Connect to the ACE Network Using EduVPN
To access the ACE HPC Cluster from off-site, users must first connect to the ACE network using EduVPN. Follow these steps:

1. **Launch EduVPN and Select Your Institution**  
   Open the EduVPN application and select "African Centre of Excellence in Bioinformatics" as the institution from the list.

2. **Authenticate with Your Credentials**  
   Log in using the same credentials for your HPC account. Once authenticated, the EduVPN client will establish a secure connection to the ACE network.

### Connect to the ACE HPC Cluster via SSH
After successfully connecting to the ACE network through EduVPN, users can access the HPC cluster using SSH. Follow these steps:

1. **Open a Terminal**  
   Launch a terminal application on your computer. For Windows users, this could be a tool like PuTTY or the Windows Subsystem for Linux (WSL) terminal. For Mac and Linux users, the default terminal application will suffice.

2. **Enter the SSH Command**  
   In the terminal, enter the following command to connect to the ACE HPC Cluster:  
   ```bash
   ssh <username>@biocompace.ace.ac.ug