# Procedure for Updating the LINKLIST in z/OS

This document outlines the standard procedures for updating the LINKLIST concatenation in a z/OS environment. It covers both dynamic updates, which take effect immediately, and permanent updates that persist across an Initial Program Load (IPL).

## Overview

The LINKLIST is a list of system and user-defined libraries that the system searches to find and load executable programs. Modifying the LINKLIST is a critical system administration task that requires careful planning and execution.

There are two primary methods for updating the LINKLIST:

*   **Dynamic Update:** Uses the `SETPROG LNKLST` command to make immediate changes without an IPL. This is suitable for testing or temporary modifications.
*   **Permanent Update:** Involves editing the `PROGxx` member in the system's PARMLIB to ensure changes are retained after the next IPL. IBM recommends using `PROGxx` for this purpose.

---

## Dynamic LINKLIST Update (Immediate Effect)

A dynamic update allows you to add or remove datasets from the LINKLIST without requiring a system restart.

**Important Considerations:**

*   **LLA and XCFAS:** The Library Lookaside (LLA) and Cross-System Coupling Facility (XCFAS) hold enqueues (locks) on LINKLIST datasets. To modify an active LINKLIST, you may need to stop LLA, which can temporarily degrade system performance.
*   **Active Jobs:** When a new LINKLIST set is activated, currently running jobs will continue to use the old LINKLIST definition until they complete or are specifically updated.

### Steps for a Dynamic Update

1.  **Display the Current LINKLIST:**
    Before making any changes, it's good practice to view the current LINKLIST configuration.
    ```zoscommand
    D PROG,LNKLST
    ```

2.  **Define a New LNKLST Set:**
    Create a new LNKLST set by copying the currently active one. This provides a safe working copy.
    ```zoscommand
    SETPROG LNKLST,DEFINE,NAME=<new_lnklst_name>,COPYFROM=CURRENT
    ```

3.  **Modify the New LNKLST Set:**
    *   **To Add a Dataset:**
        ```zoscommand
        SETPROG LNKLST,ADD,NAME=<new_lnklst_name>,DSN=<dataset_to_add>,VOLUME=<volume_serial>
        ```
        **Note:** You can control the position of the new dataset using the `ATTOP` or `AFTER=<existing_dataset>` parameters. A search for a module stops at the first match found.

    *   **To Delete a Dataset:**
        ```zoscommand
        SETPROG LNKLST,DELETE,NAME=<new_lnklst_name>,DSN=<dataset_to_delete>
        ```

4.  **Activate the New LNKLST Set:**
    This command makes your newly defined LINKLIST the active one for the system.
    ```zoscommand
    SETPROG LNKLST,ACTIVATE,NAME=<new_lnklst_name>
    ```
    The system will issue message `CSV500I LNKLST SET <new_lnklst_name> HAS BEEN ACTIVATED`.

5.  **Update Address Spaces (Optional but Recommended):**
    To make the new LINKLIST effective for all currently running and future jobs, use the `UPDATE` command.
    ```zoscommand
    SETPROG LNKLST,UPDATE,JOB=*
    ```
    You will see the confirmation message `CSV504I JOB * IS NOW USING THE CURRENT LNKLST SET`.

6.  **Refresh LLA:**
    After modifying the LINKLIST, it is a good practice to refresh the Library Lookaside facility.
    ```zoscommand
    F LLA,REFRESH
    ```

---

## Permanent LINKLIST Update (Effective After IPL)

To ensure your LINKLIST changes are permanent and will be active after a system IPL, you must update the `PROGxx` member in your system's PARMLIB.

### Steps for a Permanent Update

1.  **Identify the Active `PROGxx` Member:**
    Determine which `PROGxx` member is used at IPL. This is typically specified in the `IEASYSxx` member of PARMLIB.

2.  **Edit the `PROGxx` Member:**
    Open the relevant `PROGxx` member in an editor.

3.  **Modify the LNKLST Statements:**
    Add, remove, or reorder the `LNKLST` statements as needed. The order of the `DSN` entries determines the concatenation order.

    **Example `PROGxx` LNKLST Definition:**
    ```
    LNKLST DEFINE NAME(LNKLST01)
      DSN(SYS1.LINKLIB)
      DSN(SYS1.MIGLIB)
      DSN(SYS1.CSSLIB)
      DSN(CEE.SCEELKED)
      DSN(YOUR.NEW.LOADLIB)
    LNKLST ACTIVATE NAME(LNKLST01)
    ```

4.  **Save the Changes:**
    Save the modified `PROGxx` member.

5.  **Activation:**
    The changes will automatically take effect at the next system IPL. To activate the changes without an IPL, you can use the `SET PROG=xx` command, where `xx` is the suffix of your updated `PROGxx` member. This should be followed by the appropriate steps for managing LLA and updating jobs as outlined in the dynamic update procedure.

    ```zoscommand
    SET PROG=xx
    ```

---
### Summary of Key Commands

| Action | Command |
| --- | --- |
| Display LINKLIST Status | `D PROG,LNKLST` |
| Define a New LNKLST Set | `SETPROG LNKLST,DEFINE,NAME=...,COPYFROM=...` |
| Add a Dataset Dynamically | `SETPROG LNKLST,ADD,NAME=...,DSN=...` |
| Delete a Dataset Dynamically | `SETPROG LNKLST,DELETE,NAME=...,DSN=...` |
| Activate a LNKLST Set | `SETPROG LNKLST,ACTIVATE,NAME=...` |
| Update All Jobs | `SETPROG LNKLST,UPDATE,JOB=*` |
| Activate a PARMLIB Member | `SET PROG=xx` |
| Refresh LLA | `F LLA,REFRESH` |
