# Core Module: Input Validation & Injection Prevention

**Token Estimate:** ~3,500 tokens
**Purpose:** Prevent injection attacks and validate all user input
**Dependencies:** core/00_overview.md

---

## Objective

Ensure all user input is properly validated, sanitized, and parameterized to prevent SQL injection, XSS, command injection, and other injection attacks.

---

## 1. SQL Injection Prevention

### 1.1 Parameterized Queries

```
Grep patterns:
- "query.*\$\{|sql.*\$\{" (template literals - DANGER)
- "query.*\+.*req\.|query.*\+.*params" (string concatenation - DANGER)
- Raw SQL: "\.raw\(|execute\(|query\("
```

**Flag P0 if:**
- Template literals used with user input in SQL
- String concatenation in queries
- No parameterization

**Example:**
```typescript
// ❌ VULNERABLE - SQL injection
const userId = req.params.id
db.query(`SELECT * FROM users WHERE id = ${userId}`) // DANGER!

// ❌ VULNERABLE - String concatenation
db.query("SELECT * FROM users WHERE id = " + userId) // DANGER!

// ✅ SECURE - Parameterized query
db.query('SELECT * FROM users WHERE id = ?', [userId])
// or
db.query('SELECT * FROM users WHERE id = $1', [userId])
```

### 1.2 ORM Usage Verification

```
Check ORM query builder usage:
Grep pattern: "\.from\(|\.where\(|\.select\("
```

**Verify safe patterns:**
```typescript
// ✅ SAFE - ORM methods
db.from('users').where('id', userId)
db.from('users').where({ id: userId })

// ❌ DANGEROUS - .ilike with interpolation
db.from('users').where('name', 'ilike', `%${userInput}%`)
```

### 1.3 Stored Procedures

```
If using stored procedures:
Grep pattern: "CALL|EXEC|EXECUTE.*PROCEDURE"
```

**Verify:**
- ✅ User input passed as parameters
- ✅ No dynamic SQL inside procedures
- ✅ Procedures follow least privilege

---

## 2. Cross-Site Scripting (XSS)

### 2.1 Output Encoding

```
Check for unescaped output:
Grep patterns (frontend):
- "dangerouslySetInnerHTML|v-html|innerHTML"
- "\.html\(|\.append\(" (jQuery-style)
```

**Flag P1 if:**
- User input rendered with dangerouslySetInnerHTML
- v-html used with user content
- Direct innerHTML assignment

**Example (React):**
```tsx
// ❌ VULNERABLE - XSS
<div dangerouslySetInnerHTML={{ __html: userComment }} />

// ✅ SECURE - React escapes by default
<div>{userComment}</div>
```

**Example (Vue):**
```vue
<!-- ❌ VULNERABLE -->
<div v-html="userComment"></div>

<!-- ✅ SECURE -->
<div>{{ userComment }}</div>
```

### 2.2 Content-Type Headers

```
Check API responses:
Grep pattern: "Content-Type|setHeader.*content"
```

**Verify:**
- ✅ APIs return `application/json`, not `text/html`
- ✅ Proper content type for all responses
- ✅ X-Content-Type-Options: nosniff header set

---

## 3. Command Injection

### 3.1 Shell Command Execution

```
Grep patterns:
- "exec\(|execSync\(|spawn\(|child_process"
- "system\(|shell_exec\(|passthru\(" (PHP)
- "os\.system|subprocess\." (Python)
```

**Flag P0 if:**
- User input in shell commands
- Uses `shell: true` with user input
- No input validation before exec

**Example:**
```typescript
// ❌ VULNERABLE - Command injection
const { exec } = require('child_process')
exec(`convert ${userFilename} output.png`) // DANGER!

// ❌ STILL VULNERABLE - shell: true
const { spawn } = require('child_process')
spawn('convert', [userFilename, 'output.png'], { shell: true })

// ✅ SECURE - No shell, validated input
const { execFile } = require('child_process')
const filename = path.basename(userFilename) // Remove path
if (!/^[a-zA-Z0-9_.-]+$/.test(filename)) {
  throw new Error('Invalid filename')
}
execFile('convert', [filename, 'output.png'])
```

### 3.2 Path Traversal

```
Grep patterns:
- "readFile|writeFile|unlink|open\("
- "\.\.\/|path.*\+|join.*req\."
```

**Flag P0 if:**
- User input directly in file paths
- No validation of `../` sequences
- Paths not resolved to absolute

**Example:**
```typescript
// ❌ VULNERABLE - Path traversal
const file = req.query.file
fs.readFile(`/uploads/${file}`, ...) // Can access ../../../../etc/passwd

// ✅ SECURE - Sanitize and validate
const filename = path.basename(req.query.file) // Remove directories
const fullPath = path.join('/uploads', filename)

// Verify still within uploads directory
const relative = path.relative('/uploads', fullPath)
if (relative.startsWith('..') || path.isAbsolute(relative)) {
  throw new Error('Invalid path')
}

fs.readFile(fullPath, ...)
```

---

## 4. NoSQL Injection

```
Check MongoDB/NoSQL queries:
Grep pattern: "find\(|findOne\(|update\(|deleteOne\("
```

**Verify:**
- ✅ User input not used directly as query object
- ✅ Operator injection prevented

**Example:**
```typescript
// ❌ VULNERABLE - Operator injection
db.users.find({ username: req.body.username })
// Attacker sends: { username: { $ne: null } } to match all users

// ✅ SECURE - Validate type
const username = String(req.body.username)
db.users.find({ username })

// ✅ BETTER - Schema validation
const schema = z.object({
  username: z.string().min(1).max(50)
})
const { username } = schema.parse(req.body)
db.users.find({ username })
```

---

## 5. LDAP Injection

```
Check LDAP queries (if applicable):
Grep pattern: "ldap|search.*filter"
```

**Verify:**
- ✅ Special LDAP characters escaped: `* ( ) \ NUL`
- ✅ User input validated before filter construction

---

## 6. XML/XXE Injection

```
Check XML parsing:
Grep pattern: "parseXML|DOMParser|libxml|xml2js"
```

**Verify:**
- ✅ External entity loading disabled
- ✅ DTD processing disabled

**Example (Node.js):**
```typescript
// ✅ SECURE - Disable external entities
const parser = new xml2js.Parser({
  explicitArray: false,
  explicitRoot: false,
  ignoreAttrs: true,
  xmldecl: false,
  // Disable external entities
  xmlns: false,
  explicitCharkey: false
})
```

---

## 7. Server-Side Template Injection (SSTI)

```
Check template engines:
Grep pattern: "render\(|compile\(|eval\(|new Function"
```

**Flag P0 if:**
- User input directly in template compilation
- Dynamic template generation from user input

**Example:**
```typescript
// ❌ VULNERABLE - SSTI
const template = `Hello ${userInput}`
eval(template) // DANGER!

// ✅ SECURE - Use template engine properly
const template = 'Hello {{name}}'
engine.render(template, { name: sanitize(userInput) })
```

---

## 8. Input Validation Framework

### 8.1 Schema Validation

```
Check for validation libraries:
Grep pattern: "zod|yup|joi|validator|ajv"
```

**Verify:**
- ✅ All API endpoints use schema validation
- ✅ Validation happens server-side
- ✅ Type coercion controlled
- ✅ Length limits enforced

**Example (Zod):**
```typescript
// ✅ COMPREHENSIVE VALIDATION
const createUserSchema = z.object({
  email: z.string().email().max(255),
  username: z.string().min(3).max(50).regex(/^[a-zA-Z0-9_]+$/),
  age: z.number().int().min(13).max(120),
  website: z.string().url().optional()
})

app.post('/users', async (req, res) => {
  const validated = createUserSchema.parse(req.body) // Throws on invalid
  // Now safe to use validated data
})
```

### 8.2 Whitelist vs Blacklist

**Prefer whitelists:**
```typescript
// ❌ BLACKLIST - Can be bypassed
const sanitize = (input) => input.replace(/[<>]/g, '')

// ✅ WHITELIST - Only allow known-safe characters
const sanitize = (input) => {
  if (!/^[a-zA-Z0-9 .,!?-]+$/.test(input)) {
    throw new Error('Invalid characters')
  }
  return input
}
```

---

## 9. File Upload Validation

### 9.1 File Type Validation

```
Check upload handlers:
Grep pattern: "upload|multer|formidable|busboy"
```

**Verify:**
- ✅ MIME type validated (not just extension)
- ✅ File extension whitelist
- ✅ Magic bytes checked (not just header)
- ✅ File size limits enforced
- ✅ Filename sanitized

**Example:**
```typescript
// ❌ INSUFFICIENT - Extension only
if (!filename.endsWith('.jpg')) reject()

// ✅ COMPREHENSIVE
const allowedTypes = ['image/jpeg', 'image/png', 'image/webp']
const maxSize = 5 * 1024 * 1024 // 5MB

if (!allowedTypes.includes(file.mimetype)) {
  throw new Error('Invalid file type')
}

if (file.size > maxSize) {
  throw new Error('File too large')
}

// Check magic bytes (first bytes of file)
const fileSignature = file.buffer.slice(0, 4).toString('hex')
const validSignatures = {
  'ffd8ffe0': 'image/jpeg',
  '89504e47': 'image/png'
}

if (!validSignatures[fileSignature]) {
  throw new Error('Invalid file signature')
}

// Sanitize filename
const safeFilename = path.basename(file.originalname)
  .replace(/[^a-zA-Z0-9.-]/g, '_')
```

### 9.2 Upload Destination

**Verify:**
- ✅ Files stored outside web root
- ✅ Generated filenames (not user-provided)
- ✅ No script execution in upload directory

---

## 10. Regular Expression DoS (ReDoS)

```
Check regex patterns:
Grep pattern: "new RegExp|\.test\(|\.match\(|\.replace\("
```

**Flag vulnerable patterns:**
- ❌ Nested quantifiers: `(a+)+`, `(a*)*`
- ❌ Overlapping alternatives: `(a|ab)+`

**Example:**
```typescript
// ❌ VULNERABLE - Exponential backtracking
const emailRegex = /^([a-zA-Z0-9]+)*@[a-zA-Z0-9]+\.[a-zA-Z]+$/
// Input: "aaaaaaaaaaaaaaaaaaaaa!" causes hang

// ✅ SECURE - No nested quantifiers
const emailRegex = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/
```

**Test complex regexes:** https://devina.io/redos-checker

---

## 11. HTTP Header Injection

```
Check header manipulation:
Grep pattern: "setHeader|res\.set\(|response\.headers"
```

**Verify:**
- ✅ User input not directly in headers
- ✅ Newline characters (`\r\n`) stripped

**Example:**
```typescript
// ❌ VULNERABLE - Header injection
res.setHeader('X-User-Name', req.query.name)
// Attacker: ?name=Admin%0D%0AX-Admin:true

// ✅ SECURE - Sanitize newlines
const sanitized = req.query.name.replace(/[\r\n]/g, '')
res.setHeader('X-User-Name', sanitized)
```

---

## 12. Open Redirect

```
Check redirect implementations:
Grep pattern: "redirect\(|location.*=|window\.location"
```

**Verify:**
- ✅ Redirect URLs validated against whitelist
- ✅ Only relative paths allowed OR
- ✅ Domain whitelist enforced

**Example:**
```typescript
// ❌ VULNERABLE - Open redirect
res.redirect(req.query.returnTo)
// Attacker: ?returnTo=https://evil.com

// ✅ SECURE - Whitelist domains
const allowed = ['https://myapp.com', 'https://api.myapp.com']
const url = new URL(req.query.returnTo)
if (!allowed.includes(url.origin)) {
  throw new Error('Invalid redirect')
}
res.redirect(req.query.returnTo)

// ✅ BETTER - Only relative paths
if (req.query.returnTo.startsWith('/')) {
  res.redirect(req.query.returnTo)
}
```

---

## Validation Checklist

For EVERY user input, verify:
- [ ] Type validation (string, number, email, etc.)
- [ ] Length limits (min/max)
- [ ] Format validation (regex)
- [ ] Character whitelist (if applicable)
- [ ] Parameterized in database queries
- [ ] Escaped in output contexts
- [ ] Sanitized for file system operations
- [ ] Validated before external API calls

---

## Testing Requirements

**Must have tests for:**
- [ ] SQL injection attempts blocked
- [ ] XSS payloads escaped
- [ ] Path traversal prevented
- [ ] Command injection blocked
- [ ] File upload restrictions enforced
- [ ] Invalid input rejected with 400 errors

---

**Severity Summary:**
- **P0:** SQL injection, command injection, arbitrary file access
- **P1:** XSS, NoSQL injection, path traversal
- **P2:** Missing validation, ReDoS risk, open redirect
