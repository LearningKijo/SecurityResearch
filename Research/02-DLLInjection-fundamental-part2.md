# DLL Injection fundamental - part2
Hello everyone,

In Part 2, I’ll start writing some code little by little and explore how someone without a coding background can grow and eventually write a DLL injection script to better understand the attack technique known as `DLL injection`

### Two important items
Not to mention, we need Visual Studio (IDE) to write code and build the injector in the end. Based on my experience, I had training in C# five years ago when I was a fresh graduate at my first company, but I gave up coding. Nowadays, thanks to ChatGPT or generative AI, these tools have helped me understand things much faster, search more effectively, and identify my questions, curiosity, and challenges. Therefore, I would say it’s much more approachable than it was five years ago.

- [x] Step 1 : Install [Visual Studio (IDE)](https://visualstudio.microsoft.com/)
- [x] Step 2 : Leverage ChatGPT or another generative AI tools

In this Part 2, as the first step toward writing the injector, what I need to do is write a function to get a process PID before moving on to the injection part. Therefore, I’m planning to cover the following points in Part 2:

1. [The code architecture for GetTargetPid](https://github.com/LearningKijo/SecurityResearch/blob/main/Research/02-DLLInjection-fundamental-part2.md#1-the-code-architecture-for-gettargetpid)
2. [Challenges I faced at each step in C language](https://github.com/LearningKijo/SecurityResearch/blob/main/Research/02-DLLInjection-fundamental-part2.md#2-challenges-i-faced-at-each-step-in-c-language)
3. [Sharing the function result](https://github.com/LearningKijo/SecurityResearch/blob/main/Research/02-DLLInjection-fundamental-part2.md#3-sharing-the-function-result)
4. [Sharing the complete C code](https://github.com/LearningKijo/SecurityResearch/blob/main/Research/02-DLLInjection-fundamental-part2.md#4-sharing-the-complete-c-code)

--- 
## 1. The code architecture for GetTargetPid

```
GetTargetPid()
     │
     ├─▶ 1. Take a snapshot of all running processes
     │      → Use CreateToolhelp32Snapshot()
     │
     ├─▶ 2. Get the first process in the list
     │      → Use Process32FirstW()
     │
     ├─▶ 3. Loop through each process in the list
     │      → Use Process32NextW() in a loop
     │
     ├─▶ 4. Check if the process is "notepad.exe"
     │      → Compare with _wcsicmp()
     │
     ├─▶ 5. If found, save the process ID
     │      → Store procInfo.th32ProcessID
     │
     └─▶ 6. Return the PID (or 0 if not found)
```

## 2. Challenges I faced at each step in C language
These were challenges or new experiences for me, since I’ve mainly been writing attack scripts in PowerShell until now.

I have seen errors because I ...
 - [x] forgot to add headers e.g. `#include <tlhelp32.h>`
 - [x] forgot to define the variables, HANDLE
 - [x] used a wrong filter `wcscmp` vs `_wcsicmp`

I leanred 
 - [x] why I need to do initialization
 - [x] difference between `Process32FirstW` and `Process32FirstA`
 - [x] difference between `Process32FirstW` and `Process32NextW`
 - [x] Filter strings `_wcsicmp`
 - [x] Loop `do.. while`
 - [x] cleanup - `goto`  
 - [x] cleanup - `CloseHandle()`


### 1. Header
When you write `#include <...>`, you're including code that gives your program access to:
- [x] Functions : like printf(), CreateToolhelp32Snapshot()
- [x] Data types : like DWORD, HANDLE, PROCESSENTRY32
- [x] Constants and macros : like TH32CS_SNAPALL

In other word, without the header, you won't be able to use Windows API  you are planning to use in the code. 
```c
#include <stdio.h>    //  Provides standard input/output functions
#include <windows.h>  //  Enables use of Windows API functions
#include <tlhelp32.h> //  Provides functions - CreateToolhelp32Snapshot
```

### 2. Function
This is a function where we write the code to get the Notepad process ID.
Since the function does not take any parameters, we define it with VOID to make that clear.
```c
DWORD 
GetTargetPid(VOID) 
{
  // Write codes to get Notepad process
}
```

### 3. HANDLE & Initialization
Within the function, we try to get a snapshot of all running processes using the Windows API, [CreateToolhelp32Snapshot](https://learn.microsoft.com/en-us/windows/win32/api/tlhelp32/nf-tlhelp32-createtoolhelp32snapshot) - Takes a snapshot of the specified processes, as well as the heaps, modules, and threads used by these processes. It's also important to add a check to see whether the snapshot was successful or not.

To do that, I had to define a variable (HANDLE) and use INVALID_HANDLE_VALUE.
1. **Initialization** : I used INVALID_HANDLE_VALUE to initialize the handle.
Without initializing it, the variable might contain random garbage data, which can lead to unexpected behavior or crashes when the handle is used later.
By setting it to INVALID_HANDLE_VALUE, I can safely check later if the API call failed or not. It’s a safe default value that means: "this handle is not valid yet"

3. **HANDLE** : In Windows programming, when you ask the system to give you access to something like a snapshot of all running processes —
Windows returns a HANDLE. It is a special kind of variable.
It doesn't hold normal numbers or text — instead, it holds a reference to a system resource, like: a running program (process), a file, a thread and a snapshot of all processes

```c
DWORD 
GetTargetPid(VOID) {

	HANDLE hSnapShot = INVALID_HANDLE_VALUE;
	
	// Get a snapshot of all processes in the system
	hSnapShot = CreateToolhelp32Snapshot(TH32CS_SNAPALL, 0);
	if (INVALID_HANDLE_VALUE == hSnapshot) {
		printf("CreateToolhelp32Snapshot failed\n");
		goto Cleanup;
	}
}
```

### 4. Process32FirstW vs Process32FirstA?
[Process32First](https://learn.microsoft.com/en-us/windows/win32/api/tlhelp32/nf-tlhelp32-process32first) retrieves information about the first process found in a system snapshot. While writing it, I noticed Process32FirstW and Process32FirstA. What’s the difference?

The A version uses ANSI (single-byte) encoding, while the W version uses Unicode (UTF-16), which is better for international character support and modern Windows development.

- `Process32FirstA` : ASCII
- `Process32FirstW` : Unicode

This is my memo to look back :)
| #                    | ASCII                             | Unicode                                        |
|:---------------------|:----------------------------------|:-----------------------------------------------|
| Number of Characters | Only 128 characters	           | Supports hundreds of thousands                 |
| Language Support     | English only (A–Z, symbols, etc.) | All languages (Japanese, Chinese, emoji, etc.) |
| Examples             | A, B, 1, ?, @                     | あ, 漢, 글, 😊, 𠮷                             |

### 5. Understanding Process32First and Process32Next
1. Why Can't We Simply Filter Directly from the Snapshot?

	When you use CreateToolhelp32Snapshot to capture a snapshot of all running processes, it gives you a sequential list of processes, not a searchable list or array.

	The snapshot contains the information about all processes, but it is structured in a way that requires you to iterate through the data, checking each process one by one. Unlike high-level languages that have built-in filtering methods (like .filter() in Python), in C, there’s no simple way to search through the snapshot directly.

2. Why Must We Use Process32First and Process32Next?
	- [Process32First](https://learn.microsoft.com/en-us/windows/win32/api/tlhelp32/nf-tlhelp32-process32first) - starts the process iteration by giving you the first process in the snapshot.
	- [Process32Next](https://learn.microsoft.com/en-us/windows/win32/api/tlhelp32/nf-tlhelp32-process32next) - continues from the first and retrieves the next process in the snapshot.

	These two functions are necessary because the snapshot doesn’t allow random access or filtering. You need to go through each process sequentially to find your target process, such as notepad.exe.

	Without these functions, you'd have no way to walk through the list of processes, making them essential for working with the snapshot data.
```
[Snapshot]
 ├─ Process A (explorer.exe)
 ├─ Process B (chrome.exe)
 ├─ Process C (notepad.exe) ←-- ★Target !!
 └─ ...
 ```

### 6. Filter Notepad process 
In PowerShell, I can write something like `if ($procName -eq "notepad.exe")`, which is simple and readable.
Why can’t I do something like if (procName == "notepad.exe") in C language? Why do I have to use `if (0 == _wcsicmp(procName, L"notepad.exe"))`?

In C, string variables like procName are actually pointers to memory (wchar_t*), not full string objects like in PowerShell or Python.
So, using == compares the memory addresses, not the contents of the strings.

To compare the actual string content, you must use a function like _wcsicmp (case-insensitive) or wcscmp (case-sensitive). These functions return 0 when the strings are equal.

### 7. wcscmp vs _wcsicmp
I initially used wcscmp, which is case-sensitive, and I defined the process name as `notepad.exe`.
But the actual process name might be `Notepad.exe` (with uppercase 'N').

Because of the case mismatch, the comparison failed — it didn’t find the process.

**Lesson :**
1. wcscmp() is case-sensitive, so "notepad" ≠ "Notepad".
2. If you want to ignore case, use `_wcsicmp()` instead.
3. Always double-check whether the process name is capitalized or not.

### 8. Cleanup, CloseHandle, goto
  1. **CloseHandle** : When you take a snapshot using CreateToolhelp32Snapshot, it returns a handle to a system resource.
     If you don’t call [CloseHandle()](https://learn.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-closehandle) afterwards, the handle remains open, which can lead to resource leaks or memory issues.
     That’s why the cleanup section is critical — to properly release the snapshot handle and avoid leaking system resources.
  
  2. **goto** : goto in C lets you jump to a specific part of your code using a label. It’s most useful for error handling and cleanup, allowing you to centralize resource release (like closing files). 

```c
	DWORD dwResult = 0; // initialize result variable to 0

	// Get a snapshot of all processes in the system
	hSnapShot = CreateToolhelp32Snapshot(TH32CS_SNAPALL, 0);
	if (INVALID_HANDLE_VALUE == hSnapshot) {
		printf("CreateToolhelp32Snapshot failed\n");
		goto Cleanup;
	}

Cleanup:
	
	if (INVALID_HANDLE_VALUE != hSnapShot) 
	{
		(VOID)CloseHandle(hSnapShot);
		hSnapShot = INVALID_HANDLE_VALUE;
	}
	return dwResult;
```

## 3. Sharing the function result
Let's set a breakpoint at each step we want to confirm in the GetTargetPid function.

1. The first one is that we are looking at `CreateToolhelp32Snapshot`

![image](https://github.com/user-attachments/assets/18a1471c-88ed-4349-8d44-0d9bd8ef0007)

2. Secondly, we can see that `Process32FirstW` is working correctly.
   It retrieves information about the first process in the snapshot and stores the process name (the executable file name) in the szExeFile array.

![image](https://github.com/user-attachments/assets/edd4cdd7-742a-419a-83c7-41706bce7180)

3. Lastly, after the do-while loop with the Notepad string filtering, I was able to retrieve the Notepad process's PID.
   I then compared the PID with the one in Task Manager, and everything looks good.
   
![image](https://github.com/user-attachments/assets/165beb2f-5335-4b88-8808-8d9358d72ed9)

## 4. Sharing the complete C code
```c
#include <stdio.h>    //  Provides standard input/output functions
#include <windows.h>  //  Enables use of Windows API functions
#include <tlhelp32.h> //  Provides functions - CreateToolhelp32Snapshot

#define TARGET_PROCESS L"notepad.exe" //  Target process name to search for

DWORD 
GetTargetPid(VOID) {

	HANDLE hSnapShot = INVALID_HANDLE_VALUE;
	PROCESSENTRY32W procInfo = {0}; // initialize result variable to 0
	DWORD dwResult = 0; // initialize result variable to 0
	
	// Get a snapshot of all processes in the system
	hSnapShot = CreateToolhelp32Snapshot(TH32CS_SNAPALL, 0);
	if (INVALID_HANDLE_VALUE == hSnapshot) {
		printf("CreateToolhelp32Snapshot failed\n");
		goto Cleanup;
	}

	// Set the size of the structure before using it
	// Get the first process information
	procInfo.dwSize = sizeof(PROCESSENTRY32W);
	if (!Process32FirstW(hSnapShot, &procInfo)) {
		printf("Process32FirstW failed\n");
		goto Cleanup;
	}

	// Loop through all processes to find Notepad
	do {
		if (0 == _wcsicmp(procInfo.szExeFile, TARGET_PROCESS))
		{
			dwResult = procInfo.th32ProcessID;
			printf("Found %S with PID: %d\n", TARGET_PROCESS, dwResult);
			break;
		}
	} while (Process32NextW(hSnapShot, &procInfo));

Cleanup:
	
	if (INVALID_HANDLE_VALUE != hSnapShot) 
	{
		(VOID)CloseHandle(hSnapShot);
		hSnapShot = INVALID_HANDLE_VALUE;
	}
	return dwResult;
}

INT 
main()
{
	DWORD dwTargetPid = 0;

	dwTargetPid = GetTargetPid();
	if (0 == dwTargetPid)
	{
		printf("Failed to find target process\n");
		goto Cleanup;
	}

Cleanup:

	return 0;  // Return 0 to indicate successful execution
}
```
---
Updated 23/04/2025 - First version

#### Disclaimer
The views and opinions expressed herein are those of the author and do not necessarily reflect the views of company.
