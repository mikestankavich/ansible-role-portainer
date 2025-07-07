# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## ⚠️ MANDATORY: Testing and Accuracy Standards

**NEVER claim "tests pass" or "ready for testing" without ACTUAL verification.**

1. **Always run tests before claiming success**:
   ```bash
   # Full test suite
   molecule test
   
   # Test specific scenario
   molecule test -s default
   
   # Quick iteration
   molecule create && molecule converge && molecule verify
   ```

2. **Report EXACT results**: "X tests passing, Y tests failing"
3. **NO false confidence** - if tests fail, report it immediately
4. **NO sweeping claims** without verification
5. **Always check idempotency**: Run converge twice, verify no changes

## Project Overview

This is an Ansible Galaxy role for installing and configuring Portainer using Docker. The role provides comprehensive Portainer management including:
- Container deployment and lifecycle management
- Admin user configuration
- Endpoint management (local and remote Docker hosts)
- Stack deployment via Portainer API
- Settings and registry configuration
- LDAP integration support

## Essential Context Documents

**🚨 CRITICAL: Always review these files before making changes:**

1. **`defaults/main.yml`** - All configurable variables with defaults
2. **`tasks/main.yml`** - Main task orchestration flow
3. **`tasks/endpoints.yml`** - Endpoint management implementation (shows API patterns)
4. **`tasks/stack_single.yml`** - Current stack implementation
5. **`meta/main.yml`** - Dependencies and Galaxy metadata
6. **`molecule/default/verify.yml`** - Test assertions

## Current Implementation Patterns

### Authentication Token Management
The role generates tokens in `tasks/main.yml`:
```yaml
- name: Generate authentication token
  uri:
    url: "{{ portainer_endpoint }}/auth"
    method: POST
    return_content: true
    body_format: json
    body: '{"Username": "{{ admin_user }}", "Password": "{{ admin_password }}"}'
  register: auth_token
```

### Endpoint ID Resolution
From `tasks/endpoints.yml`, the role maintains a fact that maps names to IDs:
```yaml
portainer_endpoint_ids: [{'name': 'local', 'id': 1}, {'name': 'remote1', 'id': 2}]
```

### API Interaction Patterns
- Uses `uri` module for JSON APIs
- Uses `curl` for multipart form data (due to Ansible uri module limitations)
- Always includes error checking after API calls

## Architecture Principles

### 1. Idempotency First
**Every task MUST be idempotent - running the role multiple times should not change the system state if nothing has changed.**

```yaml
# CORRECT: Check before modifying
- name: Check if stack exists
  uri:
    url: "{{ portainer_url }}/api/stacks"
    method: GET
  register: existing_stacks

- name: Create stack only if needed
  shell: curl -X POST...
  when: stack_name not in (existing_stacks.json | map(attribute='Name'))

# WRONG: Always creating
- name: Create stack
  shell: curl -X POST...  # Fails if exists
```

### 2. Token Lifecycle Management
**Handle token generation, validation, and renewal consistently.**

```yaml
# CORRECT: Validate token before use
- name: Check token validity
  uri:
    url: "{{ portainer_url }}/api/users/me"
    headers:
      Authorization: "Bearer {{ portainer_token }}"
    status_code: [200, 401]
  register: token_check
  ignore_errors: true

# WRONG: Assume token is valid
- name: Use token
  uri:
    headers:
      Authorization: "Bearer {{ portainer_token }}"  # May be expired
```

### 3. Consistent Error Handling
**All API interactions must handle errors gracefully.**

```yaml
- name: API call with error handling
  uri:
    url: "{{ portainer_url }}/api/endpoint"
    status_code: [200, 201]
  register: result
  failed_when:
    - result.status not in [200, 201]
    - "'already exists' not in result.json.message | default('')"
```

### 4. Environment Variable Format
**Portainer expects specific JSON format for environment variables.**

```yaml
# Transform dict to Portainer's expected format
stack_env: >-
  {{
    (item.env | default({}))
    | dict2items
    | map('combine', {'name': item.key, 'value': item.value})
    | list
  }}
```

## Testing Strategy

### Molecule Test Structure
```yaml
scenario:
  test_sequence:
    - destroy
    - create
    - prepare
    - converge
    - idempotence  # Critical for Ansible roles
    - verify
    - destroy
```

### Comprehensive Verification
- Verify container is running
- Check API accessibility
- Validate stack deployment
- Test service functionality
- Ensure idempotency (no changes on second run)

### Testing Commands
```bash
# IMPORTANT: Install molecule docker plugin first
uv add 'molecule-plugins[docker]'

# During development
molecule create
molecule converge
molecule verify

# Check idempotency
molecule idempotence

# Full test
molecule test

# IMPORTANT: Clean test state between runs
# Portainer data persists in Docker volumes. The full `molecule test` 
# includes automated cleanup, but for manual testing:
docker stop portainer 2>/dev/null || true
docker rm portainer 2>/dev/null || true
docker volume rm portainer-test-data 2>/dev/null || true
```

## Development Workflow

### 1. Implementation Checklist
- [ ] Review existing patterns in tasks/
- [ ] Add variables to defaults/main.yml
- [ ] Implement idempotent task logic
- [ ] Add error handling
- [ ] Clean up temporary resources
- [ ] Update README
- [ ] Add molecule tests
- [ ] Verify idempotency

### 2. Manual API Testing
```bash
# Get token
TOKEN=$(curl -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"password"}' | jq -r .jwt)

# Test API endpoints
curl -H "Authorization: Bearer $TOKEN" http://localhost:9000/api/stacks
```

## Code Style Guidelines

### YAML Best Practices
- Explicit, descriptive task names
- Group related tasks with `block`
- Use `failed_when`/`changed_when` for shell tasks
- Clean up temporary files
- Document complex logic with comments

### Variable Naming
- Prefix with `portainer_`
- Use descriptive names
- Group related variables
- Document in defaults/main.yml

### Error Messages
```yaml
# GOOD: Actionable error
fail_msg: "Stack '{{ item.name }}' references unknown endpoint '{{ item.environment_name }}'. Available: {{ portainer_endpoint_ids | map(attribute='name') | join(', ') }}"

# BAD: Generic error
fail_msg: "Invalid configuration"
```

## Project-Specific Notes

1. **Portainer API**: Targets Portainer CE 2.x
2. **Compose Type**: Use `Type=2` for docker-compose format
3. **Token Expiry**: JWT tokens expire after 8 hours
4. **Local Endpoint**: Uses empty string for URL
5. **Stack Updates**: Use PUT with full definition
6. **Environment Format**: JSON array with name/value objects
7. **Multipart Forms**: Use curl for file uploads

## Common Pitfalls

1. **Multipart Form Data**: Ansible uri module doesn't handle well - use curl
2. **Compose Format**: Must be string, not YAML object
3. **Endpoint Names vs IDs**: Always resolve to ID
4. **Token Expiration**: Validate before use
5. **Temporary Files**: Always clean up

## Success Criteria

Stack implementation is complete when:
- ✅ Creates stacks idempotently
- ✅ Updates only when changed
- ✅ Handles environment variables
- ✅ Supports force redeploy
- ✅ Works with multiple endpoints
- ✅ All molecule tests pass
- ✅ Idempotency verified
- ✅ README updated
- ✅ No resource leaks

The systematic configuration issues that were causing repeated "importing .js instead of .ts" and module
resolution problems are completely solved:

1. ✅ Module Resolution: All TypeScript imports standardized
2. ✅ Package Exports: Vite config pointing to correct TypeScript sources
3. ✅ Import Consistency: No more .js extensions in TypeScript files
4. ✅ Type Configuration: Web Bluetooth types properly configured

SOLID 3-LEVEL TESTING FOUNDATION

- Level 1: 3/3 tests passing - Fast business logic validation
- Level 2: 2/2 tests passing - Real hardware validation
- Level 3: Web Bluetooth mock working (no system dialogs)

CLEAN ARCHITECTURE ACHIEVED

- One CS108 simulator using constants/builders/helpers consistently
- Pure transport abstraction with debug hooks for byte monitoring
- Store-first architecture with proper lifecycle management

NEXT CONTEXT: Level 3 Integration Debugging

The remaining work is targeted debugging, not systematic fixes:
- Web app connect function not triggering "Connecting" state
- Need to trace why button doesn't change after click
- Specific integration issue, not architectural problem

Ready for context reset with solid foundation and clear next steps.