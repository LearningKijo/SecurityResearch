# DLL Injection fundamental - part1
Hello everyone, I am Kijo Ninja

This is my first note aimed at diving deeper into each attack technique. The content itself might be too simple depending on your intermediate or professional level. 
However, the objective of sharing this is to reflect on my own learning journey and also to support others in the security community who may be facing similar challenges.

### What is DLL Injection?
DLL injection is a technique used to make a running program load and execute code from a Dynamic Link Library (DLL) that it wasn’t originally designed to use.

### How does DLL Injection work?
DLL Injection requires two files:
1. Injector (`injector.exe`)     - This is the tool that injects the DLL into another process
2. Malicious DLL (`malicious.dll`) - This is the payload.

The injector is a custom-made `.exe` file that does not contain the DLL’s logic. Its job is to inject a DLL into another running process.

To do this, the injector:
 - Writes the DLL path (which can be a malicious or harmless DLL) into the memory space of the target process.
 - It then tells the target process to load the DLL by using a Windows function like LoadLibrary.

```
Injector (injector.exe)
    |
    |--> Opens target process
    |--> Allocates memory in target
    |--> Writes DLL path (malicious.dll)
    |--> Creates thread in target to call LoadLibraryA
           |
           --> Target process loads malicious.dll
                  |
                  --> DllMain runs (your payload)
```

**Typical 4-Step API Flow for DLL Injection**
| Steps | Windows API name   | Purpose                         |
|:------|:-------------------|:--------------------------------|
| 1     | OpenProcess        | Obtain a handle to the target process |
| 2     | VirtualAllocEx     | Allocate memory in the target process |
| 3     | WriteProcessMemory | Write the DLL path into the allocated memory |
| 4     | CreateRemoteThread | Start a remote thread to load the DLL |

## Simulation - mavinject.exe and message.dll
Let's try a simple DLL Injection simulation.

This scenario is an example of using mavinject, a legitimate tool from Microsoft Application Virtualization (App-V), to inject a DLL into a running process (Notepad) using PowerShell.

### mavinject.exe?
`mavinject.exe` was a tool included with Microsoft Application Virtualization (App-V),
primarily used as an internal utility to inject DLLs into the processes of virtualized applications.

- To load specific DLLs into virtual processes within an App-V environment. However, in some versions of Window s— especially older ones — `mavinject.exe` could be used for general-purpose DLL injection. This capability led to the tool being abused primarily by attackers and threat actors for malicious purposes.

### Simulation code
You can execute the following code in PowerShell.
```powershell
$mypid = (Start-Process notepad -PassThru).id
mavinject $mypid /INJECTRUNNING "<DLL file path>"
```

> [!Important]
> During this time, I created a simple DLL file called `message.dll` <br> 
> You can use it for your simulation, but **please test it in a sandbox environment just to be safe.**
>
> **Download : [message.dll](https://github.com/LearningKijo/SecurityResearch/blob/main/Simulation/message.dll)**

Let me explain what `message.dll` does when it is loaded.

When this DLL is injected into a process (like Notepad), it immediately shows a message box saying:
- [x] "DLL Injection Successful!"

When the process exits or the DLL is unloaded, it shows another message box:
- [x] "DLL Unloaded!"
      
![image](https://github.com/user-attachments/assets/76726682-3ceb-45d0-a8a3-5f84b61be4db)

![image](https://github.com/user-attachments/assets/585afa01-78b6-4b5a-999a-d09937904b91)

### message.dll 
Basic DLL file example using MessageBox (C Language)
```c
#include <windows.h>
#include <stdio.h>
#include "pch.h"

// This function is automatically called when the DLL is loaded or unloaded by a process.
BOOL APIENTRY DllMain(HMODULE hModule, DWORD ul_reason_for_call, LPVOID lpReserved) {
    if (ul_reason_for_call == DLL_PROCESS_ATTACH) {
        // This block is executed when the DLL is loaded into the process.
        // Show a message box indicating that the DLL was successfully injected.
        MessageBoxW(NULL, L"DLL Injection Successful!", L"DLL Injection Test", MB_OK);

        // Output a debug message indicating success. This can be viewed using debugging tools.
        OutputDebugString(L"DLL Injection Successful!\n");

        // Check if the module handle (hModule) is NULL, which indicates an error in loading the DLL.
        if (hModule == NULL) {
            // Show an error message if the DLL failed to load.
            MessageBoxW(NULL, L"Failed to load DLL!", L"Error", MB_ICONERROR);
            // Output an error message to the debug log.
            OutputDebugString(L"Error: Failed to load DLL!\n");
            // Return FALSE to indicate failure to load the DLL.
            return FALSE;
        }
    }

    else if (ul_reason_for_call == DLL_PROCESS_DETACH) {
        // This block is executed when the DLL is unloaded from the process.
        // Show a message box indicating that the DLL has been successfully unloaded.
        MessageBoxW(NULL, L"DLL Unloaded!", L"DLL Injection Test", MB_OK);

        // Output a debug message indicating the DLL has been unloaded.
        OutputDebugString(L"DLL Unloaded!\n");

        // Optionally, handle cases where the DLL is unloaded unexpectedly.
        if (lpReserved == NULL) {
            // Show an error message if there was an issue during the unload process.
            MessageBoxW(NULL, L"Error occurred during DLL unloading!", L"Unload Error", MB_ICONERROR);
            // Output an error message to the debug log.
            OutputDebugString(L"Error: Issue during DLL unloading!\n");
        }
    }
    // Return TRUE to indicate that the DLL has been loaded or unloaded successfully.
    return TRUE;
}
```

### Microsoft Defender for Endpoint detection
In the case of simulating the above scenario on MDE onboarded device, you might see the following alerts;
- A process was injected with potentially malicious code
- Potential Windows DLL process injection
- Use of living-off-the-land binary to run malicious code

---
#### Reference
1. [MITRE ATT&CK T1055 - Process Injection: Dynamic-link Library Injection](https://attack.mitre.org/techniques/T1055/001/)
2. [Process Injection - Red Canary Threat Detection Report](https://redcanary.com/threat-detection-report/techniques/process-injection/)
3. [Detecting and remediating command and control attacks at the network layer](https://techcommunity.microsoft.com/blog/microsoftdefenderatpblog/detecting-and-remediating-command-and-control-attacks-at-the-network-layer/3650607)

#### Disclaimer
The views and opinions expressed herein are those of the author and do not necessarily reflect the views of company.
