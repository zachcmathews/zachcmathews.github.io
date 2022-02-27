---
layout: post
title: Batch Updating Software on Active Directory Managed Computers using PowerShell
categories: [Active Directory, PowerShell, WinRM]
---

In order to expedite Revit updates across domain-managed computers, PowerShell is used
to copy and invoke an executable on office workstations using WinRM. WinRM and its 
associated ports are then shutdown by group policy  and the workstations are restarted.
Updates are only performed if the registry key associated with Revit 2021 is present and
the registry key for the update is not present.

### Checking for the registry keys
The following lines assume the remote computer is available and the variables $ComputerName,
$SoftwareKey, and $SoftwareUpdateKey are already defined, where $ComputerName is the AD
name of the remote computer, $SoftwareKey is the fully qualified path to the software
registry key, and $SoftwareUpdateKey is the fully qualified path to the software update.

```powershell
$Registry = [Microsoft.Win32.RegistryKey]::OpenRemoteBaseKey('LocalMachine', $ComputerName)
$HasSoftwareKey = $Registry.OpenSubKey($SoftwareKey) -ne $null
$HasSoftwareUpdateKey = $Registry.OpenSubKey($SoftwareUpdateKey) -ne $null
```

### Enabling WinRM
WinRM must be enabled on the remote computer to invoke an executable using Invoke-Command.
This is accomplished by adding the computer to an active directory security group with
the required GPO already configured and linked. For instructions on how to do that, I found
the following article very helpful:
[Adam the Automator: How to Enable PSRemoting (Locally and Remotely)](https://adamtheautomator.com/enable-psremoting).

The following lines assume the variables $ComputerName and $EnabledWinRMGroup are already
defined, where $EnabledWinRMGroup is the distinguished name of the active directory group
with WinRM configured.

```powershell
Add-ADGroupMember -Confirm:$false -Identity $EnabledWinRMGroup -Members $ComputerName
Restart-Computer -ComputerName $ComputerName -Force -Wait -For WinRM
```

### Copying the update executable
We must now copy the update executable that we wish to run on the remote machine using
WinRM. This circumvents all of the issues involved with double-hop credential passing.

The following lines assume the variables $NetworkPath and $SoftwareUpdatePath are already
defined, where $NetworkPath is the UNC path where the update executable should be copied to
on the remote computer and $SoftwareUpdatePath is the path where the update executable is
currently located on the local computer or on a file server.

```powershell
$null = New-Item -Path $NetworkPath -Force
Copy-Item -Path $SoftwareUpdatePath -Destination $NetworkPath -Force
```

### Invoking the update executable
Now with WinRM enabled and the update executable hosted on the remote machine, we can tell
the remote computer to run the executable and wait for a set amount of time for the update
to complete.

Note if the update executable exited automatically after completing, the timeout would not
be required.

The following lines assume the variables $Credential, $ComputerName, $LocalSoftwareUpdatePath,
and $SoftwareUpdateTimeout are already defined, where $Credential is administrator account
credentials (obtained using Get-Credential command), $LocalSoftwareUpdatePath is the path to
the update executable from the remote machine's perspective, and $SoftwareUpdateTimeout is the
number of seconds to wait for the update to complete before killing the process automatically.

```powershell
Invoke-Command -Credential $Credential -ComputerName $ComputerName -ScriptBlock {
	Start-Process -FilePath $LocalSoftwareUpdatePath
	Start-Sleep -Seconds $SoftwareUpdateTimeout
}
```

### Removing the update executable
To save disk space on the workstation, we can now remove the update executable from the computer.

```powershell
Remove-Item -Path $NetworkPath -Force
```

### Disabling WinRM
In the interest of security, the WinRM service can now be disabled on the remote computer and
the corresponding ports can be closed in the Windows firewall settings. This is accomplished 
by using another group policy object and security group.

The following lines assume the variables $ComputerName, $EnabledWinRMGroup, $DisabledWinRMGroup,
and $RestartTimeout are already defined, where $DisabledWinRMGroup is the distinguished name of 
the active directory group with WinRM and associated ports disabled and $RestartTimeout is the
number of seconds to wait for the computer to restart before checking if the software update
registry key now exists.

```powershell
Remove-ADGroupMember -Confirm:$false -Identity $EnabledWinRMGroup -Members $ComputerName
Add-ADGroupMember -Confirm:$false -Identity $DisabledWinRMGroup -Members $ComputerName
Restart-Computer -ComputerName $ComputerName
Start-Sleep -Seconds $RestartTimeout
```

### Checking for the registry keys (to verify update succeeded)
After restarting the computer, we can notify the user whether the update was successful
or not by checking the registry for the software update key once again.

```powershell
$Registry = [Microsoft.Win32.RegistryKey]::OpenRemoteBaseKey('LocalMachine', $ComputerName)
$HasSoftwareKey = $Registry.OpenSubKey($SoftwareKey) -ne $null
$HasSoftwareUpdateKey = $Registry.OpenSubKey($SoftwareUpdateKey) -ne $null
```
