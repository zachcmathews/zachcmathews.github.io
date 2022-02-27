---
layout: post
title: Batch Updating Software on Active Directory Managed Computers using PowerShell
categories: [Active Directory, PowerShell, WinRM]
---

In order to expedite Revit updates across domain-managed computers, PowerShell is used
to copy and invoke an executable on office workstations using WinRM. WinRM and its 
associated ports are then shutdown by group policy  and the workstations are restarted.
Updates are only performed if the registry key associated with Revit 2021 is present and
the registry key for the update is present.

### Checking for the registry keys
### Enabling WinRM
### Copying the update executable
### Invoking the update executable
### Removing the update executable
### Disabling WinRM
### Checking for the registry keys (to verify update succeeded)
