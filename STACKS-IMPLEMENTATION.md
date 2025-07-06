# Implementation Notes for ansible-role-portainer

## Current State Analysis

Based on reviewing the provided files, here's the current state of the stack implementation:

### What Already Works

1. **Authentication Flow**
    - Token generation is implemented in `tasks/main.yml`
    - Token is stored in `auth_token` variable
    - Token format: `{{ (auth_token.content|from_json).jwt }}`

2. **Endpoint Management**
    - `tasks/endpoints.yml` successfully manages endpoints
    - Creates `portainer_endpoint_ids` fact mapping names to IDs
    - Uses curl for multipart form data (established pattern)

3. **Basic Stack Creation**
    - `tasks/stack_single.yml` can create stacks
    - Templates compose files
    - Formats environment variables
    - Uses curl for stack creation API

### Issues in Current Implementation

1. **Token Access**
    - Stack tasks reference `portainer_token` but token is stored as `auth_token`
    - No token validation before use
    - Token could be expired (8-hour TTL)

2. **Idempotency Problems**
   ```yaml
   # Current code always tries to create
   - name: Create stack if it does not exist
     shell: curl -X POST...
     when: existing_stack_id is not defined
   ```
    - Doesn't handle updates when compose file changes
    - Force redeploy uses wrong endpoint

3. **Error Handling**
    - No error checking on curl commands
    - No validation of API responses
    - Silent failures possible

4. **Resource Cleanup**
    - Temporary files not cleaned up
    - Could leave artifacts in /tmp/

5. **Variable Naming**
    - Inconsistent: `portainer_url` vs `portainer_endpoint`
    - Missing `portainer_` prefix on some vars

## Recommended Improvements

### 1. Fix Token Management

```yaml
# Add to tasks/stacks.yml
- name: Ensure token is available
  set_fact:
    portainer_token: "{{ (auth_token.content|from_json).jwt }}"
  when: auth_token is defined

- name: Validate token
  uri:
    url: "{{ portainer_url }}/api/users/me"
    headers:
      Authorization: "Bearer {{ portainer_token }}"
    status_code: [200, 401]
  register: token_check
  ignore_errors: true

- name: Regenerate token if needed
  include_tasks: generate_token.yml
  when: token_check.status == 401
```

### 2. Implement Proper Idempotency

```yaml
# Check if content changed
- name: Get current stack details
  uri:
    url: "{{ portainer_url }}/api/stacks/{{ existing_stack_id }}"
    headers:
      Authorization: "Bearer {{ portainer_token }}"
  register: current_stack
  when: existing_stack_id is defined

- name: Compare stack content
  set_fact:
    stack_needs_update: >-
      {{
        current_stack.json.StackFileContent != lookup('template', item.compose_src)
        or current_stack.json.Env != formatted_env
      }}
  when: existing_stack_id is defined

# Update if changed
- name: Update stack
  uri:
    url: "{{ portainer_url }}/api/stacks/{{ existing_stack_id }}"
    method: PUT
    headers:
      Authorization: "Bearer {{ portainer_token }}"
    body_format: json
    body:
      stackFileContent: "{{ lookup('template', item.compose_src) }}"
      env: "{{ formatted_env }}"
  when:
    - existing_stack_id is defined
    - stack_needs_update
```

### 3. Add Error Handling

```yaml
- name: Create stack with error handling
  shell: |
    response=$(curl -s -w '\n%{http_code}' -X POST \
      "{{ portainer_url }}/api/stacks?endpointId={{ endpoint_id }}" \
      -H "Authorization: Bearer {{ portainer_token }}" \
      -F "Name={{ item.name }}" \
      -F "StackFile=@/tmp/{{ item.name }}.yml" \
      -F "Env=@/tmp/{{ item.name }}.env.json;type=application/json" \
      -F "Method=string" \
      -F "Type=2")
    
    http_code=$(echo "$response" | tail -n1)
    body=$(echo "$response" | head -n-1)
    
    if [ "$http_code" != "200" ] && [ "$http_code" != "201" ]; then
      echo "Error: HTTP $http_code - $body" >&2
      exit 1
    fi
    
    echo "$body"
  register: create_result
  failed_when: create_result.rc != 0
  changed_when: true
```

### 4. Resource Cleanup

```yaml
- name: Clean up temporary files
  file:
    path: "{{ item }}"
    state: absent
  loop:
    - "/tmp/{{ stack_item.name }}.yml"
    - "/tmp/{{ stack_item.name }}.env.json"
  loop_control:
    loop_var: cleanup_item
  when: not ansible_check_mode
  tags: [always]
```

### 5. Improved Variable Structure

Update `defaults/main.yml`:
```yaml
# Portainer API settings
portainer_api_url: "http://localhost:9000"
portainer_api_timeout: 30

# Stack defaults
portainer_stack_default_endpoint: "local"
portainer_stack_temp_dir: "/tmp"
portainer_stack_force_redeploy: false
```

## Testing Improvements

### 1. Add Idempotency Test

```yaml
# molecule/default/verify.yml additions
- name: Save initial stack state
  uri:
    url: "http://localhost:9000/api/stacks"
    headers:
      Authorization: "Bearer {{ verify_token.json.jwt }}"
  register: initial_stacks

- name: Run role again (should be idempotent)
  include_role:
    name: ansible-role-portainer

- name: Get final stack state
  uri:
    url: "http://localhost:9000/api/stacks"
    headers:
      Authorization: "Bearer {{ verify_token.json.jwt }}"
  register: final_stacks

- name: Verify no changes
  assert:
    that:
      - initial_stacks.json == final_stacks.json
    fail_msg: "Stack state changed during idempotent run"
```

### 2. Test Stack Updates

```yaml
# molecule/stack-update/converge.yml
- name: Deploy initial stack
  include_role:
    name: ansible-role-portainer
  vars:
    portainer_stacks:
      - name: test-update
        compose_content: |
          version: '3'
          services:
            web:
              image: nginx:1.19

- name: Update stack
  include_role:
    name: ansible-role-portainer
  vars:
    portainer_stacks:
      - name: test-update
        compose_content: |
          version: '3'
          services:
            web:
              image: nginx:1.20  # Changed version
```

## File Structure Recommendations

```
tasks/
├── main.yml                    # Existing orchestration
├── stacks.yml                  # New: Main stack logic
├── stacks/
│   ├── validate_token.yml      # Token validation/renewal
│   ├── prepare_endpoints.yml   # Ensure endpoint IDs available
│   ├── deploy_single.yml       # Deploy one stack (improved)
│   └── cleanup.yml            # Resource cleanup
```

## Migration Path

1. **Phase 1**: Fix critical issues
    - Token access bug
    - Add error handling
    - Clean up temp files

2. **Phase 2**: Improve idempotency
    - Detect content changes
    - Implement proper updates
    - Fix force redeploy

3. **Phase 3**: Enhanced testing
    - Add idempotency tests
    - Test update scenarios
    - Test error cases

## Known Portainer API Quirks

1. **Stack Creation**: Must use multipart form, not JSON
2. **Environment Variables**: Must be JSON array format
3. **Stack Updates**: Requires full content, not patches
4. **Force Redeploy**: Use `/api/stacks/{id}/redeploy` not recreate
5. **Endpoint IDs**: Start at 1, not 0
6. **Compose Content**: Must be string, not YAML structure