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

## Git Troubleshooting Summary for jenkins-vm

**Problem:** The `jenkins-vm` directory and its subfolders were not being pushed to the Git repository, even though the main folder was present. This was due to Git ignoring the contents, likely because of `.vagrant` directories within the `jenkins-vm` structure, which are typically used by Vagrant for VM state and are often ignored by default or explicitly.

**Troubleshooting Steps:**
1.  **Initial `git status` check:** Revealed that `jenkins-vm` was listed as an untracked directory, indicating its contents were not being staged for commit.
2.  **Attempted `git add jenkins-vm`:** This did not resolve the issue, suggesting a deeper ignore rule or Git's handling of nested repositories.
3.  **Checked for nested Git repositories:** Used `ls -a jenkins-vm` to see if `jenkins-vm` itself was a Git repository (which would prevent its contents from being tracked by the parent repo). It was not, but it contained `.vagrant` directories.
4.  **Identified `.vagrant` as the culprit:** The `.vagrant` directories are internal to Vagrant and should generally not be tracked by Git. Their presence can sometimes interfere with Git's ability to track the parent directory's contents if not properly ignored.
5.  **Checked existing `.gitignore`:** Found that the entire `jenkins-vm/` directory was being ignored.

**Solution:**
1.  **Modified `.gitignore`:** Updated the project's `.gitignore` file to specifically ignore only the `.vagrant` directories within `jenkins-vm` and its subdirectories, rather than the entire `jenkins-vm` folder.
    *   Original: `jenkins-vm/`
    *   Changed to:
        ```
        jenkins-vm/.vagrant/
        jenkins-vm/sonar-server-jenkins-dependency/.vagrant/
        jenkins-vm/nexus-server-jenkins-dependency/.vagrant/
        ```
2.  **Staged changes:** Used `git add jenkins-vm` to stage all the files within `jenkins-vm` (excluding the newly ignored `.vagrant` directories). Also staged the modified `.gitignore` file itself.
3.  **Committed changes:** Created a commit with a descriptive message.

**For Future Reference:**
*   When encountering issues with Git not tracking files within a directory that contains VM-related files (like Vagrant's `.vagrant` folder), always check your `.gitignore` file.
*   Ensure that only the VM's internal state directories (like `.vagrant/`) are ignored, and not the entire project directory containing the VM configuration files (like `Vagrantfile`).
*   If a directory is unintentionally ignored, modify `.gitignore` to be more specific.
*   Remember that `.gitignore` rules apply recursively. If a parent directory is ignored, its contents are also ignored unless explicitly re-included with a `!` prefix in `.gitignore`.
*   If you suspect a directory is a nested Git repository preventing tracking, you can remove its internal `.git` folder (but be cautious, as this will delete its separate Git history). In this case, it was not a nested Git repo, but rather an overly broad `.gitignore` rule.