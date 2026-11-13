# Web Server Attacks - II

**Platform:** TryHackMe  
**Path:** Jr Penetration Tester  
**Module:** Web Application Security Fundamentals  
**Target Technology:** Microsoft IIS / ASP.NET  
**Status:** Completed ✅

---

## 1. Introduction

This room focuses on attacking Microsoft Internet Information Services (IIS), a web server commonly used to host websites, web applications, intranet portals, and REST APIs on Windows Server.

Unlike Apache and Nginx, IIS is deeply integrated with the Windows operating system, Windows authentication, Active Directory, and the .NET framework.

This integration introduces several attack surfaces, particularly when IIS is improperly configured.

### Learning Objectives

- Fingerprint IIS servers and identify their versions.
- Enumerate IIS services and supported HTTP methods.
- Discover hidden files and directories using IIS tilde enumeration.
- Understand and exploit WebDAV misconfigurations.
- Upload and execute ASPX web shells.
- Understand IIS application pool identities and their privileges.
- Automate IIS enumeration using Nmap and other tools.

---

## 2. IIS Fingerprinting and Enumeration

The first step in attacking an IIS server is identifying the web server technology, version, and enabled features.

### 2.1 Identify the Web Server

Using curl:

```bash
curl -I http://TARGET_IP
```

To display only important headers:

```bash
curl -sI http://TARGET_IP | grep -iE 'Server:|X-Powered-By:'
```

Important HTTP response headers:

| Header | Purpose |
|---|---|
| Server | Identifies the web server software and potentially its version |
| X-Powered-By | May reveal the application framework |
| Set-Cookie | May reveal application technologies or session handling |

Example:

```http
Server: Microsoft-IIS/10.0
X-Powered-By: ASP.NET
```

**Observation:** The response indicates that the target is running Microsoft IIS and uses ASP.NET.

Fingerprinting helps determine which technologies, vulnerabilities, and enumeration techniques are relevant.

### 2.2 Enumerating HTTP Methods

HTTP methods define the operations a client can request from a web server.

Use the OPTIONS method:

```bash
curl -X OPTIONS http://TARGET_IP/webdav -sv
```

To filter important information:

```bash
curl -X OPTIONS http://TARGET_IP/webdav -sv 2>&1 | grep -E "Allow:|DAV:"
```

Important HTTP methods:

| Method | Description |
|---|---|
| GET | Retrieves a resource |
| POST | Submits data to the server |
| HEAD | Retrieves response headers |
| OPTIONS | Identifies supported communication methods |
| PUT | Uploads or replaces a resource |
| DELETE | Deletes a resource |
| MOVE | Moves a resource |
| COPY | Copies a resource |
| PROPFIND | Retrieves WebDAV resource properties |
| MKCOL | Creates a WebDAV collection |

Some methods, such as PUT and DELETE, can introduce security risks when improperly configured.

**Important:** A server advertising a method does not automatically mean that an unauthenticated user can successfully execute it.

---

## 3. IIS Tilde (Short Filename) Enumeration

### 3.1 Understanding 8.3 Filename Convention

Windows supports an older filename convention called the 8.3 naming convention.

It allows long filenames to have shorter aliases.

Example:

```text
Long filename: BackupFiles
Short filename: BACKUP~1
```

The general format is:

```text
FIRST6~1.EXT
```

The tilde (`~`) character is important because it is used in the short filename representation.

### 3.2 IIS Tilde Enumeration

IIS can reveal information about files and directories through differences in HTTP responses when specially crafted paths containing the tilde character are requested.

An attacker can use these response differences to determine whether particular filename prefixes exist.

This technique can expose information about:

- Hidden directories.
- Backup folders.
- Configuration files.
- Application directories.
- Other resources that are not easily discoverable through normal browsing.

### 3.3 Manual Enumeration

Example:

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
--path-as-is 'http://TARGET_IP/backup~1/*~1*'
```

Test a nonexistent prefix for comparison:

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
--path-as-is 'http://TARGET_IP/zzzzzz~1/*~1*'
```

The difference between the HTTP responses can provide an enumeration oracle.

**Key takeaway:** Do not assume that a 400 or 404 response means the technique failed. The difference between responses is what matters.

### 3.4 Automated Enumeration

A specialized IIS short filename enumeration script can automate the process.

Example:

```bash
python3 iis_shortname_scan.py TARGET_IP
```

The exact script arguments depend on the tool version.

Automation helps identify potential short filename aliases without manually testing every character.

### 3.5 Accessing Discovered Directories

Once a short directory name is identified, attempt to determine its corresponding long directory name.

For example:

```bash
curl -i http://TARGET_IP/BackupFiles/
```

If directory access is permitted, enumerate the discovered resources.

```bash
curl -i http://TARGET_IP/BackupFiles/webdav_notes.txt
```

**Security impact:** Information disclosure can reveal backup files, internal configuration details, or credentials that may lead to further compromise.

---

## 4. WebDAV Exploitation: Uploading an ASPX Shell

### 4.1 What is WebDAV?

WebDAV (Web Distributed Authoring and Versioning) is an extension of HTTP that enables users to manage files on a remote web server.

It provides additional methods such as:

- PROPFIND
- PROPPATCH
- MKCOL
- PUT
- DELETE
- COPY
- MOVE
- LOCK
- UNLOCK

WebDAV is useful for collaborative file management, but improper access controls can allow attackers to upload or manipulate files.

### 4.2 Identifying WebDAV

Use the OPTIONS method:

```bash
curl -X OPTIONS http://TARGET_IP/webdav -sv
```

Look for response headers such as:

```http
DAV: 1,2,3
MS-Author-Via: DAV
```

These headers indicate WebDAV support.

Check whether PUT is supported:

```http
Allow: OPTIONS, GET, HEAD, PUT, DELETE, PROPFIND
```

The presence of PUT indicates that the server advertises upload functionality.

However, authentication and authorization controls must still be tested.

### 4.3 Testing WebDAV Access

Attempt an unauthenticated upload:

```bash
curl -X PUT http://TARGET_IP/webdav/test.txt \
-d "WebDAV test"
```

Observe the HTTP response.

Common response codes:

| Status Code | Meaning |
|---|---|
| 200 OK | Request succeeded |
| 201 Created | Resource created |
| 204 No Content | Request succeeded without response body |
| 401 Unauthorized | Authentication required or failed |
| 403 Forbidden | Access denied |
| 404 Not Found | Requested resource or endpoint not found |

### 4.4 NTLM Authentication

IIS can use Windows authentication mechanisms such as NTLM and Negotiate.

If valid credentials are discovered during enumeration, they can be tested against the WebDAV endpoint.

Example:

```bash
curl -v --ntlm -u 'USERNAME:PASSWORD' \
-T test.txt http://TARGET_IP/webdav/test.txt
```

Command breakdown:

| Option | Explanation |
|---|---|
| `-v` | Enables verbose output |
| `--ntlm` | Uses NTLM authentication |
| `-u` | Specifies username and password |
| `-T` | Uploads a local file using PUT |

### 4.5 Uploading an ASPX Web Shell

IIS supports ASP.NET applications, including ASPX pages.

If an attacker can upload an executable ASPX file into a directory where ASP.NET execution is enabled, it may be possible to execute commands through the web server.

Example upload:

```bash
curl -v --ntlm -u 'USERNAME:PASSWORD' \
-T cmd.aspx http://TARGET_IP/webdav/cmd.aspx
```

The general attack chain is:

1. Identify the IIS server.
2. Discover WebDAV functionality.
3. Enumerate accessible directories.
4. Identify exposed credentials.
5. Authenticate using NTLM.
6. Upload an ASPX file.
7. Access the uploaded file through the browser or curl.
8. Validate whether server-side execution is possible.

**Important:** A successful file upload does not necessarily mean successful code execution. The uploaded file must also be accessible and processed by an ASP.NET handler.

---

## 5. ASPX Web Shells

### 5.1 What is an ASPX Web Shell?

An ASPX web shell is a server-side script written using ASP.NET technologies that allows an attacker to interact with the underlying server through HTTP requests.

Depending on its implementation and execution permissions, it may allow:

- Executing operating system commands.
- Enumerating directories.
- Reading files.
- Identifying the current user.
- Inspecting system information.
- Interacting with accessible resources.

### 5.2 Understanding the Execution Context

When an ASPX page is executed, IIS processes it through its worker process:

```text
w3wp.exe
```

The process runs under an application pool identity.

This is important because the permissions available to a web shell depend on the identity and privileges of the worker process.

### 5.3 Identifying the Current User

After obtaining authorized command execution, the following Windows commands help identify the execution context.

```cmd
whoami
```

Display user privileges:

```cmd
whoami /priv
```

Display group memberships:

```cmd
whoami /groups
```

Check the hostname:

```cmd
hostname
```

Check operating system information:

```cmd
systeminfo
```

These commands help determine what access the web application process has.

### 5.4 SeImpersonatePrivilege

One important Windows privilege to understand is:

```text
SeImpersonatePrivilege
```

This privilege allows a process to impersonate a client after authentication under specific conditions.

It is particularly relevant when investigating privilege escalation opportunities involving IIS application pool identities.

Certain Potato-family privilege escalation techniques abuse Windows impersonation behavior.

**Key takeaway:** Obtaining command execution through a web shell does not automatically mean SYSTEM-level access. Always identify the current execution identity and its privileges.

---

## 6. IIS Misconfigurations

IIS security depends heavily on proper configuration.

Even when the server is fully patched, configuration weaknesses can expose sensitive resources or provide opportunities for further exploitation.

Important areas to investigate:

### 6.1 Directory Listing

Directory listing occurs when a web server displays the contents of a directory instead of restricting access or serving a default document.

Potentially exposed information:

- Backup files.
- Application source files.
- Configuration files.
- Old versions of applications.
- Sensitive documents.

### 6.2 Sensitive File Exposure

Files such as the following may contain sensitive information:

```text
web.config
config.bak
backup.zip
database.bak
.env
```

The IIS configuration file `web.config` may contain application settings and connection strings.

Access should be restricted appropriately.

### 6.3 Trace Method

HTTP TRACE can reveal request information and may create security concerns depending on the application and surrounding browser security controls.

Check whether it is supported through OPTIONS enumeration.

### 6.4 ASP.NET Diagnostic Features

ASP.NET diagnostic features can expose application information if improperly configured.

For example:

```text
/trace.axd
```

If detailed tracing is exposed publicly, it may reveal:

- Request information.
- Application execution details.
- Session-related information.
- Server-side diagnostic data.

These features should not be publicly accessible in production unless explicitly required and properly protected.

### 6.5 Security Assessment Checklist

- [ ] Check directory listing.
- [ ] Search for exposed backup files.
- [ ] Review accessible configuration files.
- [ ] Identify unnecessary HTTP methods.
- [ ] Check WebDAV authentication and authorization.
- [ ] Verify upload restrictions.
- [ ] Check ASP.NET diagnostic endpoints.
- [ ] Review application pool permissions.
- [ ] Identify exposed server information.

---

## 7. Automating IIS Enumeration

Automation makes enumeration more efficient, especially when assessing multiple systems.

### 7.1 Nmap HTTP Methods Script

Nmap provides an NSE script for identifying supported HTTP methods.

```bash
nmap --script http-methods -p80 TARGET_IP
```

For a more detailed scan:

```bash
nmap -sV --script http-methods -p80 TARGET_IP
```

This can identify supported methods and flag potentially risky methods.

### 7.2 Nmap HTTP Enumeration

A broader HTTP enumeration scan:

```bash
nmap -p80,443 -sV --script http-title,http-headers,http-methods TARGET_IP
```

This helps identify:

- Web server version.
- Page title.
- HTTP response headers.
- Supported HTTP methods.

### 7.3 Manual vs Automated Enumeration

| Manual Enumeration | Automated Enumeration |
|---|---|
| Better understanding of individual responses | Faster across multiple targets |
| Useful for validating findings | Useful for initial discovery |
| Provides more control over requests | Depends on tool capabilities |
| Helps identify unusual behavior | May miss application-specific issues |

**My takeaway:** Automated tools should support manual investigation, not replace it. Understanding the underlying HTTP requests and responses is essential when validating vulnerabilities.

---

## 8. Practical Attack Methodology

The overall methodology covered in this room can be summarized as follows:

```text
Initial Reconnaissance
        |
        v
IIS Fingerprinting
        |
        v
HTTP Method Enumeration
        |
        v
IIS Tilde Enumeration
        |
        v
Hidden Directory Discovery
        |
        v
Information Disclosure
        |
        v
WebDAV Enumeration
        |
        v
Credential Discovery
        |
        v
NTLM Authentication
        |
        v
ASPX File Upload
        |
        v
Web Shell Execution
        |
        v
Execution Context Enumeration
        |
        v
Privilege Assessment
```

Each stage provides information that can help determine the next testing step.

---

## 9. Important Commands Reference

| Purpose | Command |
|---|---|
| Identify IIS headers | `curl -I http://TARGET_IP` |
| Filter server headers | `curl -sI http://TARGET_IP \| grep -iE 'Server:|X-Powered-By:'` |
| Enumerate HTTP methods | `curl -X OPTIONS http://TARGET_IP/webdav -sv` |
| Filter DAV headers | `curl -X OPTIONS http://TARGET_IP/webdav -sv 2>&1 \| grep -E "Allow:|DAV:"` |
| Test PUT | `curl -X PUT http://TARGET_IP/webdav/test.txt -d "test"` |
| NTLM authentication | `curl -v --ntlm -u 'USER:PASS' http://TARGET_IP/webdav/` |
| Upload a file | `curl -v --ntlm -u 'USER:PASS' -T test.txt http://TARGET_IP/webdav/test.txt` |
| Check IIS methods | `nmap --script http-methods -p80 TARGET_IP` |
| HTTP enumeration | `nmap -sV --script http-title,http-headers,http-methods -p80,443 TARGET_IP` |
| Identify current user | `whoami` |
| Check privileges | `whoami /priv` |
| Check groups | `whoami /groups` |

---

## 10. Key Learnings

After completing this room, I gained a better understanding of the Windows-based web server attack surface.

Key concepts learned:

1. **IIS Fingerprinting:** HTTP headers can reveal the web server and application framework.

2. **HTTP Method Enumeration:** OPTIONS requests help identify supported HTTP methods and potential misconfigurations.

3. **IIS Tilde Enumeration:** Differences in HTTP responses can expose information about hidden files and directories.

4. **WebDAV:** Improperly configured WebDAV services can expose file upload and modification functionality.

5. **NTLM Authentication:** Windows authentication can be used to access protected WebDAV resources when valid credentials are available.

6. **ASPX Web Shells:** An improperly protected upload location may allow server-side code execution.

7. **Application Pool Identity:** IIS applications execute under specific Windows identities, making process permissions important during security assessments.

8. **Privilege Enumeration:** Understanding Windows privileges, particularly SeImpersonatePrivilege, is important when assessing post-exploitation opportunities.

9. **Automation:** Nmap NSE scripts can simplify IIS enumeration, but manual verification remains important.

---

## 11. Conclusion

This room helped me understand how to approach the security assessment of Microsoft IIS web servers.

The most important lesson was that web server exploitation is not always about identifying an unpatched vulnerability. Information disclosure, weak authentication configurations, exposed WebDAV functionality, unrestricted file uploads, and excessive application permissions can all contribute to a successful attack.

I also gained practical experience connecting HTTP-level enumeration with Windows authentication and post-exploitation concepts.

**Room Status:** Completed ✅

**Next Focus:** Continue developing practical web application penetration testing skills, particularly authentication testing, access control, file upload vulnerabilities, server-side execution, and Windows-based web infrastructure security.

---

## Disclaimer

All techniques and commands documented here are intended for authorized penetration testing, educational purposes, and controlled lab environments such as TryHackMe. Never perform testing against systems without explicit authorization.
