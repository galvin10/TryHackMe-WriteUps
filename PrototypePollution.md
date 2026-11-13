# Prototype Pollution

**Platform:** TryHackMe  
**Category:** Web Application Security  
**Topic:** JavaScript / Node.js Security  
**Status:** Completed ✅

---

## 1. Introduction

Prototype Pollution is a JavaScript vulnerability that occurs when an attacker can modify properties of an object's prototype through unsafe handling of user-controlled input.

JavaScript uses prototype-based inheritance. Objects can inherit properties from other objects through their prototype chain.

When an attacker successfully modifies a shared prototype, the changes can affect other objects that inherit from it.

Prototype Pollution can potentially lead to:

- Unauthorized privilege escalation.
- Authentication and authorization bypass.
- Application logic manipulation.
- Denial of Service (DoS).
- Cross-Site Scripting (XSS).
- Remote Code Execution (RCE) in certain vulnerable application contexts.

## 2. Understanding JavaScript Prototypes

Every ordinary JavaScript object has an internal prototype that allows it to inherit properties and methods from another object.

Example:

```javascript
let user = {
    name: "John"
};

console.log(user.name);
console.log(user.toString);
```

The first property belongs directly to the `user` object.

The second property is inherited through the prototype chain.

### Understanding `__proto__`

The `__proto__` accessor provides access to an object's prototype.

Example:

```javascript
let user = {};

console.log(user.__proto__ === Object.prototype);
```

Output:

```text
true
```

This demonstrates that an ordinary object inherits from `Object.prototype`.

### Understanding Object.prototype

`Object.prototype` is the top-level prototype for most ordinary JavaScript objects.

Properties added to it can become accessible through inheritance to many other objects.

Example:

```javascript
Object.prototype.newProperty = "polluted";

let user = {};

console.log(user.newProperty);
```

Output:

```text
polluted
```

The `user` object does not directly contain `newProperty`. It inherits the property from `Object.prototype`.

This is the fundamental concept behind Prototype Pollution.

---

## 3. Understanding the Prototype Pollution Vulnerability

Prototype Pollution occurs when an application allows user-controlled input to modify an object's prototype or another shared prototype.

Three important property names to understand:

| Property | Purpose |
|---|---|
| `__proto__` | Accesses an object's prototype through an accessor |
| `constructor` | References the constructor function associated with an object |
| `prototype` | Represents the prototype object of a constructor function |

These properties become especially important when applications perform unsafe recursive object merging.

---

## 4. Vulnerable Recursive Merge Function

Consider the following vulnerable implementation:

```javascript
function recursiveMerge(target, source) {
    for (let key in source) {
        if (source[key] instanceof Object) {
            if (!target[key]) target[key] = {};
            recursiveMerge(target[key], source[key]);
        } else {
            target[key] = source[key];
        }
    }
}

let globalUserSettings = {
    theme: "light",
    language: "en"
};

app.post("/updateSettings", (req, res) => {
    const userSettings = req.body;

    recursiveMerge(globalUserSettings, userSettings);

    res.send("Settings updated!");
});
```

### Why is this vulnerable?

The application directly merges user-controlled input into an internal JavaScript object.

The following problems exist:

1. No validation of user-controlled property names.
2. No protection against dangerous property paths.
3. Recursive processing of nested objects.
4. No restrictions on which properties can be modified.
5. User-controlled input directly affects internal application objects.

### Example attack payload

```json
{
    "__proto__": {
        "isAdmin": true
    }
}
```

The vulnerable merge function can access the target object's prototype and recursively modify it.

Conceptually, the operation can result in:

```javascript
Object.prototype.isAdmin = true;
```

Now consider:

```javascript
let user = {};

console.log(user.isAdmin);
```

Output:

```text
true
```

The property is inherited rather than directly assigned to the user object.

**Important:** Actual exploitability depends on how the input is parsed and how the merge function handles special properties.

---

## 5. JSON and Prototype Pollution

JSON is frequently used to transfer user-controlled objects to Node.js applications.

Example:

```json
{
    "theme": "dark",
    "language": "en"
}
```

An application may parse this using:

```javascript
const input = JSON.parse(requestBody);
```

A significant distinction is that JSON parsing does not automatically pollute `Object.prototype`.

For example:

```javascript
let data = JSON.parse(
    '{"__proto__":{"isAdmin":true}}'
);

console.log(data.isAdmin);
console.log(Object.prototype.isAdmin);
```

Output:

```text
undefined
undefined
```

The `__proto__` key exists as an own property of the parsed object.

The vulnerability may occur later when an unsafe merge or assignment operation processes that object.

This distinction is important when investigating prototype pollution.

---

## 6. Dangerous JavaScript Operations

During source-code review, pay attention to functions that merge or assign user-controlled object properties.

Examples:

```javascript
_.merge(target, source);
```

```javascript
_.defaultsDeep(target, source);
```

```javascript
_.set(target, path, value);
```

```javascript
Object.assign(target, source);
```

Also investigate custom recursive merge functions.

These operations are not automatically vulnerable. Their security depends on the implementation, library version, input structure, and how the resulting objects are used.

### Important distinction

| Operation | Security consideration |
|---|---|
| `_.merge()` | Review special-property handling and library version |
| `_.defaultsDeep()` | Review nested property handling |
| `_.set()` | Review user-controlled property paths |
| `Object.assign()` | Review assignment to special properties and target objects |
| Custom recursive merge | Inspect recursion and property validation |

---

## 7. Prototype Pollution Through User-Controlled Property Paths

Consider this example:

```javascript
let friends = [
    {
        id: 1,
        name: "Sabalenka",
        age: 25,
        country: "UK",
        reviews: [],
        albums: [],
        password: "xxx"
    }
];

app.post("/submit-friend-review", (req, res) => {

    if (!req.session.user) {
        return res.redirect("/signin");
    }

    const { friendId, reviewContent } = req.body;

    const friend = friends.find(
        f => f.id === parseInt(friendId)
    );

    if (!friend) {
        return res.status(404).send("Friend not found");
    }

    try {
        const input = JSON.parse(reviewContent);

        _.set(friend, input.path, input.value);

    } catch (e) {}

    res.redirect(`/friend/${friendId}`);
});
```

### Vulnerable operation

```javascript
_.set(friend, input.path, input.value);
```

The application allows users to control the property path.

For example:

```json
{
    "path": "password",
    "value": "attacker123"
}
```

This demonstrates arbitrary property modification.

However, arbitrary property modification and Prototype Pollution are not the same vulnerability.

Prototype Pollution requires demonstrating that a prototype is actually modified.

**Key learning:** Always distinguish between mass assignment, arbitrary property modification, and Prototype Pollution.

---

## 8. Identifying Prototype Pollution in Bug Bounty Hunting

In real-world assessments, the backend source code is not always available.

Bug hunters generally use two approaches.

### White-box testing

When source code is available, inspect:

- Request handlers.
- Controllers.
- Recursive merge functions.
- Object assignment operations.
- Third-party JavaScript libraries.
- User-controlled property paths.

Look for data flowing from an input source into a potentially unsafe operation.

Example:

```javascript
app.post("/updateSettings", (req, res) => {
    recursiveMerge(globalSettings, req.body);
});
```

### Black-box testing

When source code is unavailable, investigate application behavior.

Useful tools:

- Burp Suite.
- Browser Developer Tools.
- curl.
- JavaScript console.

General testing methodology:

1. Identify endpoints accepting JSON or nested objects.
2. Intercept requests using Burp Suite.
3. Identify user-controlled properties.
4. Investigate how the application processes nested input.
5. Use harmless prototype-pollution test cases in an authorized environment.
6. Check whether inherited properties change.
7. Determine whether polluted properties affect application functionality.
8. Document the security impact.

A suspicious input sink alone is not sufficient evidence of a vulnerability.

---

## 9. Secure Implementation

The best approach is to avoid unrestricted recursive merging of untrusted input.

Instead, use an allowlist of permitted properties.

Example:

```javascript
app.post("/updateSettings", (req, res) => {

    const input = req.body;

    if (!input || typeof input !== "object" ||
        Array.isArray(input)) {
        return res.status(400).send("Invalid settings");
    }

    if (
        typeof input.theme === "string" &&
        ["light", "dark"].includes(input.theme)
    ) {
        globalUserSettings.theme = input.theme;
    }

    if (
        typeof input.language === "string" &&
        ["en", "fr", "de"].includes(input.language)
    ) {
        globalUserSettings.language = input.language;
    }

    res.send("Settings updated!");
});
```

### Security improvements

1. Avoid unrestricted recursive merging.
2. Allow only explicitly permitted properties.
3. Validate data types and values.
4. Reject unexpected nested properties.
5. Keep application configuration separate from user-controlled data.
6. Use maintained libraries with appropriate prototype-pollution protections.
7. Review dependencies for known vulnerabilities.

---

## 10. Vulnerable vs Secure Comparison

| Feature | Vulnerable Implementation | Secure Implementation |
|---|---|---|
| User-controlled object merging | Unrestricted | Restricted |
| Property validation | Missing | Implemented |
| Dangerous property handling | Not protected | Prevented by design |
| Recursive object processing | Unrestricted | Avoided or controlled |
| Application state | Directly modified | Explicitly updated |
| Prototype pollution risk | Potentially exploitable | Significantly reduced |

---

## 11. Important Things to Remember

1. JavaScript uses prototype-based inheritance.
2. Ordinary objects inherit properties through `Object.prototype`.
3. `__proto__`, `constructor`, and `prototype` are important when investigating prototype-related vulnerabilities.
4. JSON parsing alone does not automatically cause Prototype Pollution.
5. Unsafe recursive merge functions can introduce Prototype Pollution.
6. User-controlled property paths can create dangerous object modifications.
7. Prototype Pollution can affect multiple objects through shared prototypes.
8. Prototype Pollution does not automatically provide administrator privileges or RCE.
9. The actual impact depends on how polluted properties are consumed by the application.
10. Input validation and property allowlisting are important mitigations.

---

## 12. Conclusion

Completing this room helped me understand JavaScript's prototype-based inheritance mechanism and how improper handling of user-controlled objects can introduce security vulnerabilities.

The most important lesson was understanding the relationship between JavaScript prototypes, recursive object merging, and user-controlled property modification.

I also learned to distinguish between ordinary property modification and actual Prototype Pollution, which is important when validating findings during source-code reviews and bug bounty assessments.

**Room Status:** Completed ✅

**Next Learning Focus:**
- JavaScript object internals.
- Prototype chain manipulation.
- Node.js source-code review.
- Client-side and server-side Prototype Pollution.
- Prototype Pollution to XSS and other impact chains.
- Secure object merging and dependency auditing.

---

## Disclaimer

All techniques documented here are intended for educational purposes, authorized penetration testing, and controlled environments such as TryHackMe. Always obtain explicit authorization before testing real-world applications.
