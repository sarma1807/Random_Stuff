To build an Oracle Data Guard configuration with a single-instance primary database and a single-instance standby database where the primary database was originally a 2-node RAC, you'll essentially be performing a single-instance to single-instance Data Guard setup. The fact that the primary was once RAC is mostly relevant during the initial setup and configuration.

Here's a general outline of the steps involved. Keep in mind that specific commands and paths might vary depending on your Oracle version and environment.

**I. Prerequisites on the Primary (Former RAC):**

1.  **Ensure Database is in ARCHIVELOG Mode:**
    ```sql
    SQL> archive log list;
    SQL> alter database archivelog;
    SQL> archive log list;
    ```

2.  **Enable Force Logging (Recommended):**
    ```sql
    SQL> alter database force logging;
    SQL> select force_logging from v$database;
    ```

3.  **Configure Initialization Parameters:** You'll need to set appropriate Data Guard related parameters in the primary database's `spfile` or `pfile`. These parameters are typically set with `SCOPE=BOTH` if using an `spfile`.

    * **`DB_UNIQUE_NAME`:** Set a unique name for the primary database (e.g., `primary_db`).
    * **`LOG_ARCHIVE_DEST_1`:** Configure a local archive log destination on the primary.
        ```sql
        SQL> alter system set log_archive_dest_1='LOCATION=/path/to/primary/archivelog/ VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=primary_db' SCOPE=BOTH SID='*';
        ```
    * **`LOG_ARCHIVE_DEST_2`:** Configure the connection information for the standby database. Use the `SERVICE` attribute with the TNS alias of the standby database.
        ```sql
        SQL> alter system set log_archive_dest_2='SERVICE=standby_db LGWR ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=standby_db' SCOPE=BOTH SID='*';
        ```
    * **`LOG_ARCHIVE_CONFIG`:** Enable sending of archive logs to the standby destinations.
        ```sql
        SQL> alter system set log_archive_config='DG_CONFIG=(primary_db,standby_db)' SCOPE=BOTH SID='*';
        ```
    * **`REMOTE_LOGIN_PASSWORDFILE`:** Set this to `EXCLUSIVE`.
        ```sql
        SQL> alter system set REMOTE_LOGIN_PASSWORDFILE=EXCLUSIVE SCOPE=SPFILE;
        ```
    * **`STANDBY_FILE_MANAGEMENT`:** Set this to `AUTO` for automatic file management on the standby.
        ```sql
        SQL> alter system set STANDBY_FILE_MANAGEMENT=AUTO SCOPE=BOTH SID='*';
        ```
    * **`FAL_SERVER` and `FAL_CLIENT` (Optional but Recommended for Fetch Archive Log):**
        ```sql
        SQL> alter system set FAL_SERVER=primary_db SCOPE=BOTH SID='*';
        SQL> alter system set FAL_CLIENT=standby_db SCOPE=BOTH SID='*';
        ```
    * **`DB_FILE_NAME_CONVERT` and `LOG_FILE_NAME_CONVERT` (If file paths are different):** These are crucial if your standby database file system structure differs from the primary. However, since the standby is a new single instance, you'll likely handle file placement during the restore/duplicate process.

4.  **Create Standby Redo Logs (SRLs) on the Primary:** The number and size of SRLs should match the online redo logs on each RAC instance of the *original* primary database. If your original RAC had 2 nodes with 3 redo log groups each, you'd need (2+1) \* 2 = 6 SRL groups. Create them for each thread.
    ```sql
    -- Assuming 2 threads in the original RAC
    ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 SIZE 100M;
    ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 SIZE 100M;
    ALTER DATABASE ADD STANDBY LOGFILE THREAD 1 SIZE 100M;
    ALTER DATABASE ADD STANDBY LOGFILE THREAD 2 SIZE 100M;
    ALTER DATABASE ADD STANDBY LOGFILE THREAD 2 SIZE 100M;
    ALTER DATABASE ADD STANDBY LOGFILE THREAD 2 SIZE 100M;
    ```
    Adjust the `SIZE` as needed.

5.  **Create a Password File:** Ensure a password file exists for the primary instance. If you're using ASM, it might be in a shared location. Copy it to a location accessible by the standby server.

6.  **Configure Network (TNS):**
    * On the primary server, configure `tnsnames.ora` to include the connection details for the standby database (using the `standby_db` service name defined in `LOG_ARCHIVE_DEST_2`).
    * On the standby server, configure `tnsnames.ora` to include the connection details for the primary database (using the `primary_db` service name). Ensure the standby listener is configured and running.

**II. Create the Standby Database (Single Instance):**

You can create the standby database using either of the following methods:

**Method 1: Using RMAN `DUPLICATE DATABASE`**

This is the most common and recommended method.

1.  **Prepare the Standby Server:** Ensure Oracle software is installed on the standby server. Create the necessary directory structure for the database files, archive logs, etc.

2.  **Start the Standby Instance in NOMOUNT:** Create an initialization parameter file (`initstandby.ora` or `spfilestandby.ora`) for the standby instance. Set the following essential parameters:
    * `DB_NAME`: Must be the same as the primary database.
    * `DB_UNIQUE_NAME`: Must be different from the primary (e.g., `standby_db`).
    * `CONTROL_FILES`: Specify the initial location for the standby control files.
    * `LOG_ARCHIVE_DEST_1`: Configure a local archive log destination on the standby.
    * `STANDBY_FILE_MANAGEMENT=AUTO`
    * `REMOTE_LOGIN_PASSWORDFILE=EXCLUSIVE`
    * `FAL_SERVER=primary_db`
    * `FAL_CLIENT=standby_db`
    * **Crucially, ensure `instance_number` and other RAC-specific parameters are NOT set in the standby initialization file.**

    Start the standby instance in `NOMOUNT` mode:
    ```sql
    SQL> startup nomount pfile='/path/to/initstandby.ora';
    ```

3.  **Run RMAN on the Standby Server:** Connect to the primary database using RMAN and duplicate the database.
    ```bash
    rman target sys/password@primary_db auxiliary sys/password@standby_db

    RMAN> run {
    allocate channel prim type disk;
    allocate auxiliary channel stby type disk;
    duplicate target database
    for standby
    db_unique_name 'standby_db'
    nofilenamecheck;
    }
    ```
    Adjust the connection strings and channel allocations as needed. The `NOFILENAMECHECK` option is often used when the file paths are different. You might need to use `DB_FILE_NAME_CONVERT` and `LOG_FILE_NAME_CONVERT` within the `DUPLICATE` command if the file structures are significantly different, though with a new single-instance standby, letting RMAN handle the creation in the desired locations is usually simpler.

**Method 2: Using Backup and Restore**

This method involves taking backups of the primary database and restoring them on the standby server. It's generally more complex than the `DUPLICATE` method.

1.  **Take a Full Backup of the Primary Database.**
2.  **Copy Backup Files, Archive Logs, and the Primary's Parameter File to the Standby Server.**
3.  **Create a Standby Control File on the Primary:**
    ```sql
    ALTER DATABASE CREATE STANDBY CONTROLFILE AS '/path/to/standby_control.ctl';
    ```
    Copy this control file to the standby server.
4.  **Create the Standby Initialization Parameter File (`initstandby.ora`).**
5.  **Start the Standby Instance in NOMOUNT using the standby parameter file.**
6.  **Restore the Primary Database Backup on the Standby Server.** Use RMAN in `NOMOUNT` state. You'll likely need to use the `SET NEWNAME` command to adjust the file paths for the single-instance standby.
7.  **Recover the Standby Database:**
    ```sql
    RECOVER STANDBY DATABASE;
    ```

**III. Start Redo Apply on the Standby Database:**

Once the standby database is created (using either method), you need to start the Redo Apply process to keep it synchronized with the primary.

1.  **Mount the Standby Database:**
    ```sql
    SQL> startup mount;
    ```

2.  **Start Managed Recovery Process (MRP):**
    ```sql
    SQL> alter database recover managed standby database using current logfile disconnect;
    ```
    For Real-Time Apply (if licensed), you can use:
    ```sql
    SQL> alter database recover managed standby database using current logfile parallel n sync;
    ```
    where `n` is the number of parallel apply processes.

**IV. Verification and Monitoring:**

1.  **Check the Standby Logs:** On the primary, query `v$ARCHIVED_LOG` to see if logs are being archived and sent to the standby. Check `v$LOG_HISTORY` for applied logs.

2.  **Check Redo Apply Status on the Standby:**
    ```sql
    SQL> select process, status, client_process, sequence#, block#, blocks_applied from v$managed_standby;
    ```
    Look for the `MRP0` process in `APPLYING` state.

3.  **Open the Standby in Read-Only Mode (Optional):** This is a good test to ensure the standby is consistent.
    ```sql
    SQL> alter database open read only;
    SQL> -- Perform read-only queries
    SQL> alter database close;
    SQL> alter database mount standby database;
    ```

4.  **Perform a Switchover Test (Highly Recommended):** This is the best way to verify the Data Guard configuration works as expected.

**Important Considerations:**

* **Licensing:** Ensure you have the necessary Oracle Data Guard licenses.
* **Network Connectivity:** Verify reliable network connectivity between the primary and standby servers.
* **File System/ASM:** Plan your file system or ASM configuration on the standby server. If using ASM, ensure ASM is configured on the standby server.
* **Backup Strategy:** Implement a robust backup strategy for both the primary and standby databases.
* **Documentation:** Document all the steps and configurations.

This comprehensive outline should help you build your Oracle Data Guard configuration. Remember to consult the Oracle Data Guard documentation for your specific Oracle version for detailed information and best practices.
