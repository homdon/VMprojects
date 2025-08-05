# Vagrant Jenkins VM - Troubleshooting Summary

This document summarizes the issues encountered and solutions applied during the setup of the Jenkins virtual machine.

### 1. Jenkins Installation Failure during Provisioning

*   **Problem:** The initial `vagrant up` command failed. The provisioning script could not install Jenkins because of a GPG key error (`NO_PUBKEY`) and a "Package 'jenkins' has no installation candidate" message.
*   **Root Cause:** The script attempted to save the Jenkins repository key to `/etc/apt/keyrings/jenkins-keyring.asc`, but the `/etc/apt/keyrings` directory does not exist by default on the base Ubuntu image. This failure prevented the system from trusting the Jenkins repository.
*   **Solution:** The `Vagrantfile`'s provisioning script was modified to follow the modern, recommended procedure:
    1.  Explicitly create a standard directory for keys: `sudo mkdir -p /usr/share/keyrings`.
    2.  Download the Jenkins key directly to that new location.
    3.  Update the Jenkins repository source list to point to the key's new path.
    4.  This fix was applied by running `vagrant provision` on the existing VM.

### 2. Cannot Access Jenkins from Web Browser (Port Forwarding)

*   **Problem:** Jenkins was confirmed to be running inside the VM, but it was inaccessible from the host machine's web browser at `http://localhost:8080`.
*   **Root Cause:** Vagrant does not automatically map or forward ports from the guest VM to the host machine. A rule must be explicitly defined.
*   **Solution:** A port forwarding rule was added to the `Vagrantfile` to map port 8080 on the host to port 8080 on the guest:
    ```ruby
    config.vm.network "forwarded_port", guest: 8080, host: 8080
    ```
    This change required a `vagrant reload` to be applied to the running VM.

### 3. Cannot Access Jenkins via Static IP Address

*   **Problem:** After configuring the VM with a static IP (`192.168.56.91`), Jenkins was still inaccessible at that address from the host browser, even though it was accessible via `localhost:8080`.
*   **Root Cause:** There was a fundamental mismatch between the network *type* defined in the `Vagrantfile` and the IP address *range* being used.
    *   The configuration was `config.vm.network "public_network"`, which creates a **Bridged Network** that tries to connect to the physical network (e.g., your Wi-Fi).
    *   However, the IP `192.168.56.91` belongs to a special range that VirtualBox uses for its internal **Host-Only Network**.
    *   The host's physical Wi-Fi network had no route to the internal VirtualBox network, so the connection timed out.
*   **Solution:** The network type in the `Vagrantfile` was corrected to match the purpose of the IP address. The line was changed to:
    ```ruby
    config.vm.network "private_network", ip: "192.168.56.91"
    ```
    This created the correct Host-Only network, allowing direct communication between the host machine and the VM. This final fix was applied with `vagrant reload`.
