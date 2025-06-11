This document outlines a plan to implement a robust, near-real-time data replication and disaster recovery (DR) solution between a primary Windows 10 Data Center (DC) machine and a secondary Disaster Recovery (DR) machine using Restic.

**Capabilities & Approach:**

The solution leverages Restic's command-line interface to perform highly efficient and secure backups. The core approach involves:

1.  **Automated Snapshots:** A scheduled task on the DC machine will capture an encrypted, deduplicated snapshot of the critical data directories every five minutes.
2.  **Secure Replication:** These snapshots will be immediately sent over the network to a secure Restic repository hosted on the DR machine, likely via the SFTP protocol.
3.  **Efficiency:** Restic's content-defined chunking ensures that only new or changed data blocks are transmitted, making the 5-minute interval feasible even over standard network links.
4.  **Rapid Failover:** In a DR event, a simple Restic command executed on the DR machine will restore the latest data snapshot, bringing the machine online with the most recent data.
5.  **Reversible Roles:** The architecture is symmetrical. After a failover, the DR machine can assume the DC role and begin replicating its data back to the original DC (which now serves as the new DR), ensuring business continuity.

**Outcome:**
The result is a cost-effective, secure, and highly automated DR strategy that provides a low Recovery Point Objective (RPO) of approximately 5 minutes and a fast Recovery Time Objective (RTO) limited only by the time to restore the data.

***

### **Brief Implementation Plan**

This plan details the steps to configure Restic for automated replication and failover.

#### **Phase 1: Initial Setup (One-Time)**

1.  **On the DR Machine (Repository Host):**
    * **Install an SFTP Server:** Set up a secure SFTP server (e.g., OpenSSH for Windows or Linux).
    * **Create a Backup User:** Create a dedicated, non-administrator user account for Restic (e.g., `restic-user`).
    * **Create Repository Directory:** Create a folder that will store the backup data (e.g., `D:\restic-repo`). Grant the `restic-user` full permissions to this directory.

2.  **On the DC Machine (Windows 10):**
    * **Install Restic:** Download the `restic.exe` binary from the official Restic website and place it in a known location (e.g., `C:\Program Files\Restic`). Add this location to your system's PATH environment variable.
    * **Create a Password File:** Create a text file containing a strong, unique password for repository encryption (e.g., `C:\restic\repo.pass`). Secure this file with appropriate filesystem permissions.
    * **Initialize the Repository:** Open Command Prompt or PowerShell and run the `init` command to set up the repository on the DR machine.
        ```bash
        restic -r sftp:restic-user@<DR_MACHINE_IP>:/D:/restic-repo --password-file C:\restic\repo.pass init
        ```
        *You should see a "created restic repository" success message.*

#### **Phase 2: Configure Automated Backup (DC to DR)**

1.  **Create a Backup Script:** On the DC machine, create a batch file (e.g., `C:\restic\run-backup.bat`). This script will perform the backup and clean up old snapshots.

    ```batch
    @echo OFF
    
    :: Set environment variables for Restic
    set RESTIC_REPOSITORY=sftp:restic-user@<DR_MACHINE_IP>:/D:/restic-repo
    set RESTIC_PASSWORD_FILE=C:\restic\repo.pass
    
    :: Path to the data you want to back up
    set SOURCE_DATA="C:\path\to\your\critical\data"
    
    :: Run the backup, adding a tag to identify the source
    echo "Starting Restic backup for %SOURCE_DATA%..."
    restic backup %SOURCE_DATA% --tag dc-backup
    
    :: Prune old snapshots to save space. Keeps the last 48 hourly snapshots (~2 days).
    echo "Pruning old snapshots..."
    restic forget --keep-hourly 48 --prune --tag dc-backup
    
    echo "Backup complete."
    ```

2.  **Schedule the Backup:**
    * Open **Task Scheduler** on the Windows 10 DC machine.
    * Create a new task.
    * **Trigger:** Set it to run on a schedule, repeating the task **every 5 minutes** indefinitely.
    * **Action:** Set it to "Start a program" and point it to your `C:\restic\run-backup.bat` script.
    * **Settings:** Crucially, set the option **"Do not start a new instance if one is already running"**. This prevents issues if a single backup job takes longer than 5 minutes.

#### **Phase 3: Failover Procedure (Manual Trigger)**

In the event the DC machine fails, perform these steps on the **DR machine**:

1.  **Install Restic:** Ensure `restic.exe` is installed on the DR machine.
2.  **Restore Data:** Create and run a restore script (`run-restore.bat`) on the DR machine.

    ```batch
    @echo OFF
    
    :: Set environment variables for Restic
    set RESTIC_REPOSITORY=D:\restic-repo  :: Use the local path since the repo is here
    set RESTIC_PASSWORD_FILE=D:\path\to\repo.pass :: Ensure the password file is on the DR machine
    
    :: The directory where the restored data should go
    set RESTORE_TARGET="C:\path\to\your\critical\data" 
    
    echo "Restoring latest snapshot to %RESTORE_TARGET%..."
    :: The restore command finds the latest snapshot and restores it
    restic restore latest --target %RESTORE_TARGET%
    
    echo "Restore complete. DR machine is now operational."
    ```
    Once this script finishes, the DR machine has the latest data and can take over the DC's workload.

#### **Phase 4: Role Reversal (Post-Failover)**

Once the DR machine is the new DC, you must re-establish replication to the original DC (which is now the new DR).

1.  **On the New DC (Original DR):**
    * Use the same backup script from **Phase 2**.
    * Ensure the `SOURCE_DATA` variable points to its local data directory.
    * Ensure the `RESTIC_REPOSITORY` variable points to the SFTP server on the **new DR machine** (the original DC).
    * Enable the 5-minute scheduled task.

2.  **On the New DR (Original DC):**
    * Ensure its SFTP server is running and the repository directory is accessible.
    * **Disable** the original backup task in Task Scheduler to prevent it from running.
    * It is now in a passive state, ready to receive backups or be restored in a future failover event.