# Kubernetes Syncthing with Automated Borg Backups

This repository contains the Kubernetes manifests to deploy a self-hosted Syncthing instance with automated, encrypted, and deduplicated backups to a Hetzner Storage Box using BorgBackup. It also includes an optional all-in-one deployment for Immich to serve photos from the same data volume.

### Technologies Used
* **Container Orchestration**: [Kubernetes](https://kubernetes.io/) (specifically [MicroK8s](https://microk8s.io/))
* **File Synchronization**: [Syncthing](https://syncthing.net/)
* **Backup Software**: [BorgBackup](https://www.borgbackup.org/) (Borg)
* **Photo Management**: [Immich](https://immich.app/) (Optional)
* **Remote Storage**: [Hetzner Storage Box](https://www.hetzner.com/storage/storage-box)
* **Authentication**: SSH Key-based authentication

### The 3-2-1 Backup Strategy

This setup is designed to implement the industry-standard 3-2-1 backup rule for your personal data (e.g., photos on your phone).

* **3 Copies of Your Data:**
    1.  **Primary Copy**: Your phone or desktop computer.
    2.  **Second Copy**: This Kubernetes server, which receives files via Syncthing.
    3.  **Third Copy**: The Hetzner Storage Box, which receives files via BorgBackup.

* **2 Different Media:**
    1.  **Media 1**: The local storage on your phone/desktop.
    2.  **Media 2**: The server's storage (`hostPath` volumes).
    3.  **Media 3 (optional)**: Your local desktop 

* **1 Off-site Copy:**
    1.  The **Hetzner Storage Box** serves as your secure, off-site backup, protecting you from local disasters like fire, theft, or hardware failure.

---

## Getting Started: Quick Start Guide

1.  **Prepare Remote Storage & SSH Key**
    * In your Hetzner console, enable **SSH support** and **BorgBackup support** for your Storage Box.
    * Generate a dedicated SSH key: `ssh-keygen -t ed25519 -f ./borg_id_ed25519`.
    * **Authorize your new key on the Hetzner Storage Box.** The easiest way is to use `ssh-copy-id`. This command will connect to your storage box (asking for your password this one time) and automatically append the public key correctly.
        ```bash
        # This copies your public key to the server
        ssh-copy-id -i ./borg_id_ed25519 uXXXXXX@uXXXXXX.your-storagebox.de
        ```
    * **Alternatively, manually add the key.** Copy the content of your public key (`cat ./borg_id_ed25519.pub`) and paste it on a new line in the `~/.ssh/authorized_keys` file on your Hetzner Storage Box.

2.  **Create Kubernetes Secrets**
    * **Test the SSH connection once manually.** This will add the server's fingerprint to your local `known_hosts` file, which is more secure than using `ssh-keyscan`. You will be asked to trust the key; type `yes`.
        ```bash
        # Use your private key and Hetzner username/address
        ssh -i ./borg_id_ed25519 uXXXXXX@uXXXXXX.your-storagebox.de
        ```
    * **Copy your `known_hosts` file and create the SSH secret.** This is simpler than extracting a single key.
        ```bash
        # First, copy your local known_hosts file to the current directory
        cp ~/.ssh/known_hosts .

        # Then, create the Kubernetes secret from your private key and the copied file
        # However you could also use the example in the secrets folder
        kubectl create secret generic borg-ssh-secret \
          --from-file=id_rsa=./borg_id_ed25519 \
          --from-file=known_hosts=./known_hosts
        ```
    * **Use the example file in the secrets folder and create the borg-credentials secret**
3.  **Deploy Storage and Applications**
    * **Create Host Directories**: Before applying the volumes, create the directories on your node:
      ```bash
      sudo mkdir -p /mnt/syncthing-config
      sudo mkdir -p /mnt/syncthing-data
      ```
    * **Apply the Kubernetes manifests** in logical order.
      ```bash
      # Apply secret first
      kubectl apply -f secrets/syncthing-secret.yml

      # Deploy Syncthing
      kubectl apply -f syncthing.yml
      
      # (Optional) Deploy Immich
      kubectl apply -f immich.yml
      ```

4.  **Initialize Repository and Configuration (One-Time Actions)**
    * Apply the `syncthing-testpod.yml` to initialize the remote repository. Check its logs, then delete the job.
    * Access the Syncthing UI and configure your folders and devices.