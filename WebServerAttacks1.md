
# TryHackMe - Web Server Attacks - I

## Room Overview

This room focuses on understanding common web server attacks and security misconfigurations. It explores how attackers enumerate web servers, identify exposed resources, discover sensitive information, and take advantage of poorly configured services.

**Platform:** TryHackMe  
**Room:** Web Server Attacks - I  
**Category:** Web Security  
**Difficulty:** Beginner / Intermediate  
**Status:** Completed  
**Date:** November 12, 2026

---

## 1. Introduction to Web Server Attacks

A web server receives HTTP/HTTPS requests from clients and responds with web pages, files, or API data.

Examples of commonly used web servers:

- Apache HTTP Server
- Nginx
- Microsoft IIS
- Node.js Express

Web server attacks often target:

- Misconfigured services
- Exposed directories and files
- Sensitive information disclosure
- Default configurations
- Debugging endpoints
- Outdated software
- Weak access controls

The objective of web server enumeration is to understand how the server is configured and identify potential security weaknesses.

---

## 2. Web Server Fingerprinting

Web server fingerprinting is the process of identifying the web server software and its version.

This information helps an attacker understand the server environment and identify possible vulnerabilities or misconfigurations.

### Using cURL

```bash
curl -I http://TARGET_IP
```

The `-I` option sends an HTTP HEAD request and displays the response headers.

Example response:

```http
HTTP/1.1 200 OK
Server: Apache/2.4.52
Content-Type: text/html
```

The `Server` header may reveal the web server software and version.

### Checking HTTP Response Headers

```bash
curl -v http://TARGET_IP
```

The `-v` option enables verbose output, displaying request and response details.

**Important:** Server headers can be removed, modified, or misleading. They should not be treated as definitive proof of the software version.

---

## 3. Python HTTP Server

Python provides a simple HTTP server that can be started using:

```bash
python3 -m http.server 8000
```

This starts an HTTP server on port `8000` and serves files from the current directory.

To access it:

```text
http://TARGET_IP:8000
```

### Security Risks

If a developer starts a temporary HTTP server in a directory containing sensitive files, those files may become accessible to other users who can reach the service.

Potentially exposed information includes:

- Configuration files
- Backup files
- Source code
- Credentials
- Internal documents

**Security takeaway:** Temporary development servers should not expose sensitive directories or be accessible to untrusted networks.

---

## 4. Apache Web Server

Apache is a widely used open-source web server.

Some common security issues include:

- Directory listing enabled
- Exposed server status pages
- Backup files accessible through the browser
- Sensitive files stored in the web root
- Incorrect access permissions

### Directory Listing

Directory listing allows visitors to view the contents of a directory when no default index page is available.

For example:

```text
http://TARGET_IP/uploads/
```

If directory listing is enabled, the server may display files and folders.

This could expose:

- Backup files
- Uploaded documents
- Source code
- Configuration files

**Security impact:** Information disclosure.

### Apache Server Status

Apache provides a module called `mod_status` that can display server activity and performance information.

A commonly used endpoint is:

```text
/server-status
```

Check whether it is accessible:

```bash
curl -i http://TARGET_IP/server-status
```

If improperly exposed, it may reveal information about:

- Active requests
- Client connections
- Requested resources
- Server activity

**Security impact:** Information disclosure and potential exposure of internal activity.

### Security Recommendation

- Disable directory listing unless required.
- Restrict access to `/server-status`.
- Avoid storing sensitive files inside the web root.
- Keep Apache updated.
- Apply proper file permissions.

---

## 5. Node.js Express Web Server

Express is a popular web framework for building applications using Node.js.

A default Express application may reveal information through HTTP response headers.

For example:

```http
X-Powered-By: Express
```

This header indicates that the application may be running on Express.

### Information Disclosure

Applications may unintentionally expose:

- Framework information
- Debugging information
- Stack traces
- Environment variables
- Internal application paths

### Debug Endpoints

Developers sometimes create debugging endpoints during development.

Examples of potentially risky endpoints:

```text
/debug
/status
/config
/env
```

These are examples of common endpoint names, not guaranteed endpoints on every application.

If a debug endpoint exposes sensitive application information, an attacker may use it to understand the internal application environment.

### Security Recommendation

- Disable unnecessary framework-identification headers.
- Remove development-only endpoints from production.
- Avoid exposing stack traces.
- Never expose environment variables or secrets.
- Apply authentication and access controls to administrative endpoints.

---

## 6. Nginx Web Server

Nginx is commonly used as a web server, reverse proxy, and load balancer.

Security issues can occur when Nginx is incorrectly configured.

### Directory Listing

Nginx supports directory listing through the `autoindex` directive.

Example configuration:

```nginx
location /files/ {
    autoindex on;
}
```

When enabled, visitors may be able to browse files in the configured directory.

### Nginx Status Module

Nginx can provide server statistics through the `stub_status` module.

Example configuration:

```nginx
location /nginx_status {
    stub_status;
}
```

If this endpoint is publicly accessible, it may expose server statistics.

### Security Recommendation

- Disable directory listing when unnecessary.
- Restrict access to status endpoints.
- Avoid exposing internal application information.
- Review Nginx configuration regularly.

---

## 7. Security Headers

HTTP security headers help browsers enforce additional security protections.

Some important security headers include:

| Header | Purpose |
|---|---|
| Content-Security-Policy | Helps reduce certain XSS and content injection risks |
| X-Frame-Options | Controls whether a page can be embedded in frames |
| Strict-Transport-Security | Enforces HTTPS connections |
| X-Content-Type-Options | Helps prevent MIME-type sniffing |
| Referrer-Policy | Controls how much referrer information is shared |

### Checking Security Headers

```bash
curl -I http://TARGET_IP
```

Review the response headers to identify which security headers are present.

**Important:** Missing security headers are not automatically critical vulnerabilities. Their impact depends on the application and its threat model.

---

## 8. Nikto - Web Server Scanner

Nikto is an open-source web server scanning tool.

It can help identify:

- Dangerous files
- Outdated server software
- Default files
- Misconfigurations
- Commonly exposed resources
- Potentially vulnerable server components

### Basic Usage

```bash
nikto -h http://TARGET_IP
```

To scan a specific port:

```bash
nikto -h http://TARGET_IP:8080
```

Nikto can generate useful findings, but its results require manual verification.

**Important:** Automated scanners may generate significant traffic and trigger security monitoring or rate-limiting controls. Only scan systems where you have authorization.

---

## 9. Web Server Enumeration Workflow

A basic web server enumeration workflow:

### Step 1: Identify Open Ports

```bash
nmap -sV -p 80,443,8080 TARGET_IP
```

- `-sV`: Attempts to identify service versions.
- `-p`: Specifies the ports to scan.

### Step 2: Identify Server Information

```bash
curl -I http://TARGET_IP
```

Review the response headers.

### Step 3: Inspect the HTTP Response

```bash
curl -v http://TARGET_IP
```

Observe the HTTP status code, response headers, and server behavior.

### Step 4: Check Commonly Exposed Resources

Manually inspect relevant paths, such as:

```text
/server-status
/nginx_status
/robots.txt
```

These are examples to investigate, not endpoints that every server will have.

### Step 5: Perform Vulnerability Scanning

```bash
nikto -h http://TARGET_IP
```

Review the results and manually validate interesting findings.

### Step 6: Document Findings

For every finding, record:

- Affected URL or endpoint
- Description of the issue
- Evidence
- Potential impact
- Recommended remediation

---

## 10. Common Web Server Misconfigurations

| Misconfiguration | Potential Impact |
|---|---|
| Directory listing enabled | Information disclosure |
| Exposed server status page | Internal server information disclosure |
| Exposed backup files | Source code or sensitive data disclosure |
| Debug endpoint accessible | Application information disclosure |
| Verbose error messages | Internal paths and technical details exposed |
| Outdated server software | Potential exposure to known vulnerabilities |
| Missing security headers | Reduced browser-side security protections |

---

## 11. Practical Security Testing Checklist

When performing an authorized web server assessment, consider checking:

- [ ] Identify the web server and technology.
- [ ] Identify open HTTP/HTTPS ports.
- [ ] Review HTTP response headers.
- [ ] Check for directory listing.
- [ ] Look for exposed status endpoints.
- [ ] Check for publicly accessible backup files.
- [ ] Review error messages for information disclosure.
- [ ] Identify unnecessary debugging endpoints.
- [ ] Review security headers.
- [ ] Perform an authorized vulnerability scan.
- [ ] Manually validate findings.
- [ ] Document evidence and remediation.

---

## 12. Key Takeaways

From this room, I learned the importance of understanding web server configurations and how seemingly small configuration mistakes can expose sensitive information.

Important concepts:

1. Web server fingerprinting helps identify the underlying server technology.
2. HTTP response headers can reveal information about the server.
3. Directory listing can expose files that should not be publicly accessible.
4. Apache and Nginx provide status modules that should be properly restricted.
5. Node.js Express applications may expose framework and debugging information.
6. Security headers provide additional browser-side protections.
7. Nikto can automate basic web server security checks.
8. Automated scanner findings must be manually validated.

---

## 13. Personal Learning Notes

**What I learned:**

- How to inspect HTTP response headers using cURL.
- How web server fingerprinting works.
- Why directory listing can be dangerous.
- How exposed status pages can disclose server information.
- Common security issues in Apache, Nginx, and Express.
- How to perform basic web server enumeration.
- How to use Nikto for identifying common web server issues.

**Areas to improve:**

- Understand Apache and Nginx configuration files in more depth.
- Learn how to identify exposed backup files systematically.
- Practice manual validation of scanner findings.
- Understand how web server misconfigurations can lead to further exploitation.
- Improve report writing and impact assessment.

---

## 14. Conclusion

Web server security is an important part of web application penetration testing.

A web server may be running without any obvious vulnerabilities, but insecure configurations can still expose sensitive information and increase the attack surface.

Understanding server fingerprinting, directory listing, status endpoints, security headers, and automated scanning provides a foundation for performing more detailed web server assessments.

**Learning approach:** Enumerate → Identify → Validate → Assess Impact → Document → Recommend Remediation.

---

**Disclaimer:** These notes are for educational purposes and authorized security testing only. Always obtain permission before scanning or testing any system.
