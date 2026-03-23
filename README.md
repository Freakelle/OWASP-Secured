# OWASP-Secured

A frontend replica of [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/) built as part of a web security assignment (HW3). The original Juice Shop is an intentionally vulnerable Node.js application used for security training. This version replicates its core UI and functionality in plain HTML and JavaScript, with the three vulnerabilities identified in Part 1 patched.

---

## What is OWASP Juice Shop?

OWASP Juice Shop is one of the most widely used intentionally insecure web applications in the security community. The application simulates a real e-commerce storefront complete with product listings, user accounts, a shopping cart, and an admin panel but every feature is deliberately broken in some way.

It covers vulnerabilities from the OWASP Top 10 (2021) and beyond, including SQL injection, cross-site scripting, broken authentication, insecure direct object references, security misconfigurations, and more. It's commonly used in university security courses, CTF competitions, and penetration testing practice because the flaws are real and exploitable, not simulated warnings.

---

## What this replica does

This project mirrors Juice Shop's frontend includes product browsing, search, login, and registration in a single self contained HTML file with no dependencies or build tools. The goal was to understand the vulnerabilities by actually exploiting them on the original, then rebuild the same interface with those issues fixed.

**Features:**
- Product catalog with 12 items and a working search bar
- Add to cart, cart page, and checkout flow
- Sign in and registration modal with client-side validation
- Password strength meter
- About page explaining each security fix

---

## Vulnerabilities fixed

### 1. SQL Injection (OWASP A03:2021 — Injection)

**Original behaviour:** The Juice Shop login page passes user input directly into a SQL query using string concatenation. Entering `' OR 1=1--` in the email field modifies the query to always return true, bypassing authentication entirely and logging in as the first user in the database (admin).

**Fix applied:** All input fields are checked against a regex pattern that detects common SQL injection payloads — single quotes, `--`, `OR 1=1`, `UNION SELECT`, `DROP TABLE`, and similar. Flagged input is rejected before it reaches any logic. In a production server, this would be backed by parameterised queries:
```js
db.query('SELECT * FROM users WHERE email = ?', [email])
```

This makes injection structurally impossible user input is always passed as a parameter, never interpolated into the query string.

---

### 2. Cross-Site Scripting — Reflected XSS (OWASP A03:2021 — Injection)

**Original behaviour:** The Juice Shop search bar reflects the query string back into the page HTML without sanitising it. Entering `<iframe src="javascript:alert('xss')">` causes the browser to execute the script, which in a real attack could be used to steal session cookies or redirect the user.

**Fix applied:** A `sanitize()` function runs on all user input before it touches the DOM. It works by creating a text node that the browser treats everything as plain text, automatically converting `<` to `&lt;` and `>` to `&gt;`:
```js
function sanitize(str) {
  const div = document.createElement('div');
  div.appendChild(document.createTextNode(str));
  return div.innerHTML;
}
```

The search result note also uses `textContent` instead of `innerHTML`, so reflected content can never be parsed as markup regardless of what the sanitizer does.

---

### 3. Broken Authentication (OWASP A07:2021 — Identification and Authentication Failures)

**Original behaviour:** Juice Shop accepts passwords of any length with no complexity requirements and no rate limiting. The admin account password is `admin123`, a credential found in every common password list. An attacker can log in with no tools, just by guessing.

**Fix applied:** Three layers of protection:

- **Password policy:** Minimum 8 characters, must include at least one uppercase letter, one number, and one special character. Common passwords (`admin123`, `password`, `123456`, etc.) are explicitly rejected by name.
- **Rate limiting:** After 5 consecutive failed login attempts, the form locks for 30 seconds. In production this would be enforced server-side using Redis to track attempts by IP address, so refreshing the page changes nothing.
- **Password hashing:** This frontend demo stores passwords in memory for demonstration purposes, but includes comments showing the correct server-side implementation using bcrypt with a cost factor of 12:
```js
// Registration
const hash = await bcrypt.hash(plainPassword, 12);
await db.query('INSERT INTO users (pw_hash) VALUES (?)', [hash]);

// Login
const match = await bcrypt.compare(plainPassword, storedHash);
```

With bcrypt at cost 12, each hash takes ~300ms to compute fast enough for a legitimate user, slow enough to make brute forcing millions of passwords completely impractical.

---





## Security notes

All validation in this project is client-side, which means a determined attacker could disable JavaScript and bypass the checks. That's intentional as this is a frontend demo. The comments throughout the code document what the server-side equivalents would be (parameterised queries, bcrypt, Redis rate limiting). Client-side validation is UX; server-side validation is security.

---

## References

- [OWASP Juice Shop Project](https://owasp.org/www-project-juice-shop/)
- [OWASP Top 10 (2021)](https://owasp.org/www-project-top-ten/)
- [bcrypt npm package](https://www.npmjs.com/package/bcrypt)
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
