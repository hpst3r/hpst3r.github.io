---
title: "The Bitlocker Script"
date: 2024-12-16T10:10:10-00:00
draft: false
---

# Problem

Needed to enable BitLocker on lots of machines with varying configurations.

Really didn't want to do it manually, especially since this problem will not go away.

# Solution

Uhh. Couple hours of PowerShell.

It works OK. Couple things I could still improve. I've run this probably a couple thousand times by now. It should be more or less idempotent and plays nicely with common edge cases (does not catastrophically fail.) It will happily encrypt my USB disks since they're just exposed to the system as fixed NVMEs, but that's not the script's fault... as annoying as it can be.

This script is running as an automated task in our RMM to keep clients' machines in their desired state.

Our RMM grabs key protectors reliably, so the AD backup is a safety measure, not the primary method of key retrieval.

This and the above means that it doesn't really matter when the drives are encrypted (we don't need to sit around and wait for contact with a DC), because the machines will probably be able to back up the keys to AD at some point.

Likewise, I use our RMM for monitoring the state of my machines as I'm rolling out drive encryption, so there's no monitoring here.

## Description block

### SYNOPSIS
Enables BitLocker encryption on fixed volumes residing on SSD media when the machine has a TPM active.

### NOTES
BitLocker drive encryption enablement and verification script

Enables BitLocker drive encryption forcibly on fixed SSD volumes with TPM and/or recovery key protectors.

Decrypts and re-encrypts encrypted volumes that lack protection.

Verifies key protectors on volumes that are already encrypted and protected.

If applicable, backs up BitLocker recovery key to Active Directory (stored as a child of the relevant Computer object) or Entra ID.

Intended to fail gracefully and skip trying to encrypt unknown volumes rather than terminating the script.

Consider pairing this with GPO to enforce BitLocker (Computer Configuration\Policies\Administrative Templates\Windows Components\BitLocker Drive Encryption).

Highly recommended to delegate access to BitLocker key management to a service account that will run this script, or use the Computer account (do not use a Domain Admin).

See: https://learn.microsoft.com/en-us/archive/blogs/craigf/delegating-access-in-ad-to-bitlocker-recovery-information

Initial commit: 10/18/2024

Tested on: Windows 11 Pro 23H2, Windows 11 Pro 24H2, Windows 11 Enterprise LTSC 2024 with PowerShell 5.1 & 7.4.5

Requires: elevated local account OR elevated account with permission to modify BitLocker recovery information in AD (if key backup is desired)

### DESCRIPTION
REQUIRES System or Local Administrator privileges to function.

Checks for presence of a TPM. If a TPM is found, we want to encrypt disks.

Iterates through volumes that can be encrypted.

Confirms that PhysicalDisk which is hosting a LogicalDisk which is hosting a Partition is an SSD.

Confirms that Win32_LogicalDisk which is hosting a Partition with MountPoint (drive letter) is of type 3, fixed.

Checks for a relatively common edge case with disk encrypted but protection off. Decrypts disk then re-enables protection.

Confirms that, if BitLocker is enabled, it has a TPM and recovery key protector, and makes it so only those two protectors are active.

If the computer is domain joined, attempts to sync the recovery key to Active Directory. This is possible with the System account. This needs to be allowed by GPOs (it is not enabled by default.)

## Code

Git repo pending... it's in with a mess of other scripts right now.

```PowerShell
#Requires -RunAsAdministrator

BEGIN {

    [int]$EncryptionFailures = 0

    function Sync-KeyProtectors {
        param (
            [System.Collections.ArrayList]$DesiredProtectors = @("Tpm", "RecoveryPassword"),
            [array]$BitLockerVolumes = (Get-BitLockerVolume),
            [bool]$Remove = $true,
            [bool]$RemoveTpm = $false
        )

        :Partition foreach ($Volume in ($BitLockerVolumes)) {
            [int]$Match = 0
            [int]$DesiredMatch = $DesiredProtectors.Count # save this here since I'll be removing objects

            if ($Volume.KeyProtector -contains("ExternalKey")) {
                $DesiredMatch += 1
            }

            :KeyProtector foreach ($KeyProtector in $Volume.KeyProtector) {

                if ($DesiredProtectors.Contains([string]$KeyProtector.KeyProtectorType)) {
                    Write-Host -Object `
                        "Info: $($KeyProtector) was detected on volume $($Volume.MountPoint) and was requested."

                    $DesiredProtectors.Remove([string]$KeyProtector.KeyProtectorType)

                    $Match++
                } # if DesiredProtectors contains KeyProtector
                elseif ($Remove) {
                    if (($Volume.VolumeType -eq "Data") -and ($KeyProtector.KeyProtectorType -eq "ExternalKey")) {
                        Write-Host -Object `
                            "Info: I won't attempt to remove ExternalKey protectors, as they're used for auto unlock."
                        
                        continue :KeyProtector
                    }
                    Write-Host -Object `
                        "Info: Key protector of type $($KeyProtector.KeyProtectorType) on volume $($Volume.MountPoint) is unwanted. Attempting to remove it."
                    try {
                        if ([string]$KeyProtector.KeyProtectorType -eq "Tpm" -and (!$RemoveTpm)) {
                            Write-Host -Object `
                                "Info: Not removing TPM protector, since RemoveTpm is false."
                            continue :KeyProtector
                        }
                        Remove-BitLockerKeyProtector `
                            -KeyProtectorId $KeyProtector.KeyProtectorId `
                            -MountPoint $Volume.MountPoint
                    } # try
                    catch {
                        Write-Error -Message `
                            $_
                        
                        Write-Error -Message `
                            "Error: Unexpected failure in removal of key protector $($KeyProtector.KeyProtectorType) on volume $($Volume.MountPoint) with ID $($KeyProtector.KeyProtectorId)."
                    } # catch Remove-BitLockerKeyProtector
                } # elseif Remove
                else {
                    Write-Host -Object `
                        "Info: Key protector $($KeyProtector) is not desired on volume $($Volume.MountPoint), but removal was not requested. Continuing."
                } # else

            } # foreach KeyProtector

            if ($Match -eq $DesiredMatch) {
                Write-Host -Object `
                    "Info: Desired key protectors on volume $($Volume.MountPoint) match set key protectors. Good!"
                continue
            }
            if ($Match -ne $DesiredMatch) {
                Write-Host -Object `
                    "Info: Desired key protectors on volume $($Volume.MountPoint) do not match set key protectors. We need to register new key protectors. $($Match), $($DesiredMatch)"
                
                Write-Host -Object `
                    "Info: Attempting to add missing key protector(s) of type(s) $($DesiredProtectors) to volume $($Volume.MountPoint)."

                :AddKeyProtector foreach ($KeyProtectorType in $DesiredProtectors) {
                    switch ($KeyProtectorType) {
                        "RecoveryPassword" {
                            try {
                                Add-BitLockerKeyProtector `
                                    -MountPoint $Volume.MountPoint `
                                    -RecoveryPasswordProtector `
                                    -ErrorAction Stop
                            } # try
                            catch {
                                Write-Error -Message `
                                    $_
                                
                                Write-Error -Message `
                                    "Sync-KeyProtectors: Failed to add RecoveryPassword key protector."
                            }
                            continue AddKeyProtector
                        } # RecoveryPassword
                        "Tpm" {
                            if ($Volume.VolumeType -eq "Data") {
                                Write-Host -Object `
                                    "Info: It is not possible to add a TPM protector to a data volume. Skipping this protector."
                                continue AddKeyProtector
                            }
                            try {
                                Add-BitLockerKeyProtector `
                                    -MountPoint $Volume.MountPoint `
                                    -TpmProtector `
                                    -ErrorAction Stop
                            } # try
                            catch {
                                Write-Error -Message `
                                    $_
                                
                                Write-Error -Message `
                                    "Sync-KeyProtectors: Failed to add Tpm key protector."
                            } # catch
                            continue AddKeyProtector
                        } # Tpm
                        default {
                            Write-Error -Message `
                                "I cannot add this type ($($KeyProtectorType)) of key protector. Error."
                            
                            continue AddKeyProtector
                        }
                    } # switch KeyProtectorType
                } 
            }

        } # foreach Partition
    } # function Sync-KeyProtectors
    
} PROCESS {

    # Collect Tpm object for reuse.

    $SystemTpm = Get-Tpm

    # Confirm the system has a TPM present.

    if ($SystemTpm.TpmActivated) {
        Write-Host `
            "TPM found. Continuing."
    } else {
        Write-Host `
            "Info: This system does not have an active TPM. I won't encrypt its disks. Exiting."

        return 0
    }
    
    # Confirm that the TPM is enabled.

    if (-not $SystemTpm.TpmEnabled) {
        Write-Error `
            "Error: TPM is active (exists), but not enabled. I won't encrypt the system's disks. Please enable the TPM and try again. Exiting."

        return 1
    }
    
    # Get a list of partitions with associated drive letters - we can enable BitLocker on these.
    # Iterate through the partitions and determine that we do want to encrypt them.

    :Partition foreach ($Partition in (Get-Partition | Where-Object DriveLetter)) {

        Write-Host -Object `
            "Info: Loop beginning on partition $($Partition.DriveLetter)."
        
        # Get the logical disk that the partition is on

        try {
            $LogicalDisk = (
                Get-Disk `
                    -Number $Partition.DiskNumber `
                    -ErrorAction 'Stop'
            )
        } catch {
            Write-Error -Message `
                "Error: unable to Get a logical disk corresponding to partition $($Partition.DriveLetter). Skipping partition $($Partition.DriveLetter)."
            
            continue Partition
        }

        # Get the physical disk associated with the logical disk by using the serial number, then get the physical disk's MediaType.
        # This may not work with hardware RAID or Storage Spaces, but that's OK; the partition will be skipped if media or SN cannot be determined.
    
        if ($null -eq $LogicalDisk.SerialNumber) {
            Write-Host -Object `
                "Notice: LogicalDisk SerialNumber is null. Something may be wrong, or perhaps this is an array or other form of LV. You are in untested territory. Skipping partition $($Partition.DriveLetter)."

            continue Partition
        }
        
        if (-not $LogicalDisk.SerialNumber) {
            Write-Error -Message `
                "Warning: LogicalDisk SerialNumber not found. Something is moderately wrong! I won't be able to determine media type. Skipping partition $($Partition.DriveLetter)."

            continue Partition
        }
        
        # PartitionMediaType = query physical disk by SN, get MediaType property. This will typically be "HDD" or "SSD."
        # Do not attempt to encrypt this volume if the volume is not on a SSD.
    
        try {
            $MediaType = (Get-PhysicalDisk `
                -SerialNumber $LogicalDisk.SerialNumber `
                -ErrorAction 'Stop'
                ).MediaType
        } catch {
            Write-Error -Message `
                $_

            Write-Error -Message `
                "Error: Unable to map LogicalDisk $($LogicalDisk) to a PhysicalDisk to determine MediaType. Skipping this partition."
            
            continue Partition
        }

        if ($MediaType -ne "SSD") {
            Write-Host -Object `
                "Info: PartitionMediaType is not an SSD. Skipping partition $($Partition.DriveLetter)."

            continue Partition
        }
    
        Write-Host -Object `
            "Info: Detected that drive $($Partition.DriveLetter) is of MediaType SSD. Good! Continuing."
    
        # Check if disk is fixed by querying Win32_LogicalDisk (via wrapper function) for the DeviceId (C:) and using the DriveType property (-eq 3 = fixed)
        # Skip this iteration if the DeviceType is not Fixed.

        $Win32_LogicalDisk = (Get-CimInstance Win32_LogicalDisk |
            Where-Object Name -eq "$($Partition.DriveLetter):")

        if (-not $Win32_LogicalDisk) {
            Write-Error -Message `
                "Error: Unable to find a Win32_LogicalDisk corresponding to our DriveLetter $($DriveLetter). Without this, I cannot determine if this drive is removable. Skipping this partition."
        }

        if ($Win32_LogicalDisk.DriveType -eq 3) {
            Write-Host -Object `
                "Info: Win32_LogicalDisk corresponding to $($Partition.DriveLetter) is of fixed type. Continuing."
        } else {
            Write-Host -Object `
                "Info: Win32_LogicalDisk.DriveType for $($Partition.DriveLetter) on SSD media is not fixed (3), real value $($Win32_LogicalDisk.DriveType). Skipping this partition."
                
            continue Partition
        }

        # get the BitLocker volume object that corresponds to our mount point.
        try {
            $BitLockerVolume = Get-BitLockerVolume `
                -MountPoint $Partition.DriveLetter `
                -ErrorAction 'Stop'
        } catch {
            Write-Error -Message `
                "Error: System partition with drive letter $($Partition) does not seem to have a matching BitLockerVolume object. This is unexpected. Skipping this volume."

            continue Partition
        }

        Write-Host "Info: Polling status of BitLocker volume $($BitLockerVolume.MountPoint)."

        # if the drive is already encrypted, make sure protection is off, and then enable disk encryption.
        # if we can't handle the state of the drive or don't want to mess with the drive in its current state (e.g., the drive is already encrypted and Bitlocker'd) pass.

        switch ($BitLockerVolume.VolumeStatus) {
            "FullyEncrypted" {

                Write-Host -Object `
                "Info: Volume $($BitLockerVolume.MountPoint) is already encrypted."
                # TODO: this means we need to decrypt the drive before we can do anything with it!
                
                switch ($BitLockerVolume.ProtectionStatus) {
                    "On" {
                        # make sure that key protectors are RecoveryKey and Tpm only. Unset others.
                        Write-Host -Object `
                            "Info: Volume $($BitLockerVolume.MountPoint) is already protected with BitLocker."

                        Write-Host -Object `
                            "Info: Proceeding to sync key protectors for volume $($BitLockerVolume.MountPoint)."

                        Sync-KeyProtectors -DesiredProtectors @("Tpm", "RecoveryPassword") -Remove $true -RemoveTpm $false -BitLockerVolumes $BitLockerVolume

                        continue Partition

                    } # ProtectionStatus On
                    "Off" {
                        Write-Host -Object `
                            "Info: Volume $($BitLockerVolume.MountPoint) is encrypted, but BitLocker key protectors are not configured. We need to decrypt the drive before we can do anything with it."

                        Write-Host -Object `
                            "Info: Beginning to decrypt volume $($BitLockerVolume.MountPoint). This may take a while. Please wait..."

                        Disable-BitLocker -MountPoint $($BitLockerVolume.MountPoint)

                        do {
                            Start-Sleep -Seconds 30
                            Write-Progress -Activity `
                                "Waiting for decryption of volume $($BitLockerVolume.MountPoint) to complete... Current status: $((Get-BitLockerVolume -MountPoint $BitLockerVolume.MountPoint).EncryptionPercentage)"
                        } # do
                        until ((Get-BitLockerVolume -MountPoint $BitLockerVolume.MountPoint).VolumeStatus -ne "DecryptionInProgress")

                        Write-Host -Object `
                            "Decryption of $($BitLockerVolume.MountPoint) complete. Continuing."
                        
                        break
                        # this requires decrypting and re-encrypting the drive
                    } # ProtectionStatus Off
                    default {
                        # this is an unknown state that means something is wrong
                    } # default
                } # switch ProtectionStatus
            } "DecryptionInProgress" {
                Write-Error -Message `
                    "Unexpected state. Error. Skipping this partition and logging an error."
                
                $EncryptionFailures++

                continue Partition
            } # DecryptionInProgress
            "EncryptionInProgress" {
                # this is not expected
                # we should verify key protectors here, too
                # the below is stuck here for the moment. IDK where it'll go.
                if ($BitLockerVolume.ProtectionStatus -ne "Off") {
                    Write-Error -Message `
                        "Warning: BitLockerVolume $($BitLockerVolume.MountPoint)'s ProtectionStatus is not Off and the volume is not FullyEncrypted. I am confused. Skipping this iteration; will not attempt to encrypt this volume because current state is not understood. Perhaps this script is already running?"
                    
                    continue Partition
                } # if
                break
            } # EncryptionInProgress
            "FullyDecrypted" {
                Write-Host -Object `
                    "Info: Volume $($BitLockerVolume.MountPoint) is FullyDecrypted. Good! Continuing."
                # we're good to go
                break
            } # FullyDecrypted
            default {
                Write-Error -Message `
                    "Error: Partition $($DriveLetter) VolumeStatus is not FullyDecrypted, EncryptionInProgress, DecryptionInProgress or FullyEncrypted: $($BitLockerVolume.VolumeStatus). I am confused and will skip this partition."

                continue Partition
            } # default
        } # switch
        


        Write-Host -Object `
            "Info: Drive checks passed. Enabling BitLocker encryption for volume $($BitLockerVolume.MountPoint)."

        # if a TPM protector that we request is already configured, enabling BitLocker or attempting to add a key protector will fail. At this point, anything registered is unwanted, so clear it all.
        Write-Host -Object `
            "Info: Syncing key protectors with desired state = @() (none configured)."

        Sync-KeyProtectors -DesiredProtectors @() -Remove $true -RemoveTpm $true -BitLockerVolumes $BitLockerVolume

        try {
            $Result = Enable-BitLocker `
                -MountPoint $BitLockerVolume.MountPoint `
                -UsedSpaceOnly `
                -RecoveryPasswordProtector `
                -SkipHardwareTest `
            
            [string] $AddTpmProtectorInternalError = 'Add-TpmProtectorInternal : The system cannot find the file specified. (Exception from HRESULT: 0x80070002)'
            If ($Result -contains $AddTpmProtectorInternalError) { # handle error where corrupted ReAgent.xml file will break Add-TpmProtectorInternal
                Write-Error -Message `
                    "Warning: Attempting to handle '$($AddTpmProtectorInternalError)' by forcing recreation of C:\Windows\System32\Recovery\ReAgent.xml."

                $ReAgentPath = 'C:\Windows\System32\Recovery\ReAgent.xml'
                Test-Path `
                    -Path $ReAgentPath `
                    -ErrorAction 'Stop'
                
                Move-Item `
                    -Path $ReAgentPath `
                    -Destination "$($ReAgentPath).old"

                try {
                    $Result = Enable-BitLocker `
                        -MountPoint $BitLockerVolume.MountPoint `
                        -UsedSpaceOnly `
                        -RecoveryPasswordProtector `
                        -SkipHardwareTest `
                } catch {
                    Write-Error -Message `
                        'Failed to resolve 0x80070002 error. Setting ReAgent.xml back the way I found it.'
                    
                    Move-Item `
                        -Path "$($ReAgentPath).old" `
                        -Destination $ReAgentPath `
                        -Force

                    throw $_
                } finally {
                    Remove-Item `
                        -Path "$($ReAgentPath).old"
                        -Confirm:$false
                }
            }
            
            Add-BitLockerKeyProtector `
                -MountPoint $BitLockerVolume.MountPoint `
                -TpmProtector

            # confirm that resultant state is EncryptionInProgress. Otherwise, log a critical error.
            if ($Result.VolumeStatus -ne "EncryptionInProgress") {
                throw `
                    "Resultant state was not expected. Volume is not encrypting, instead $($Result.VolumeStatus)"
            } # if
        } # try
        catch {

            Write-Error -Message `
                $_

            Write-Error -Message `
                "Critical: Unexpected failure in drive encryption. Logging a failure and continuing to the next partition."
            
            $EncryptionFailures++
            continue Partition
        } # catch

        do {
            Start-Sleep -Seconds 30
            Write-Progress -Activity `
                "Waiting for encryption of volume $($BitLockerVolume.MountPoint) to complete... Current status: $((Get-BitLockerVolume -MountPoint $BitLockerVolume.MountPoint).EncryptionPercentage)"
        } # do
        until ((Get-BitLockerVolume -MountPoint $BitLockerVolume.MountPoint).VolumeStatus -ne "EncryptionInProgress")

        $BitLockerVolume = Get-BitLockerVolume `
            -MountPoint $BitLockerVolume.MountPoint

        if ($BitLockerVolume.VolumeStatus -ne "FullyEncrypted") {
            throw "BitLocker volume is not FullyEncrypted after encryption. Fatal error."
        }

        Write-Host -Object `
            "Info: Successfully enabled BitLocker on volume $($BitLockerVolume.MountPoint)."

        # refresh object now that drive is encrypted

        if ($BitLockerVolume.VolumeType -ne 'OperatingSystem') {
            Enable-BitLockerAutoUnlock `
                -MountPoint $BitLockerVolume.MountPoint
                -Confirm:$false
        }

    } # foreach Partition

    Write-Host -Object `
    "Info: Checking to see if the machine is domain or Entra joined, so we can back up BitLocker recovery keys."

    # if the computer is domain joined, attempt to back up BitLocker key protectors to AD.

    [bool] $DomainMember = & {
        if ((Get-CimInstance Win32_ComputerSystem).PartOfDomain) {
            Write-Host -Object `
                "Info: This computer is domain joined, according to Win32_ComputerSystem.PartOfDomain."
            return $true
        }
        Write-Host -Object `
            "Info: This machine is not domain joined, according to Win32_ComputrySystem.PartOfDomain."
        return $false
    }

    [bool] $EntraJoined = & {
        if (Test-Path 'HKLM:\SYSTEM\CurrentControlSet\Control\CloudDomainJoin\JoinInfo') {
            Write-Host -Object `
                "Info: This computer is Entra ID joined, according to HKLM:\...\CloudDomainJoin\JoinInfo's existence."
            return $true
        }
        Write-Host -Object `
            "Info: This computer is not Entra ID joined, according to HKLM:\...\CloudDomainJoin\JoinInfo."
        return $false
    }

    :Volume foreach ($BitLockerVolume in Get-BitLockerVolume) {

        # ensure volumes automatically unlock

        if ($BitLockerVolume.VolumeType -ne 'OperatingSystem') {
            Enable-BitLockerAutoUnlock `
                -MountPoint $BitLockerVolume.MountPoint |
            Out-Null
        }

        :KeyProtector foreach ($KeyProtector in (Get-BitLockerVolume -MountPoint $BitLockerVolume.MountPoint).KeyProtector) {
            
            if ([string]$KeyProtector.KeyProtectorType -eq "RecoveryPassword") {

                try {

                    if ($DomainMember) {

                        Write-Host -Object `
                            "Info: Attempting to back up password key protector $($KeyProtector) to Active Directory."
        
                        Backup-BitLockerKeyProtector `
                            -MountPoint $BitLockerVolume.MountPoint `
                            -KeyProtectorId $KeyProtector.KeyProtectorId

                    } # if DomainMember
        
                    if ($EntraJoined) {
                    
                        Write-Host -Object `
                            "Info: Attempting to back up password key protector $($KeyProtector) to Entra ID."
        
                        BackupToAAD-BitLockerKeyProtector `
                            -MountPoint $BitLockerVolume.MountPoint `
                            -KeyProtectorId $KeyProtector.KeyProtectorId
        
                    } # if EntraJoined

                } # try
                catch {

                    Write-Error -Message `
                        "Error: failed to back up password key protector for volume $($BitLockerVolume.MountPoint)."
                    Write-Error $_

                } # catch

            } # if KeyProtectorType -eq RecoveryPassword
            
        } # foreach KeyProtector

    } # foreach Volume

} END {

    if ($EncryptionFailures -eq 0) {
        Write-Host -Object `
            "Info: Execution completed without major errors."

        return 0
    } else {
        Write-Error -Message `
            "Warning: Execution completed with $($EncryptionFailures) failure(s) to encrypt volumes that may be compatible. Please review logged output for more details."
            
        return 1
    }

} # end

# TODO: wrap the whole process in a try/catch block and make sure to
# decrypt drives that we attempted and failed to encrypt/back up?
```