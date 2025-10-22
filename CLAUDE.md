# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Zuul CI configuration repository for Sean Mooney's personal CI system. It defines jobs, pipelines, nodesets, and playbooks for running OpenStack-related continuous integration tests, particularly focused on NFV (Network Function Virtualization) testing with DevStack and Tempest.

## Architecture

### Zuul Configuration Structure

The repository follows Zuul's configuration layout:

- `zuul.d/` - Contains all Zuul configuration files:
  - `jobs.yaml` - Job definitions (base jobs, DevStack jobs, Tempest jobs)
  - `projects.yaml` - Project pipeline assignments
  - `pipelines.yaml` - Pipeline definitions (manual-ci, automatic-ci, nightly-ci, weekly-ci, check)
  - `nodesets.yaml` - Node and multi-node topology definitions

- `playbooks/` - Ansible playbooks executed by jobs:
  - `base/` - Base setup playbooks (pre-run for environment setup, post-run for logs/SSH)
  - `tempest/` - Tempest-specific playbooks (run and post playbooks)

### Job Hierarchy

Jobs inherit from each other in this order:
1. `base` - Root job with basic pre/post playbooks
2. `dsvm-minimal-base` - Inherits from upstream `devstack` job
3. `dsvm-base` - Adds Python 3 and core OpenStack services (Cinder, etc.)
4. `dsvm-tempest-base` - Adds Tempest test framework
5. Specific test jobs (e.g., `tempest-full`, `tempest-multinode-full-py3`, `devstack-plugin-ceph-multinode-tempest-py3`)

### Pipeline Types

- `check` - GitHub PR checks
- `automatic-ci` - Runs on patchset creation and "recheck" comments (for OpenDev.org)
- `manual-ci` - Triggered by "seans-manual-ci: recheck" or "check experimental" comments
- `nightly-ci` - Timer-based, runs daily at 05:17 UTC
- `weekly-ci` - Timer-based, runs Saturdays at 08:00

## Common Development Commands

### Python Virtual Environment

**IMPORTANT**: Always use the local virtual environment for Python tools to avoid system dependency issues.

```bash
# Create virtual environment (if not exists)
python3 -m venv .venv

# Activate virtual environment
source .venv/bin/activate

# Install tools
.venv/bin/pip install zuul-client

# Use tools directly without activation
.venv/bin/zuul-client --help
```

### Linting and Validation

```bash
# Run all linters (yamllint and ansible-lint)
tox -e linters

# Check system dependencies
tox -e bindep

# Run specific linter
tox -e venv -- yamllint zuul.d/
tox -e venv -- ansible-lint
```

### Testing Individual Components

```bash
# Validate YAML syntax for specific files
yamllint -s zuul.d/jobs.yaml

# Test ansible playbook syntax (requires ansible)
ansible-playbook --syntax-check playbooks/base/pre.yaml
```

### Debugging Configuration Errors

**Zuul API Endpoints for Debugging:**

```bash
# Check configuration errors for the tenant
curl https://zuul.teim.app/api/tenant/main/config-errors

# View tenant information
curl https://zuul.teim.app/api/tenant/main

# Check Zuul instance info
curl https://zuul.teim.app/api/info
```

**Common Configuration Issues:**
- Connection name mismatches (e.g., `opendev` vs `opendev.org`)
- Missing secret definitions
- Invalid pipeline configurations
- Undefined nodesets or jobs

**Fix Workflow:**
1. Check API for errors: `curl https://zuul.teim.app/api/tenant/main/config-errors`
2. Identify file and line number from error message
3. Fix the configuration issue
4. Commit and push changes
5. Monitor Zuul scheduler logs for tenant reload
6. Verify errors cleared: Re-check API endpoint

## Key Configuration Details

### Nodesets

- `openstack-single-node-focal` - Single Ubuntu Focal node (default for most jobs)
- `openstack-two-node-focal` - Two-node setup (controller + compute)
- `nfv-multi-numa-multinode` - Multi-node NFV topology with compute group
- Additional nodesets for CentOS 8, Ubuntu Xenial, Bionic, Jammy

### DevStack Service Configuration

Jobs configure DevStack services via `devstack_services` and `devstack_localrc` variables. Common patterns:

- `LIBVIRT_TYPE: kvm` - Use KVM virtualization
- `USE_PYTHON3: true` - Python 3 support
- Services are explicitly enabled/disabled per job (Cinder, Tempest, dstat, etc.)
- Multi-node jobs use `group-vars.subnode` for compute node configuration

### Role Dependencies

Jobs pull roles from external Zuul repositories:
- `zuul/zuul-jobs` - Standard Zuul roles
- `openstack/openstack-zuul-jobs` - OpenStack-specific roles
- `openstack/project-config` - OpenStack project configuration
- `openstack/devstack` - DevStack roles
- `openstack/tempest` - Tempest roles

## Modifying Jobs

When creating or modifying jobs:

1. **Job Inheritance**: Use `parent:` to inherit from existing jobs rather than duplicating configuration
2. **Required Projects**: Add projects that need to be checked out in `required-projects`
3. **Timeout**: Default is 1800s (30 min); Tempest jobs use 7200s (2 hours)
4. **Branch Filters**: Use `branches:` with regex patterns (e.g., `^(?!stable/ocata).*$`)
5. **Nodesets**: Reference by name from `nodesets.yaml`
6. **Variables**: Define in `vars:` section; use `group-vars:` for multi-node specific config

## Managing Secrets

Zuul secrets are encrypted per-project and stored in the configuration repository.

### Creating/Updating Secrets

1. **Install zuul-client** (use venv):
   ```bash
   python3 -m venv .venv
   .venv/bin/pip install zuul-client
   ```

2. **Encrypt secret data**:
   ```bash
   # From stdin
   echo -n "secret-value" | .venv/bin/zuul-client --zuul-url https://zuul.teim.app \
     encrypt --tenant main --project SeanMooney/ci-sean-mooney

   # From file
   .venv/bin/zuul-client --zuul-url https://zuul.teim.app \
     encrypt --tenant main --project SeanMooney/ci-sean-mooney --infile secret.txt
   ```

3. **Add to `zuul.d/secrets.yaml`**:
   ```yaml
   - secret:
       name: my-secret
       data:
         secret_key: !encrypted/pkcs1-oaep
           - <encrypted-line-1>
           - <encrypted-line-2>
   ```

4. **Reference in jobs**:
   ```yaml
   - job:
       name: my-job
       secrets:
         - name: variable_name
           secret: my-secret
   ```

### Secret Management Notes

- Each project has its own RSA keypair for encryption
- Public keys are available at: `https://zuul.teim.app/api/tenant/main/key/SeanMooney/ci-sean-mooney.pub`
- Secrets are only available to playbooks in the job that declares them (not child jobs)
- Use structured data (nested dicts/lists) for complex secrets like `zuul_log_targets`

## Log Collection

Logs are automatically collected via:
- `zuul_copy_output` variable defines which files/directories to collect
- `extensions_to_txt` converts specific file types to .txt for web viewing
- ARA (Ansible Run Analysis) reports generated as HTML in `ara-report/`
- Logs published to MinIO S3 at https://minio-api.teim.app/zuul-logs/

## Pre-commit Hooks

The repository uses pre-commit for automated checks. Configuration in `.pre-commit-config.yaml`.

## System Dependencies

System packages required (via bindep.txt):
- libffi development libraries
- OpenSSL development libraries
- RE2 development libraries

These are platform-specific and automatically resolved by bindep.
