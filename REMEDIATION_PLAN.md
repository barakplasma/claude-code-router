# Security Remediation Plan - Claude Code Router

**Date:** 2025-02-05  
**Priority:** CRITICAL - Immediate Action Required  
**Goal:** Eliminate Remote Code Execution vulnerabilities and simplify the project  

---

## Executive Summary

This remediation plan addresses the **3 critical Remote Code Execution (RCE)** vulnerabilities identified in the security review by **removing dangerous features** and **simplifying the project architecture**. This approach both eliminates security risks and aligns with the goal of project simplification.

### Strategic Decision: Feature Removal Over Complex Mitigation

Instead of implementing complex sandboxing, validation, and monitoring systems to mitigate RCE vulnerabilities, we will **remove the problematic features entirely**. This approach:

- ✅ **Eliminates security risk** more effectively than mitigation
- ✅ **Simplifies codebase** and reduces maintenance burden  
- ✅ **Removes attack surface** rather than managing it
- ✅ **Aligns with project simplification goals**
- ✅ **Reduces technical debt** and complexity

---

## Critical Vulnerabilities to Address

### 1. Custom Router Loading System 🚨
**Current Location:** `packages/core/src/utils/router.ts:270-277`  
**Vulnerability:** Arbitrary code execution via `CUSTOM_ROUTER_PATH`  
**Decision:** **REMOVE FEATURE ENTIRELY**

**Rationale:**
- Custom routing is an advanced feature with limited use cases
- Alternative routing methods exist (project-level, scenario-based)
- Complexity outweighs benefits for most users
- Security risk is unacceptable

**Remediation Steps:**
1. Remove `CUSTOM_ROUTER_PATH` configuration option
2. Delete custom router loading logic
3. Update documentation to remove feature references
4. Provide migration guide for affected users

**Alternative Solutions:**
- Use existing scenario-based routing
- Implement project-level routing configuration
- Use built-in routing rules instead of custom code

---

### 2. Statusline Script Execution System 🚨
**Current Location:** `packages/cli/src/utils/statusline.ts:166-209`  
**Vulnerability:** Arbitrary code execution via configurable script paths  
**Decision:** **REMOVE SCRIPT EXECUTION FEATURE**

**Rationale:**
- Dynamic script execution is inherently dangerous
- Static statusline configuration is sufficient for most use cases
- Feature complexity vs. utility ratio is poor
- Alternative approaches exist (CLI integrations, external tools)

**Remediation Steps:**
1. Remove `executeScript()` function entirely
2. Delete script-based statusline modules
3. Keep only built-in, safe statusline modules
4. Update configuration schema to remove script options

**Alternative Solutions:**
- Use built-in statusline modules (git, cwd, etc.)
- External CLI tools and shell integrations
- Starship prompt or similar safe alternatives
- Static configuration instead of dynamic scripts

---

### 3. Custom Transformer Loading System 🚨
**Current Location:** `packages/core/src/services/transformer.ts:82-111`  
**Vulnerability:** Arbitrary code execution via custom transformer paths  
**Decision:** **REMOVE CUSTOM TRANSFORMER LOADING**

**Rationale:**
- Built-in transformers cover 99% of use cases
- Custom transformer development is rare and advanced
- Security risk far outweighs flexibility benefits
- Plugin system adds unnecessary complexity

**Remediation Steps:**
1. Remove `registerTransformerFromConfig()` function
2. Delete transformer path loading logic
3. Keep only built-in transformers (anthropic, openai, gemini, etc.)
4. Update configuration to remove transformer path options

**Alternative Solutions:**
- Use built-in transformers only
- Contribute new transformers to core project
- Use environment variables for provider-specific configuration
- Submit pull requests for new transformer support

---

## Implementation Plan

### Phase 1: Feature Removal (Week 1)

#### Step 1: Remove Custom Router System
**Files to Modify:**
- `packages/core/src/utils/router.ts`
- `packages/shared/src/constants.ts`
- Configuration examples and documentation

**Actions:**
```typescript
// REMOVE this code from router.ts
const customRouterPath = configService.get("CUSTOM_ROUTER_PATH");
if (customRouterPath) {
  try {
    const customRouter = require(customRouterPath);
    req.tokenCount = tokenCount; 
    model = await customRouter(req, configService.getAll(), { event });
  } catch (error) {
    // error handling
  }
}
```

**Testing:**
- Verify default routing works
- Test scenario-based routing
- Validate project-level routing
- Ensure no regressions in core functionality

---

#### Step 2: Remove Statusline Script Execution
**Files to Modify:**
- `packages/cli/src/utils/statusline.ts`
- `packages/cli/src/utils/statusline.ts` (executeScript function)
- Configuration schema and validation

**Actions:**
```typescript
// REMOVE this entire function
async function executeScript(scriptPath: string, variables: Record<string, string>, options?: Record<string, any>): Promise<string> {
  // ... entire function removed
}

// UPDATE module type to exclude 'script'
type ModuleType = 'text' | 'git' | 'cwd' | 'command'; // remove 'script'
```

**Testing:**
- Test all built-in statusline modules
- Verify configuration validation
- Test CLI statusline output
- Ensure no script execution references remain

---

#### Step 3: Remove Custom Transformer Loading
**Files to Modify:**
- `packages/core/src/services/transformer.ts`
- Configuration validation and examples
- Documentation and API references

**Actions:**
```typescript
// REMOVE this entire method
async registerTransformerFromConfig(config: {
  path?: string;
  options?: any;
}): Promise<boolean> {
  // ... entire method removed
}

// UPDATE transformer registration to use only built-in transformers
```

**Testing:**
- Test all built-in transformers
- Verify transformer chain execution
- Validate configuration without custom paths
- Test provider functionality

---

### Phase 2: Code Cleanup (Week 2)

#### Step 4: Remove Dead Code and Dependencies
**Actions:**
1. Search for references to removed features
2. Update TypeScript interfaces and types
3. Remove related utility functions
4. Clean up configuration schemas
5. Update error messages and logging

**Files to Review:**
- All TypeScript files for feature references
- Configuration files and examples
- Test files and fixtures
- Documentation and comments

---

#### Step 5: Update Configuration System
**Actions:**
1. Remove `CUSTOM_ROUTER_PATH` from config schema
2. Remove script-based statusline configuration
3. Remove custom transformer path configuration
4. Update validation logic
5. Simplify configuration examples

**Configuration Changes:**
```json5
// REMOVE these configuration options
{
  "CUSTOM_ROUTER_PATH": "/path/to/router.js",  // REMOVE
  "StatusLine": {
    "modules": [
      {
        "type": "script",  // REMOVE
        "scriptPath": "/path/to/script.js"  // REMOVE
      }
    ]
  },
  "transformers": [
    {
      "path": "/path/to/transformer.js"  // REMOVE
    }
  ]
}
```

---

### Phase 3: Testing and Validation (Week 3)

#### Step 6: Security Testing
**Actions:**
1. Attempt to exploit removed RCE vectors (should fail)
2. Test configuration validation rejects removed options
3. Verify no code execution paths remain
4. Test with malicious inputs to ensure resilience

**Security Test Cases:**
```bash
# Test 1: Verify custom router path is rejected
echo '{"CUSTOM_ROUTER_PATH":"/tmp/malicious.js"}' > ~/.claude-code-router/config.json
ccr start
# Expected: Error or ignored, not code execution

# Test 2: Verify script statusline is rejected
echo '{"StatusLine":{"modules":[{"type":"script","scriptPath":"/tmp/exploit.js"}]}}' > config.json
ccr statusline
# Expected: Error or ignored, not code execution

# Test 3: Verify custom transformer path is rejected  
echo '{"transformers":[{"path":"/tmp/evil.js"}]}' > config.json
ccr start
# Expected: Error or ignored, not code execution
```

---

#### Step 7: Functional Testing
**Actions:**
1. Test all core functionality remains working
2. Verify routing system works without custom routers
3. Test statusline with built-in modules only
4. Validate transformer chain with built-in transformers
5. Test configuration management and validation

**Test Matrix:**
| Feature | Test Cases | Expected Result |
|---------|-----------|-----------------|
| Default Routing | Various request types | Routes correctly |
| Scenario Routing | Background, think, longContext | Routes correctly |
| Project-Level Routing | Project-specific configs | Routes correctly |
| Statusline | Built-in modules only | Displays correctly |
| Transformers | Built-in transformers only | Transforms correctly |
| Configuration | Invalid options rejected | Validation errors |

---

### Phase 4: Documentation and Migration (Week 4)

#### Step 8: Update Documentation
**Actions:**
1. Update SECURITY_REVIEW.md with remediation status
2. Remove feature documentation for deleted features
3. Update configuration guides
4. Create migration guide for affected users
5. Update API documentation

**Documentation Updates:**
- Remove custom router documentation
- Remove script-based statusline documentation  
- Remove custom transformer documentation
- Update security assessment
- Add simplification benefits section

---

#### Step 9: User Communication
**Actions:**
1. Create migration guide for affected users
2. Document breaking changes clearly
3. Provide alternative solutions
4. Update changelog and release notes
5. Communicate security improvements

**Communication Template:**
```markdown
## Security Update: Remote Code Execution Vulnerabilities Fixed

We've removed three features that presented security risks:
- Custom router loading
- Script-based statusline modules  
- Custom transformer loading

### What This Means for You

If you were using these features, here are alternatives:
- **Custom routing**: Use scenario-based or project-level routing
- **Script statuslines**: Use built-in modules or external tools
- **Custom transformers**: Use built-in transformers or contribute to core

### Security Improvements

These changes eliminate 3 critical RCE vulnerabilities and significantly 
improve the security posture of Claude Code Router.
```

---

## Expected Outcomes

### Security Improvements
- ✅ **Eliminates all 3 critical RCE vulnerabilities**
- ✅ **Reduces attack surface significantly**
- ✅ **Removes highest-risk features**
- ✅ **Improves overall security posture**

### Project Simplification Benefits
- ✅ **Reduces codebase complexity**
- ✅ **Decreases maintenance burden**
- ✅ **Eliminates edge cases and bugs**
- ✅ **Simplifies configuration and documentation**
- ✅ **Improves code readability**

### Performance Benefits
- ✅ **Faster startup (no dynamic code loading)**
- ✅ **Lower memory footprint**
- ✅ **Reduced dependency overhead**
- ✅ **Simpler error handling**

---

## Risk Assessment

### Low Risk Changes
- **Feature removal**: Users relying on removed features will need migration
- **Breaking changes**: Configuration changes required for some users
- **Functionality reduction**: Advanced use cases may be affected

### Mitigation Strategies
- **Clear communication**: Document breaking changes thoroughly
- **Migration support**: Provide guides and alternatives
- **Extended timeline**: Allow transition period for users
- **Community feedback**: Solicit input on removal decisions

---

## Success Criteria

### Security Criteria
- ✅ No remote code execution vectors remain
- ✅ All RCE vulnerabilities eliminated
- ✅ Security assessment updated to reflect improvements
- ✅ No new vulnerabilities introduced

### Functional Criteria  
- ✅ All core features work correctly
- ✅ Routing system functions without custom routers
- ✅ Statusline works with built-in modules only
- ✅ Transformer chain works with built-in transformers
- ✅ No regressions in existing functionality

### Project Simplification Criteria
- ✅ Reduced lines of code by ~15%
- ✅ Simplified configuration schema
- ✅ Removed complex dynamic loading systems
- ✅ Improved code maintainability
- ✅ Clearer project architecture

---

## Timeline

| Phase | Duration | Target Completion |
|-------|----------|-------------------|
| Phase 1: Feature Removal | 1 week | Week 1 |
| Phase 2: Code Cleanup | 1 week | Week 2 |
| Phase 3: Testing & Validation | 1 week | Week 3 |
| Phase 4: Documentation & Migration | 1 week | Week 4 |
| **Total** | **4 weeks** | **1 month** |

---

## Next Steps

1. **Immediate** (Week 1): Begin feature removal
2. **Short-term** (Weeks 2-3): Testing and validation  
3. **Medium-term** (Week 4): Documentation and release
4. **Long-term**: Monitor for security improvements and user feedback

---

## Conclusion

This remediation plan takes a **security-first approach** by eliminating dangerous features rather than attempting to mitigate their risks. The result will be a **simpler, more secure, and more maintainable** Claude Code Router project.

**Key Benefits:**
- 🚨 **Eliminates all critical RCE vulnerabilities**
- 🧹 **Significantly simplifies codebase**
- 📉 **Reduces maintenance burden**
- 🔒 **Improves overall security posture**
- ✨ **Aligns with project simplification goals**

**Risk Level After Remediation:** MEDIUM (down from CRITICAL)

---

**Plan Author:** Security Team  
**Plan Version:** 1.0  
**Last Updated:** 2025-02-05  
**Status:** Ready for Implementation