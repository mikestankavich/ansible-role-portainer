# NEXT STEPS - Ansible Portainer Stack Implementation

## Current Status
**🎉 MAJOR BREAKTHROUGH ACHIEVED!** 

We successfully fixed all molecule testing infrastructure issues and reached our stack management code. Portainer is deploying, authenticating, and we're now hitting our actual stack implementation.

## Immediate Next Action
**Fix deprecated 'warn' parameter in endpoints.yml** - This is blocking us from reaching the stack deployment code.

### The Issue
```
failed: [instance] => {"msg": "Unsupported parameters for (ansible.legacy.command) module: warn. Supported parameters include: _raw_params, _uses_shell, argv, chdir, creates, executable, expand_argument_vars, removes, stdin, stdin_add_newline, strip_empty_ends."}
```

### What to Do
1. **Find and fix the deprecated `warn` parameter** in `tasks/endpoints.yml`
2. **Test molecule converge again** - should reach stack deployment code
3. **Validate stack creation works** - our main implementation goal
4. **Fix any remaining issues** in stack_single.yml (idempotency, error handling, cleanup)

## Working Test Environment
- **Container**: Ubuntu 22.04 with Docker socket mounted
- **Portainer**: Running at http://192.168.215.3:9000/api  
- **Admin**: admin / SuperSecure123!
- **Token flow**: Working correctly
- **Test command**: `uv run molecule converge`

## Key Files to Focus On
- `tasks/endpoints.yml` - Fix deprecated warn parameter
- `tasks/stack_single.yml` - Our main stack implementation
- `tasks/stacks.yml` - Token handling (working)
- `molecule/default/playbook.yml` - Test configuration (working)

## Success Criteria
When we fix the warn parameter, we should see:
1. Endpoints processing complete
2. Stack deployment attempt
3. Actual testing of our portainer_stacks functionality

## Context Files to Read
- `CLAUDE.md` - Project guidelines and patterns
- `STACKS-IMPLEMENTATION.md` - Current implementation analysis
- Current git status shows we're on `add-stacks` branch with good progress

**Ready to test actual stack management!** 🚀