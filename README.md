# AAPRFE-409 Test Repository

This repository is for testing the **galaxy_update_on_launch** feature implemented in AAPRFE-409.

## Purpose

Demonstrates and validates the new `galaxy_update_on_launch` project setting that bypasses the ansible-galaxy response cache to ensure collections are resolved against the latest available versions from Galaxy/Automation Hub.

## Problem Being Solved

In environments where execution environment containers persist across many project updates (e.g., Controller task pods with long-lived EE sidecars on OpenShift), `ansible-galaxy` maintains a response cache at `~/.ansible/galaxy_cache/api.json` with a 24-hour TTL.

This cache can become stale and cause project sync to install **older collection versions** even after newer versions have been published to Automation Hub.

## Solution

New project setting: **"Update Galaxy cache on launch"**

When enabled, passes `--no-cache` to `ansible-galaxy collection install` during project updates.

## Test Setup

### 1. Create Project in AAP/AWX

```
Name: AAPRFE-409 Test
Organization: Default
Source Control Type: Git
Source Control URL: https://github.com/jonnyfiveiq/rfe409-test.git
```

### 2. Create Job Template

```
Name: Test Galaxy Cache
Inventory: Demo Inventory (or any localhost inventory)
Project: AAPRFE-409 Test
Playbook: test-galaxy-cache.yml
```

## Testing Procedure

### Test 1: Without Cache Bypass (Default Behavior)

1. **Disable** the "Update Galaxy cache on launch" checkbox on the project
2. Trigger a project update
3. Check the project update job output for:
   ```
   ansible-galaxy collection install -r collections/requirements.yml
   ```
   ⚠️ **Should NOT see `--no-cache` flag**

4. Run the job template
5. Verify collections were installed correctly

### Test 2: With Cache Bypass (New Feature)

1. **Enable** the "Update Galaxy cache on launch" checkbox on the project
2. Trigger a project update
3. Check the project update job output for:
   ```
   ansible-galaxy collection install --no-cache -r collections/requirements.yml
   ```
   ✅ **Should SEE `--no-cache` flag**

4. Run the job template
5. Verify collections were installed correctly

### What to Look For

**In Project Update Job Output:**

Without feature:
```
TASK [Fetch galaxy collections from collections/requirements.(yml/yaml)]
...
ansible-galaxy collection install -r /tmp/.../collections/requirements.yml -p ...
```

With feature enabled:
```
TASK [Fetch galaxy collections from collections/requirements.(yml/yaml)]
...
ansible-galaxy collection install --no-cache -r /tmp/.../collections/requirements.yml -p ...
```

**In Job Template Output:**

The playbook will verify that all collections are installed and usable by importing modules from each collection.

## Collections Used

This test uses the following collections (all unpinned to demonstrate latest version resolution):

- **ansible.posix** - Ansible maintained, POSIX utilities
- **ansible.utils** - Ansible maintained, utility plugins
- **community.general** - Large community collection, frequently updated
- **community.crypto** - Cryptography collection, smaller/faster for testing

## API Testing

You can also test via API:

```bash
# Get project ID
PROJECT_ID=$(curl -sk -u admin:password \
  "https://aap.example.com/api/controller/v2/projects/" \
  | jq -r '.results[] | select(.name == "AAPRFE-409 Test") | .id')

# Check current setting
curl -sk -u admin:password \
  "https://aap.example.com/api/controller/v2/projects/${PROJECT_ID}/" \
  | jq '{id, name, galaxy_update_on_launch}'

# Enable the feature
curl -sk -X PATCH -u admin:password \
  -H "Content-Type: application/json" \
  "https://aap.example.com/api/controller/v2/projects/${PROJECT_ID}/" \
  -d '{"galaxy_update_on_launch": true}'

# Trigger project update
curl -sk -X POST -u admin:password \
  "https://aap.example.com/api/controller/v2/projects/${PROJECT_ID}/update/"
```

## Database Verification

If you have direct database access:

```sql
-- Check the setting in database
SELECT id, name, galaxy_update_on_launch 
FROM main_project 
WHERE name LIKE '%AAPRFE-409%';

-- Enable it
UPDATE main_project 
SET galaxy_update_on_launch = true 
WHERE name LIKE '%AAPRFE-409%';
```

## Expected Results

### ✅ Success Indicators

1. **Database**: Field stores true/false correctly
2. **API**: Field exposed in `/api/controller/v2/projects/` responses
3. **Playbook**: `--no-cache` appears in ansible-galaxy command when enabled
4. **Behavior**: Collections install successfully in both modes
5. **Performance**: Minimal impact (small number of HTTP GETs vs. full git clone)

### When to Use This Feature

Enable "Update Galaxy cache on launch" when:

- ✅ You've just published a new collection version to Private Automation Hub
- ✅ Pinning to a specific version fails with resolution errors
- ✅ You need to guarantee the latest matching version is installed
- ✅ Testing/development environments where fresh metadata matters

Leave it **disabled** (default) when:

- ✅ Production environments with pinned collection versions
- ✅ You want faster project syncs (cache reduces HTTP requests)
- ✅ Your collections don't change frequently

## Related Resources

- **Jira**: [AAPRFE-409](https://redhat.atlassian.net/browse/AAPRFE-409)
- **AWX PR**: [ansible/awx#16737](https://github.com/ansible/awx/pull/16737)
- **ansible-ui PR**: [ansible/ansible-ui#3740](https://github.com/ansible/ansible-ui/pull/3740)

## License

Apache-2.0

## Author

Created for testing AAPRFE-409 implementation.
