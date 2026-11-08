# TryHackMe – Walking An Application

> Hands-on web application security assessment using browser developer tools.

| Field | Details |
|---|---|
| Platform | TryHackMe |
| Room | Walking An Application |
| Difficulty | Easy |
| Category | Web Application Security |
| Target | ACME IT Support |
| Date Completed | 09 November 2026 |
| Tools | Firefox Developer Tools, Inspector, Debugger, Network Monitor |
| Status | Completed |

**Room Link:** https://tryhackme.com/room/walkinganapp

---

## Table of Contents

1. Introduction
2. Exploring the Website
3. Viewing the Page Source
4. Developer Tools – Inspector
5. Developer Tools – Debugger
6. Developer Tools – Network
7. Developer Tools – Storage
8. Conclusion
9. Key Takeaways
10. Tools & Techniques Reference

---

# 1. Introduction

## What is Manual Web Application Testing?

Manual web application testing involves examining a website to identify security weaknesses by interacting with its components and analyzing how it behaves.

Unlike automated vulnerability scanners, manual testing allows a penetration tester to understand the application's functionality, inspect its source code, observe network communication and identify weaknesses that automated tools might overlook.

This room focuses on using only the built-in developer tools available in modern browsers.

## Learning Objectives

- Understand how to explore a web application.
- Identify sensitive information in HTML source code.
- Inspect and modify webpage elements.
- Analyze JavaScript execution.
- Monitor HTTP requests and responses.
- Examine browser storage and cookies.
- Identify common web application security misconfigurations.

## Browser Developer Tools

The room introduces the following browser features:

| Tool | Purpose |
|---|---|
| View Source | Examine the original HTML returned by the server. |
| Inspector | Inspect and temporarily modify webpage elements. |
| Debugger | Analyze JavaScript and control execution using breakpoints. |
| Network | Monitor requests and responses between the browser and server. |
| Storage | Examine cookies and locally stored application data. |

## Lab Environment

The target application is a fictional company website called ACME IT Support.

The lab machine provides a web application that can be accessed through the browser after deploying the TryHackMe machine.

The recommended workflow is to use Firefox on the AttackBox or access the target through the TryHackMe VPN.

---

# 2. Exploring the Website

## Why Website Exploration Matters

Before performing any security testing, it is important to understand the application's functionality.

A penetration tester should identify the different features and user interactions available on the website.

These may include:

- Login and registration pages.
- Customer portals.
- Contact forms.
- News and articles.
- File downloads.
- Ticket creation functionality.
- Administrative or restricted pages.

## 2.1 Mapping Application Functionality

During the initial exploration, examine the available pages and identify how users interact with the application.

Pay particular attention to:

1. Forms that accept user input.
2. Authentication mechanisms.
3. Links that may not appear in the main navigation.
4. Features that require user registration.
5. Pages containing restricted or premium content.
6. Functions that communicate with the server in the background.

## 2.2 Customer Ticket Functionality

The room also introduces the customer ticket functionality.

The endpoint associated with creating a new customer ticket is:

```text
/customers/ticket/new
```

Understanding application endpoints is useful because they can reveal functionality that may not be immediately visible through the website's navigation.

### Enumeration Takeaway

A website should not be treated as a collection of static pages.

Every form, button, link, API endpoint and user interaction can reveal additional information about the application's attack surface.

---

# 3. Viewing the Page Source

## What is Page Source?

Page source is the human-readable code returned by the web server to the browser.

It generally contains:

- HTML – Defines the structure of the webpage.
- CSS – Controls the appearance and layout.
- JavaScript – Provides interactive functionality.

Viewing the source code can reveal information that is not immediately visible on the rendered webpage.

## 3.1 Accessing Page Source

There are several ways to access the page source.

**Method 1: Keyboard Shortcut**

```text
Ctrl + U
```

**Method 2: Browser Address Bar**

```text
view-source:http://TARGET_IP
```

**Method 3: Browser Context Menu**

1. Right-click the webpage.
2. Select View Page Source.
3. Examine the returned HTML.

## 3.2 Finding Information in HTML Comments

HTML comments are not displayed as normal webpage content.

Example:

```html
<!-- This page is temporary while we work on the new homepage @ /new-home-beta -->
```

Although comments are hidden from normal visitors, they are still delivered to the browser.

An attacker can inspect these comments to discover:

- Hidden directories.
- Development endpoints.
- Temporary pages.
- Internal application information.
- Developer notes.

Following the path discovered in the comment leads to:

```text
/new-home-beta
```

The page contains the first flag.

**Finding:** Sensitive information disclosed through HTML comments.

## 3.3 Discovering Hidden Links

While examining the homepage source, another link can be identified that is not displayed in the website's normal navigation.

The hidden endpoint is:

```text
/secret-page
```

Navigating to this endpoint reveals another flag.

This demonstrates why examining HTML source code is important even when a website appears to have limited functionality.

**Finding:** Hidden application endpoints exposed through page source.

## 3.4 Directory Listing

Directory listing occurs when a web server allows visitors to view the contents of a directory.

During the room, the following directory is identified:

```text
/assets/
```

Navigating to the directory reveals its contents, including:

```text
flag.txt
```

The file can be accessed directly:

```text
http://TARGET/assets/flag.txt
```

This exposes information that should not necessarily be publicly accessible.

**Security Impact:**

- Disclosure of internal files.
- Exposure of sensitive information.
- Discovery of application resources.
- Potential exposure of backup files and configuration files.

**Finding:** Directory listing enabled due to improper server configuration.

## 3.5 Identifying the Web Framework

While inspecting the page source and footer, the application reveals that it uses:

```text
THM Framework v1.2
```

The framework information provides an opportunity to research the software and identify known security issues.

The framework website is:

https://static-labs.tryhackme.cloud/sites/thm-web-framework

The next step is to inspect the framework's Change Log.

The changelog reveals that version 1.3 fixed an issue involving a backup process that created a publicly accessible ZIP archive.

The vulnerable version used by the target is 1.2.

The backup archive can be accessed through:

```text
/tmp.zip
```

Download the archive:

```bash
wget http://TARGET_IP/tmp.zip
```

Extract its contents:

```bash
unzip tmp.zip
```

Inspect the extracted flag file:

```bash
cat flag.txt
```

This reveals the fourth flag.

### Security Impact

Outdated frameworks may contain known vulnerabilities and insecure default configurations.

Publicly accessible backup archives can expose:

- Source code.
- Configuration files.
- Internal information.
- Credentials.
- Sensitive application data.

**Finding:** Outdated framework version and publicly accessible backup archive.

### Page Source Takeaways

- HTML comments can expose hidden application information.
- Hidden links can reveal unlisted endpoints.
- Directory listing can expose sensitive files.
- Framework version information helps identify outdated software.
- Backup files should never be publicly accessible without proper authorization.

---

# 4. Developer Tools – Inspector

## What is the Inspector?

The Inspector is a browser developer tool that allows users to examine the live Document Object Model (DOM) of a webpage.

The original page source does not always represent what is currently displayed because JavaScript, CSS and user interactions can modify the webpage after it loads.

The Inspector provides a live view of the current page structure.

## 4.1 Understanding the DOM

The Document Object Model represents a webpage as a structured tree of elements.

For example:

```html
<div class="container">
    <h1>Welcome</h1>
    <p>Example content</p>
</div>
```

Using the Inspector, we can examine the elements and modify their attributes or styling temporarily.

## 4.2 Inspecting Restricted Content

The room demonstrates a page containing premium content.

When accessing the article, a message appears indicating that the content is only available to premium customers.

However, the underlying article content is already present in the webpage.

The blocking element uses the following CSS class:

```text
premium-customer-blocker
```

## 4.3 Removing the Paywall Overlay

Steps:

1. Navigate to the restricted news article.
2. Open Developer Tools using `F12`.
3. Select the Inspector tab.
4. Locate the element with the class `premium-customer-blocker`.
5. Inspect the associated HTML and CSS.
6. Remove the blocking element or temporarily hide it.

The CSS can be modified as follows:

```css
display: block;
```

Change it to:

```css
display: none;
```

Alternatively, remove the blocking element from the DOM.

The underlying article becomes visible.

The article also contains a hidden image-based flag. Its Base64-encoded image data can be decoded to reveal the flag.

## 4.4 Understanding the Security Issue

The application relies on a client-side visual restriction to hide premium content.

However, the actual content is already delivered to the browser.

Changing the DOM or CSS removes the visual restriction without requiring any server-side authorization.

### Security Impact

- Unauthorized viewing of restricted content.
- Exposure of information intended for premium users.
- Improper implementation of access control.

### Remediation

- Enforce authorization on the server side.
- Do not send restricted content to unauthorized users.
- Validate user permissions before returning sensitive data.
- Never rely on CSS or JavaScript to enforce access control.

**Important:** Changes made through the Inspector are temporary and affect only the local browser session. They do not modify the actual server-side website.

---

# 5. Developer Tools – Debugger

## What is the Debugger?

The Debugger allows developers and security testers to examine JavaScript execution.

It provides the ability to:

- Inspect JavaScript files.
- Set breakpoints.
- Pause execution.
- Examine variables.
- Understand application logic.
- Analyze dynamically generated content.

In Firefox, this tool is called Debugger.

In Google Chrome, the equivalent functionality is available under Sources.

## 5.1 Understanding JavaScript Execution

Modern web applications rely heavily on JavaScript.

JavaScript can dynamically:

- Modify webpage elements.
- Validate form inputs.
- Display notifications.
- Hide or remove content.
- Send background requests.
- Control application functionality.

From a penetration-testing perspective, examining JavaScript can reveal information about the application's behavior and potential security weaknesses.

## 5.2 Finding the Hidden Red Box

The room contains a red box that appears briefly and disappears almost immediately.

The relevant JavaScript file is:

```text
flash.min.js
```

The script contains functionality that removes the red box after a very short delay.

One of the relevant statements is:

```javascript
flag['remove']();
```

## 5.3 Using Breakpoints

A breakpoint pauses JavaScript execution at a particular line.

Steps:

1. Open the contact page.
2. Press `F12`.
3. Navigate to the Debugger tab.
4. Locate `flash.min.js`.
5. Find the statement responsible for removing the red box.
6. Click the line number to set a breakpoint.
7. Refresh the webpage.
8. JavaScript execution pauses at the breakpoint.

Since the script is paused before removing the element, the red box remains visible.

The flag can then be inspected.

### Alternative: Analyze the JavaScript

The flag is also assembled inside the JavaScript source using character codes.

The important concept is that examining the script itself can be more reliable than trying to capture a very short-lived visual element.

## 5.4 Security Learning

JavaScript should never be treated as a secure location for sensitive information.

Anything delivered to the browser can potentially be inspected by the user.

Security testers should examine:

- Hardcoded credentials.
- API keys.
- Hidden endpoints.
- Client-side authorization checks.
- Sensitive application logic.
- Obfuscated scripts.

### Debugger Takeaway

Understanding how JavaScript executes is important when investigating client-side security issues.

Breakpoints allow a tester to observe application behavior at specific points rather than relying only on the final rendered output.

---

# 6. Developer Tools – Network

## What is the Network Tab?

The Network tab allows us to monitor all requests made by a webpage.

Whenever a browser loads a webpage or interacts with an application, it communicates with the web server through HTTP or HTTPS requests.

The Network Monitor helps us examine:

- Request URLs.
- HTTP methods.
- Status codes.
- Request headers.
- Response headers.
- Request parameters.
- Response bodies.
- AJAX and API communication.

## 6.1 Monitoring Network Requests

Steps:

1. Open the website.
2. Press `F12`.
3. Navigate to the Network tab.
4. Refresh the webpage.
5. Observe the requests generated by the browser.

The Network tab displays the different resources requested by the webpage.

These can include:

- HTML documents.
- JavaScript files.
- CSS files.
- Images.
- API requests.
- Background requests.

## 6.2 Understanding AJAX

AJAX stands for Asynchronous JavaScript and XML.

It allows a webpage to communicate with a server in the background without requiring a complete page reload.

Modern applications commonly use AJAX or the Fetch API to exchange information.

For example, submitting a contact form may trigger a background HTTP request.

## 6.3 Capturing the Contact Form Request

The room demonstrates identifying a request generated when submitting the contact form.

Steps:

1. Navigate to the Contact Us page.
2. Open Developer Tools.
3. Select the Network tab.
4. Clear previous network requests.
5. Enter information into the contact form.
6. Click Send Message.
7. Observe the newly generated request.

A request named:

```text
contact-msg
```

appears in the Network Monitor.

Select the request and inspect its response.

The response contains a flag that is not displayed directly on the webpage.

## 6.4 Understanding the Security Implication

The important lesson is that the browser displays only the information selected by the frontend application.

However, the server may return additional information in its HTTP response.

A penetration tester should therefore inspect the complete response body rather than relying only on what appears on the screen.

### Security Testing Opportunities

- Sensitive information exposed through API responses.
- Excessive data returned by backend endpoints.
- Insecure direct object references.
- Improper authorization.
- Information disclosure.
- Sensitive data transmitted through requests.

### Network Takeaway

The Network tab is one of the most useful tools for understanding how a web application communicates with its backend.

It helps identify endpoints and inspect data exchanged between the client and server.

---

# 7. Developer Tools – Storage

## What is Browser Storage?

Web applications frequently store information locally in the browser.

This information can be used to maintain sessions, remember preferences and support application functionality.

The Storage section of Developer Tools allows testers to inspect data stored by websites.

Common storage mechanisms include:

| Storage Type | Description |
|---|---|
| Cookies | Store information such as session identifiers and preferences. |
| Local Storage | Stores persistent key-value data in the browser. |
| Session Storage | Stores data for the current browser tab session. |
| IndexedDB | Stores structured data within the browser. |
| Cache Storage | Stores cached resources for web applications. |

## 7.1 Inspecting Cookies

Cookies are frequently used for authentication and session management.

A session cookie can identify an authenticated user when communicating with the server.

Steps:

1. Register a customer account on the website.
2. Log in using the registered account.
3. Open Developer Tools.
4. Navigate to the Storage section.
5. Select Cookies.
6. Examine the cookies created by the application.

The room demonstrates examining the session cookie and its security attributes.

## 7.2 Understanding the HttpOnly Attribute

The `HttpOnly` attribute is a security setting that prevents JavaScript running in the browser from accessing a cookie through interfaces such as:

```javascript
document.cookie
```

A cookie configured with HttpOnly:

```http
Set-Cookie: session=example; HttpOnly
```

A cookie without HttpOnly:

```http
Set-Cookie: session=example
```

Without this attribute, client-side JavaScript may be able to read the cookie.

## 7.3 Session Cookie Analysis

During the room, the session cookie is examined.

The application does not set the `HttpOnly` attribute on the session cookie.

The answer to the room's HttpOnly question is:

```text
false
```

The application also exposes an `admin=false` cookie, demonstrating why security testers should examine client-side authentication and authorization-related data carefully.

## 7.4 Security Impact

A session cookie without HttpOnly can increase the impact of a Cross-Site Scripting (XSS) vulnerability.

If an attacker can execute malicious JavaScript in the victim's browser, the script may be able to access the session cookie.

This could potentially lead to session theft and account compromise.

Client-side cookies must also never be trusted as the sole mechanism for enforcing authorization.

## 7.5 Remediation

- Set the `HttpOnly` attribute on session cookies.
- Use the `Secure` attribute to restrict cookies to HTTPS connections.
- Configure an appropriate `SameSite` attribute.
- Regenerate session identifiers after authentication.
- Implement authorization checks on the server.
- Avoid storing sensitive information in browser-accessible storage.

### Storage Takeaway

Browser storage can reveal important information about how an application manages authentication, sessions and user data.

Cookie attributes are particularly important when assessing web application security.

---

# 8. Conclusion

This room demonstrated how to manually inspect a web application using only browser developer tools.

Instead of relying on automated scanners or external exploitation tools, the exercises focused on understanding the application from the browser's perspective.

The practical exercises covered:

- Exploring application functionality and endpoints.
- Inspecting HTML source code.
- Discovering hidden links and directories.
- Identifying outdated frameworks.
- Examining client-side access restrictions.
- Analyzing JavaScript execution.
- Using breakpoints to inspect hidden content.
- Monitoring HTTP requests and responses.
- Understanding AJAX communication.
- Examining cookies and browser storage.
- Identifying insecure session-cookie configurations.

The most important lesson is that a web application exposes a significant amount of information to the browser.

A penetration tester should understand how to examine that information before moving towards more advanced exploitation techniques.

Manual inspection is an essential part of web application penetration testing.

---

# 9. Key Takeaways

| Area | What I Learned |
|---|---|
| Website Exploration | Identify application features, forms and endpoints. |
| Page Source | Find hidden links, comments and sensitive information. |
| Directory Listing | Identify improperly exposed directories and files. |
| Framework Identification | Research outdated software and known issues. |
| Inspector | Understand the DOM and modify client-side elements. |
| Debugger | Analyze JavaScript execution using breakpoints. |
| Network | Monitor HTTP requests, responses and AJAX communication. |
| Storage | Examine cookies and locally stored application data. |
| Cookie Security | Understand HttpOnly and other cookie security attributes. |

## Personal Learning Notes

- The browser is an important penetration-testing tool.
- Information hidden in HTML comments is still accessible to users.
- Hidden links should not be considered secure endpoints.
- Client-side restrictions do not replace server-side authorization.
- JavaScript source code can expose sensitive application information.
- Breakpoints help understand how an application behaves.
- API responses may contain information that is not visible in the frontend.
- Session cookies must be configured securely.
- Manual testing helps identify issues that automated scanners may overlook.

---

# 10. Tools & Techniques Reference

## Browser Developer Tools

```text
F12
```

Opens browser developer tools.

## View Page Source

```text
Ctrl + U
```

Or:

```text
view-source:http://TARGET_IP
```

## Inspect HTML Comments

Example:

```html
<!-- Hidden developer information -->
```

## Inspect Hidden Endpoints

Example endpoints discovered in the room:

```text
/new-home-beta
/secret-page
/assets/
/tmp.zip
```

## Download Files

```bash
wget http://TARGET_IP/tmp.zip
```

## Extract ZIP Files

```bash
unzip tmp.zip
```

## Read Extracted Files

```bash
cat flag.txt
```

## JavaScript Debugging

```javascript
debugger;
```

Can be used to pause JavaScript execution when the statement is reached.

## Inspect Cookies

```javascript
document.cookie
```

Note: JavaScript cannot access cookies marked with the HttpOnly attribute.

---

## Room Completion Summary

| Task | Topic | Status |
|---|---|---|
| Task 1 | Walking An Application | Completed |
| Task 2 | Exploring The Website | Completed |
| Task 3 | Viewing The Page Source | Completed |
| Task 4 | Developer Tools – Inspector | Completed |
| Task 5 | Developer Tools – Debugger | Completed |
| Task 6 | Developer Tools – Network | Completed |
| Task 7 | Developer Tools – Storage | Completed |
| Task 8 | Conclusion | Completed |

## References

- [TryHackMe – Walking An Application](https://tryhackme.com/room/walkinganapp)
- [MDN – Firefox Developer Tools](https://developer.mozilla.org/en-US/docs/Tools)
- [MDN – HTTP Cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies)
- [MDN – Document Object Model](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model)
- [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)

---

**Room Completed:** 09 November 2026

**Learning Path:** Jr Penetration Tester

*Documented as part of my hands-on cybersecurity, web application security and penetration testing learning journey.*

*All testing activities are intended for authorized lab environments only.*
