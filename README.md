<img width="400" src="https://github.com/user-attachments/assets/44bac428-01bb-4fe9-9d85-96cba7698bee" alt="Tor Logo with the onion and a crosshair on it"/>

# Threat Hunt Report: Unauthorized TOR Usage
- [Scenario Creation](https://github.com/KennyjonesSec-Ops/threat-hunting-scenario-tor-/blob/main/threat-hunting-scenario-tor-event-creation.md)

## Platforms and Languages Leveraged
- Windows 10 Virtual Machines (Microsoft Azure)
- EDR Platform: Microsoft Defender for Endpoint
- Kusto Query Language (KQL)
- Tor Browser

##  Scenario

Management suspects that some employees may be using TOR browsers to bypass network security controls because recent network logs show unusual encrypted traffic patterns and connections to known TOR entry nodes. Additionally, there have been anonymous reports of employees discussing ways to access restricted sites during work hours. The goal is to detect any TOR usage and analyze related security incidents to mitigate potential risks. If any use of TOR is found, notify management.

### High-Level TOR-Related IoC Discovery Plan

- **Check `DeviceFileEvents`** for any `tor(.exe)` or `firefox(.exe)` file events.
- **Check `DeviceProcessEvents`** for any signs of installation or usage.
- **Check `DeviceNetworkEvents`** for any signs of outgoing connections over known TOR ports.

---

## Steps Taken

### 1. Searched the `DeviceFileEvents` Table

Searched for any file that had the string "tor" in it and discovered what looks like the user "employee" downloaded a TOR installer, did something that resulted in many TOR-related files being copied to the desktop, and the creation of a file called `tor-shopping-list.txt` on the desktop at `2025-02-02T20:50:28.0387896Z`. These events began at `2025-02-02T20:02:31.3710543Z`.

**Query used to locate events:**

```kql
DeviceFileEvents  
| where DeviceName == "project-threat-"  
| where InitiatingProcessAccountName == "labuser"  
| where FileName contains "tor"  
| where Timestamp >= datetime(2025-02-02T20:02:31.3710543Z)  
| order by Timestamp desc  
| project Timestamp, DeviceName, ActionType, FileName, FolderPath, SHA256, Account = InitiatingProcessAccountName
```
![image](https://github.com/user-attachments/assets/f884d807-ca33-46ac-91aa-583e8a21b76b)


---

### 2. Searched the `DeviceProcessEvents` Table

Searched for any `ProcessCommandLine` that contained the string "tor-browser-windows-x86_64-portable-14.0.4.exe". Based on the logs returned, at `2025-02-02T20:02:40.0812959Z`, an employee whose account is called "labuser" on the "project-threat-" device ran the file `tor-browser-windows-x86_64-portable-14.0.4.exe` from their Downloads folder, using a command that triggered a silent installation.

**Query used to locate event:**

```kql

DeviceProcessEvents  
| where DeviceName == "project-threat-"  
| where ProcessCommandLine contains "tor-browser-windows-x86_64-portable-14.0.4.exe"  
| project Timestamp, DeviceName, AccountName, ActionType, FileName, FolderPath, SHA256, ProcessCommandLine
```
![image](https://github.com/user-attachments/assets/c2ffe786-eba0-46d5-9f70-9be3056525c0)

---

### 3. Searched the `DeviceProcessEvents` Table for TOR Browser Execution

Searched for any indication that user "labuser" actually opened the TOR browser. There was evidence that they did open it at `2025-02-02T20:34:40.4892778Z`. There were several other instances of `firefox.exe` (TOR) as well as `tor.exe` spawned afterwards.

**Query used to locate events:**

```kql
DeviceProcessEvents  
| where DeviceName == "project-threat-"  
| where FileName has_any ("tor.exe", "firefox.exe", "tor-browser.exe")  
| project Timestamp, DeviceName, AccountName, ActionType, FileName, FolderPath, SHA256, ProcessCommandLine  
| order by Timestamp desc
```
![image](https://github.com/user-attachments/assets/b01c0f88-24d9-48e4-8658-e72d06cec525)


---

### 4. Searched the `DeviceNetworkEvents` Table for TOR Network Connections

Searched for any indication the TOR browser was used to establish a connection using any of the known TOR ports. At `2025-02-02T20:34:51.7031768Z`, an employee on the "project-threat" device successfully established a connection to the remote IP address `127.0.0.1` on port `9150`. The connection was initiated by the process `firefox.exe`, located in the folder `c:\users\labuser\desktop\tor browser\browser\torbrowser\tor\tor.exe`. There were a couple of other connections to site(s) over port `443`.

**Query used to locate events:**

```kql
DeviceNetworkEvents  
| where DeviceName == "project-threat-"  
| where InitiatingProcessAccountName != "system"  
| where InitiatingProcessFileName in ("tor.exe", "firefox.exe")  
| where RemotePort in ("9001", "9030", "9040", "9050", "9051", "9150", "80", "443")  
| project Timestamp, DeviceName, InitiatingProcessAccountName, ActionType, RemoteIP, RemotePort, RemoteUrl, InitiatingProcessFileName, InitiatingProcessFolderPath  
| order by Timestamp desc
```
![image](https://github.com/user-attachments/assets/7967784a-b47a-471e-aeb5-8bbffef5e950)


---

## Chronological Event Timeline 

### 1. File Download - TOR Installer

- **Timestamp:** `2025-02-02T20:02:31.3710543Z`
- **Event:** The employee "labuser" downloaded a file named `tor-browser-windows-x86_64-portable-14.0.1.exe` to the Downloads folder.
- **Action:** File download detected.
- **File Path:** `C:\Users\Labuser\Downloads\tor-browser-windows-x86_64-portable-14.0.4.exe`

### 2. Process Execution - TOR Browser Installation

- **Timestamp:** `2025-02-02T20:02:40.0812959Z`
- **Event:** The user "employee" executed the file `tor-browser-windows-x86_64-portable-14.0.1.exe` in silent mode, initiating a background installation of the TOR Browser.
- **Action:** Process creation detected.
- **Command:** `tor-browser-windows-x86_64-portable-14.0.4.exe /S`
- **File Path:** `C:\Users\Labuser\Downloads\tor-browser-windows-x86_64-portable-14.0.4.exe`

### 3. Process Execution - TOR Browser Launch

- **Timestamp:** `2025-02-02T20:34:40.4892778Z`
- **Event:** User "employee" opened the TOR browser. Subsequent processes associated with TOR browser, such as `firefox.exe` and `tor.exe`, were also created, indicating that the browser launched successfully.
- **Action:** Process creation of TOR browser-related executables detected.
- **File Path:** `C:\Users\employee\Desktop\Tor Browser\Browser\TorBrowser\Tor\tor.exe`

### 4. Network Connection - TOR Network

- **Timestamp:** `2025-02-02T20:34:51.7031768Z`
- **Event:** A network connection to IP `212.132.108.187` on port `443` by user "employee" was established using `tor.exe`, confirming TOR browser network activity.
- **Action:** Connection success.
- **Process:** `tor.exe`
- **File Path:** `c:\users\employee\desktop\tor browser\browser\torbrowser\tor\tor.exe`

### 5. Additional Network Connections - TOR Browser Activity

- **Timestamps:**
  - `2025-02-02T20:34:51.706405Z` - Connected to `5.181.77.8` on port `443`.
  - `2025-02-02T20:34:51.7031768Z` - Connected to `212.132.108.187` on port `443`.
- **Event:** Additional TOR network connections were established, indicating ongoing activity by user "employee" through the TOR browser.
- **Action:** Multiple successful connections detected.

### 6. File Creation - TOR Shopping List

- **Timestamp:** `2025-02-02T20:50:28.0387896Z`
- **Event:** The user "employee" created a file named `tor-shopping-list.txt` on the desktop, potentially indicating a list or notes related to their TOR browser activities.
- **Action:** File creation detected.
- **File Path:** `C:\Users\employee\Desktop\tor-shopping-list.txt`

---

## Summary

The user "employee" on the "threat-hunt-lab" device initiated and completed the installation of the TOR browser. They proceeded to launch the browser, establish connections within the TOR network, and created various files related to TOR on their desktop, including a file named `tor-shopping-list.txt`. This sequence of activities indicates that the user actively installed, configured, and used the TOR browser, likely for anonymous browsing purposes, with possible documentation in the form of the "shopping list" file.

---

## Response Taken

TOR usage was confirmed on the endpoint `project-threat-` by the user `labuser`. The device was isolated, and the user's direct manager was notified.

---
