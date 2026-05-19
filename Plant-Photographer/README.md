# Plant Photographer Room Write-up By Inna Gyurova (0x1nnx)
https://tryhackme.com/room/plantphotographer
## **Initial recon**
Upon accessing the target machine, I noticed a simple portfolio website. My first instinct was to inspect the page source to look for hidden directories or sensitive information that might not be visible on the surface. While going through the HTML, I discovered a reference to an internal directory, which suggested that the application might expose additional functionality behind the scenes. There was an admin directory. On the homepage, there was a button allowing users to download the photographer’s resume. This immediately stood out, since file download features often rely on backend logic that can sometimes be manipulated.

![Pasted image 20260517132923.png](https://github.com/0x1nnx/TryHackMe-Write-ups/blob/70e43899c443dd0a787433654fdee3e14e53bee2/Plant-Photographer/images/Pasted%20image%2020260517132923.png)

![Pasted image 20260517132928.png](https://github.com/0x1nnx/TryHackMe-Write-ups/blob/70e43899c443dd0a787433654fdee3e14e53bee2/Plant-Photographer/images/Pasted%20image%2020260517132928.png)
## **Investigating file download functionality**
Instead of just downloading the file, I focused on how the request was being handled. By inspecting the request, I noticed that it was being processed by a server running Werkzeug, which hinted at a Python-based backend (likely Flask). This gave me an idea of how errors might be handled and what kind of information leakage I could expect.

![Pasted image 20260518191605.png](https://github.com/0x1nnx/TryHackMe-Write-ups/blob/70e43899c443dd0a787433654fdee3e14e53bee2/Plant-Photographer/images/Pasted%20image%2020260518191605.png)

Looking deeper into the page source and request patterns, I identified that the file download mechanism relied on an id parameter to fetch files.

![Pasted image 20260517132950.png](https://github.com/0x1nnx/TryHackMe-Write-ups/blob/70e43899c443dd0a787433654fdee3e14e53bee2/Plant-Photographer/images/Pasted%20image%2020260517132950.png)

At this point, I started thinking:

If the server is using this parameter directly, there might be a way to manipulate it or trigger unexpected behavior.

Instead of guessing blindly, I intentionally broke the request. I modified the URL by adding an extra “?”, which is a simple way to disrupt how parameters are parsed. My goal here was to force the server into an error state and observe how it responds. This worked. The server returned an error page, and because it was running Werkzeug in a misconfigured/debug-like state, it exposed sensitive information. That’s how I found the API key.

![Pasted image 20260517133002.png](https://github.com/0x1nnx/TryHackMe-Write-ups/blob/70e43899c443dd0a787433654fdee3e14e53bee2/Plant-Photographer/images/Pasted%20image%2020260517133002.png)
## **Admin section access via URL parsing manipulation**
During analysis of the download functionality, I identified that the application constructs backend requests using a user-controlled server parameter combined with a fixed file path and filename

![Pasted image 20260517133012.png](https://github.com/0x1nnx/TryHackMe-Write-ups/blob/70e43899c443dd0a787433654fdee3e14e53bee2/Plant-Photographer/images/Pasted%20image%2020260517133012.png)

server + "/public-docs-k057230990384293/" + filename

This immediately indicated that the backend is making server-side HTTP requests based on user input, with limited control over the file path structure. The filename itself is derived from the id parameter and forced into a numeric .pdf format, which restricted direct manipulation of the requested file type. Initial interaction with the application showed the request being made to an external storage service secure-file-storage.com:8087. From this, I confirmed that the application relies on a remote file storage backend. I tested whether the server parameter could be redirected to internal services. By setting the server value to localhost:8087, I observed that the application successfully accessed the same file, confirming that the backend request is not restricted to external hosts and can target internal services. The /admin endpoint was described as being accessible only from localhost. This suggested a potential internal-only interface. Since the application allowed control over the backend request target, I attempted to combine the internal service access with the admin route.

I tried this payload:

download?server=localhost:8087/admin%23&id=75482342

This caused the backend to resolve the request to the internal admin endpoint instead of the forced /public-docs-k057230990384293/ path, allowing access to the protected file and revealing the flag.

![Pasted image 20260517133038.png](https://github.com/0x1nnx/TryHackMe-Write-ups/blob/70e43899c443dd0a787433654fdee3e14e53bee2/Plant-Photographer/images/Pasted%20image%2020260517133038.png)

## Debug console discovery and initial exploration of PIN generation inputs
To enumerate additional attack surface, I performed directory bruteforcing against the web application. During enumeration, I discovered an exposed Werkzeug debugging console. It was protected by a PIN, indicating that interactive code execution was restricted unless the correct debugger PIN could be obtained. However, the presence of the console itself confirmed that the application was running in debug mode, which significantly increased the amount of information leaked through error messages and stack traces.

![Pasted image 20260517133055.png](https://github.com/0x1nnx/TryHackMe-Write-ups/blob/70e43899c443dd0a787433654fdee3e14e53bee2/Plant-Photographer/images/Pasted%20image%2020260517133055.png)

![Pasted image 20260517133058.png](https://github.com/0x1nnx/TryHackMe-Write-ups/blob/70e43899c443dd0a787433654fdee3e14e53bee2/Plant-Photographer/images/Pasted%20image%2020260517133058.png)

To understand how the debugger protection works, I reviewed external documentation on Werkzeug debug mode behavior. From this research, I learned that the Werkzeug debugger PIN is not random. It is generated from a combination of runtime environment values and application metadata. The purpose is to restrict access to the interactive debug console while still allowing local development usage. The construction logic combines both public and private system attributes, which are then hashed to produce the final PIN. The public values are typically recoverable through application exposure, stack traces, or Python introspection. These values are username, modname, application name, application file path. The private values depend on the host environment and are harder to obtain without system-level access. These values are MAC address and machine identifier.

After understanding the PIN construction model, I attempted to recover the required public and private attributes through application behavior and filesystem access. 

For the username, msfconsole gave me root by default as an option so I left it like that and later I saw that it worked with root.

The application path was leaked from the errors earlier.

![Pasted image 20260519154844.png](https://github.com/0x1nnx/TryHackMe-Write-ups/blob/70e43899c443dd0a787433654fdee3e14e53bee2/Plant-Photographer/images/Pasted%20image%2020260519154844.png)

To obtain the machine ID, I experimented with multiple file access approaches using URL-based file schemes. An initial attempt targeting /etc/machine-id did not succeed through the exposed endpoint. Further testing focused on alternative Linux runtime locations. I eventually retrieved the machine identifier from file:///proc/self/cgroup

For the MAC address component, I tested standard network interface locations under Linux. Since eth0 is commonly used as a default interface name, I attempted to access file:///sys/class/net/eth0/address

![Pasted image 20260517193903.png](https://github.com/0x1nnx/TryHackMe-Write-ups/blob/70e43899c443dd0a787433654fdee3e14e53bee2/Plant-Photographer/images/Pasted%20image%2020260517193903.png)

After obtaining all the information that is needed for PIN reconstruction I used the multi/http/werkzeug_debug_rce module in msfconsole because it is used for PIN reconstruction and console access when given the needed values.

![Pasted image 20260518191414.png](https://github.com/0x1nnx/TryHackMe-Write-ups/blob/70e43899c443dd0a787433654fdee3e14e53bee2/Plant-Photographer/images/Pasted%20image%2020260518191414.png)

After successfully getting the PIN, I gained access to the Werkzeug interactive console. I executed basic Python to enumerate the application filesystem and that's how I found the last flag. Setting the LHOST option and running commands directly from meterpreter is a valid approach as well.

![Pasted image 20260517202640.png](https://github.com/0x1nnx/TryHackMe-Write-ups/blob/70e43899c443dd0a787433654fdee3e14e53bee2/Plant-Photographer/images/Pasted%20image%2020260517202640.png)

## Key Takeaways
This room focuses on how insecure development and deployment practices can expose an application to full compromise even without a traditional vulnerability such as direct command injection.

The main concept behind the challenge is attack chaining. Individually, the exposed behaviors may appear limited: a backend request parameter, verbose error handling, localhost-only functionality and a protected debug console. However, when combined, they create a realistic exploitation path that leads from information disclosure to authenticated code execution.

The challenge also demonstrates why debug features should never be exposed in production environments. Werkzeug debug mode leaked internal paths, runtime behavior, stack traces and enough environmental information to assist in reconstructing the debugger PIN. Once authenticated, the debug console effectively became a remote Python execution interface inside the running Flask application.

Another important aspect of the room is understanding how attackers use observation and experimentation rather than relying only on predefined payloads. Much of the exploitation process involved analyzing application behavior, intentionally triggering errors, testing assumptions about request parsing, and reconstructing how the backend handled user-controlled input.
