# `scsserver` Ansible Deployment Guide

This document provides instructions and details for the automated deployment of the `scsserver` application using Ansible.

## Table of Contents

1.  [Overview](#1-overview)
2.  [Prerequisites](#2-prerequisites)
3.  [Deployment Process](#3-deployment-process)
    - [Key Concepts](#key-concepts)
    - [Step-by-Step Flow](#step-by-step-flow)
4.  [Ansible Role Structure](#4-ansible-role-structure)
    - [Main Playbook: `scsserver-playbook.yml`](#main-playbook-scsserver-playbookyml)
    - [Role: `scsserver`](#role-scsserver)
    - [Shared Role: `kmacommon`](#shared-role-kmacommon)
    - [Configuration: `group_vars/scsserver.yml`](#configuration-group_varsscsserveryml)
5.  [Running the Deployment](#5-running-the-deployment)
    - [Via Jenkins (Recommended)](#via-jenkins-recommended)
    - [Manual Execution](#manual-execution)
6.  [Verification and Rollback](#6-verification-and-rollback)
    - [Post-Deployment Checks](#post-deployment-checks)
    - [Manual Rollback Procedure](#manual-rollback-procedure)
7.  [File and Directory Structure on Target Host](#7-file-and-directory-structure-on-target-host)

---

### 1. Overview

The `scsserver` is a Java web application deployed on a Tomcat container. This Ansible project automates the entire deployment lifecycle, from fetching the application artifact to configuring, starting, and verifying the service. The automation is designed to be reliable, repeatable, and safe, minimizing the need for manual intervention.

### 2. Prerequisites

Before running a deployment, ensure the following conditions are met:

-   **Ansible Control Node:** A machine with Ansible installed and configured with access to the target servers. This is typically managed by the Jenkins runner.
-   **Target Host:** The destination server (e.g., `uehudson11`) must have SSH access enabled for the `build` user (or the user specified in the Ansible inventory).
-   **Application Bundle:** The `scsserver` application bundle (e.g., `scsserver-1.0.89.tar.gz`) must be promoted and available in the artifact repository at `/ings/archive/maven/<environment>/scsserver`.
-   **Ansible Inventory:** The target host must be correctly defined in the Ansible inventory file (e.g., `inventories/stable/hosts`).

### 3. Deployment Process

#### Key Concepts

-   **Blue-Green Symlink Strategy:** The deployment uses a symbolic link-based approach to ensure zero-downtime cutovers. A new version is deployed to a separate, version-numbered directory. The "live" traffic is switched to the new version by atomically updating a series of symlinks.
-   **Idempotency:** The Ansible playbook is written to be idempotent. Running it multiple times will result in the same state without causing errors.
-   **Reusability:** The deployment leverages a shared `kmacommon` role for common tasks like artifact handling and pre-flight checks, promoting code reuse and consistency across different application deployments.

#### Step-by-Step Flow

The automated deployment executes the following steps:

1.  **Locking:** Creates a `deploy.lock` file in the artifact repository to prevent concurrent deployments.
2.  **Artifact Fetching:** Copies the latest `scsserver-*.tar.gz` bundle from the artifact repository to a temporary directory on the target host.
3.  **Version Extraction:** Dynamically determines the version number from the bundle's filename (e.g., "1.0.89").
4.  **Shutdown:** Gracefully stops the currently running `scsserver` instance.
5.  **Staging:** Unpacks the new bundle into a version-specific directory (e.g., `/usr/local/ingenuity/scsserver/scsserver-1.0.89`).
6.  **Configuration:**
    -   Modifies `setup/scsserver.properties` to set the correct `scServerPort`.
    -   Updates the `docBase` path in `setup/contextroot.xml` to point to the correct live directory.
    -   Executes the application's internal `setup/setup.sh` script to apply configurations.
7.  **Symlink Cutover:**
    -   Creates a log directory symlink: `logs -> /usr/local/ingenuity/logs/scsserver/`.
    -   Updates the primary symlinks in a specific order:
        -   `/usr/local/ingenuity/scsserver/<port>` now points to the new version directory.
        -   `/usr/local/ingenuity/scsserver/scsserver-live` now points to the `<port>` symlink.
8.  **Activation:** Starts the `scsserver` application using the `scsserver-live` symlink.
9.  **Health Check:** Waits for a brief period, then repeatedly checks the status URL (`http://<host>:<port>/web/public/status.jsp`) until it receives a `200 OK` response. The deployment fails if the application does not come up in time.
10. **Cleanup:**
    -   Removes old, inactive version directories and bundles, keeping only the current and previous versions.
    -   Removes the `deploy.lock` file.

### 4. Ansible Role Structure

#### Main Playbook: `scsserver-playbook.yml`

This is the top-level playbook that initiates the deployment. Its sole purpose is to call the `scsserver` role.

#### Role: `scsserver`

This is the core role containing all logic specific to deploying the `scsserver` application. It handles stopping, configuring, linking, starting, and verifying the service.

#### Shared Role: `kmacommon`

A dependency of the `scsserver` role. It performs generic pre-deployment tasks that are common across multiple applications, such as locking, artifact synchronization, and directory creation.

#### Configuration: `group_vars/scsserver.yml`

This file contains variables that control the behavior of the `scsserver` deployment. Key variables include:

-   `application_name`: Set to `scsserver`.
-   `healthcheck_url`: The relative path to the application's status page.
-   `timeout`: The number of seconds to wait after starting the application before beginning the health checks.

### 5. Running the Deployment

#### Via Jenkins (Recommended)

The standard method for deployment is through the configured Jenkins job.

1.  Navigate to the `scsserver_stable_deploy_ansible` (or similar) job in Jenkins.
2.  Click **"Build with Parameters"**.
3.  Ensure the parameters (like `server_name`, `tcp_port`, etc.) are correct.
4.  Click **"Build"**.

The Jenkins console output will show the live progress of the Ansible playbook.

#### Manual Execution

For development or emergency troubleshooting, the playbook can be run manually from a configured Ansible control node.

```bash
# Example command
ansible-playbook scsserver-playbook.yml \
    -i inventories/stable/hosts \
    -t scsserver \
    -e server_name=uehudson11 \
    -e tcp_port=8090 \
    -e env=stable
```

**Note:** Ensure all required extra variables (`-e`) that are normally provided by Jenkins are included in the command.

### 6. Verification and Rollback

#### Post-Deployment Checks

After a successful deployment, you can manually verify the application state:

1.  **Check Symlinks:**
    ```bash
    ls -l /usr/local/ingenuity/scsserver/
    ```
    Verify that `scsserver-live` and the port number (`8090`) point to the newly deployed version directory.

2.  **Check Service Status:**
    ```bash
    /usr/local/ingenuity/scsserver/scsserver-live/bin/scsserver.sh status
    ```

3.  **Check Network Port:**
    ```bash
    netstat -tulpn | grep 8090
    ```
    Confirm that a Java process is listening on the configured port.

4.  **Check Health URL:**
    ```bash
    curl -I http://<server_name>.ingenuity.com:8090/web/public/status.jsp
    ```
    The command should return an `HTTP/1.1 200 OK` status.

#### Manual Rollback Procedure

If a deployment fails or introduces a critical issue, a rollback can be performed by reverting the symlinks to the previous version. The `kmacleanup` role is configured to keep the last working version.

1.  **Identify the previous version directory** (e.g., `scsserver-1.0.88`).
2.  **Stop the broken service:**
    ```bash
    /usr/local/ingenuity/scsserver/scsserver-live/bin/scsserver.sh stop
    ```
3.  **Update the symlinks to point to the previous version:**
    ```bash
    # Point the port symlink to the previous version
    ln -sfn /usr/local/ingenuity/scsserver/scsserver-1.0.88 /usr/local/ingenuity/scsserver/8090

    # The scsserver-live symlink will now automatically point to the old version
    ```
4.  **Start the old service:**
    ```bash
    /usr/local/ingenuity/scsserver/scsserver-live/bin/scsserver.sh start
    ```
5.  **Verify** that the old version is running correctly.

### 7. File and Directory Structure on Target Host

After a successful deployment of version `1.0.89` on port `8090`, the directory structure will look like this:

```
/usr/local/ingenuity/
|-- scsserver/
|   |-- 8090 -> /usr/local/ingenuity/scsserver/scsserver-1.0.89   (Port Symlink)
|   |-- scsserver-1.0.88/                                         (Previous Version - Kept for Rollback)
|   |-- scsserver-1.0.89/                                         (Current Live Version Directory)
|   |   |-- bin/
|   |   |-- conf/
|   |   |-- logs -> /usr/local/ingenuity/logs/scsserver/
|   |   |-- setup/
|   |   `-- web/
|   `-- scsserver-live -> /usr/local/ingenuity/scsserver/8090     (Live Symlink)
|
`-- logs/
    `-- scsserver/
        `-- (Application log files are written here)
```