# PyTorch POWER Backend CI/CD

This repository contains the GitHub Actions workflow for building and testing PyTorch on IBM POWER architecture (ppc64le) with full integration to the PyTorch HUD (Heads-Up Display) via the Cross-Repository CI Relay (CRCR) system.

## Architecture Overview

The CI/CD pipeline uses a hybrid approach to work around GitHub Actions' lack of native ppc64le runner support:

```
GitHub Event (repository_dispatch from pytorch/pytorch)
      │
      ▼
Self Hosted PPC64 Machine
      │
      ├── Checkout this Downstream repo
      ├── Checkout PyTorch
      ├── Build Power Whell and Run Tests
      ├── Upload artifcats and test results
      └── Report completed to HUD ✨
            │
            ▼
      Results visible on https://hud.pytorch.org
```

## CRCR L2 Integration

This repository is configured for **Level 2 (L2)** integration with PyTorch's Cross-Repository CI Relay system, which provides:

✅ **Automatic Dispatch**: Receives `repository_dispatch` events from pytorch/pytorch PRs  
✅ **HUD Reporting**: Build and test results appear on https://hud.pytorch.org  
✅ **OIDC Authentication**: Secure callback authentication using GitHub's OIDC tokens  
✅ **Test Metrics**: Detailed pass/fail/skip counts visible in HUD  
✅ **Artifact Links**: Direct links to build artifacts from HUD interface  

### How It Works

```
pytorch/pytorch PR (opened/synchronized)
    ↓
CRCR Webhook Lambda (checks allowlist)
    ↓
repository_dispatch → TorchPowerCI/pytorch-power-backend
    ↓
Workflow runs on self hosted POWER machine
    ↓
Callbacks (in_progress → completed) → CRCR Callback Lambda
    ↓
DynamoDB → ClickHouse → PyTorch HUD
```

### Viewing Results on HUD

After a workflow run completes:

1. **Wait 30-60 seconds** for ClickHouse propagation
2. **Visit PyTorch HUD**: https://hud.pytorch.org
3. **Search by PR number** from pytorch/pytorch
4. **Look for your results**: 
   - Repository: `TorchPowerCI/pytorch-power-backend`
   - Job: `power-build-and-test`
   - Status: `in_progress` → `completed`
   - Conclusion: `success` or `failure`
   - Test results: passed/failed/skipped counts
   - Artifact URL: Link to GitHub Actions run

**Example HUD Query**:
```
https://hud.pytorch.org/api/clickhouse/crcr_pr_results?parameters=%7B%22pr%22%3A<PR_NUMBER>%7D
```

## Workflow Details

The workflow (`power-crcr-ci.yml`) performs the following steps:

1. **Self-hosted PPC64 Runner**:
   - Receives `repository_dispatch` event from pytorch/pytorch
   - Checks out the PyTorch source code at the dispatched SHA
   - Create virtual environment for PyTorch build
   - Build PyTorch and Run Tests
   - Runs smoke Test
   - Collect test results
   - Uploads build artifacts to GitHub Actions


## Prerequisites

### 1. Allowlist Configuration (Required for HUD Integration)

**Status**: ⏳ Pending addition to pytorch/pytorch allowlist

This repository must be added to the CRCR allowlist in `pytorch/pytorch/.github/allowlist.yaml`:

```yaml
L2:
  - TorchPowerCI/pytorch-power-backend
```

**To request allowlist addition**:
1. Open an issue in pytorch/pytorch
2. Reference: https://github.com/pytorch/pytorch/issues/175022
3. Provide: Repository name, desired level (L2), use case (POWER backend CI)

**Allowlist Levels**:

| Level | Dispatch | HUD Reporting | Upstream Check Runs |
|-------|----------|---------------|---------------------|
| L1    | ✅       | ❌            | ❌                  |
| L2    | ✅       | ✅            | ❌                  |
| L3    | ✅       | ✅            | ✅ (with label)     |
| L4    | ✅       | ✅            | ✅ (always)         |

### 2. GitHub Secrets

Configure the following secrets in your GitHub repository:

#### `POWER_SSH_USER`
- **Description**: SSH username for connecting to the Power LPAR
- **Example**: `pytorch-ci`
- **Setup**: Settings → Secrets and variables → Actions → New repository secret

#### `POWER_SSH_PRIVATE_KEY`
- **Description**: SSH private key for authentication to the Power LPAR
- **Format**: Full private key including headers
- **Example**:
  ```
  -----BEGIN OPENSSH PRIVATE KEY-----
  b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
  ...
  -----END OPENSSH PRIVATE KEY-----
  ```
- **Setup**: 
  1. Generate SSH key: `ssh-keygen -t rsa -b 4096 -C "pytorch-ci"`
  2. Copy public key to LPAR: `ssh-copy-id user@ailiblpar1.pperf.tadn.ibm.com`
  3. Add private key as GitHub secret

### 3. Power LPAR Requirements

The Power LPAR must have:

- Python 3.8 or later
- `python3-venv` package
- Build tools (gcc, g++, make)
- Git (for submodules)
- Sufficient disk space (~10GB recommended)


## Triggering the Workflow

### Automatic Triggers (via CRCR)

Once added to the allowlist, the workflow automatically triggers when:

- A PR is **opened** in pytorch/pytorch
- A PR is **synchronized** (new commits pushed) in pytorch/pytorch
- A PR is **closed** in pytorch/pytorch (triggers cancellation)

### Manual Testing

For testing without CRCR integration:

```bash
# Trigger via workflow_dispatch
gh workflow run power-crcr-ci.yml
```

**Note**: Manual triggers won't appear on HUD - only real `repository_dispatch` events from pytorch/pytorch are reported.

## Artifacts

Build artifacts (PyTorch wheel files) are:

1. **Uploaded to GitHub Actions**: Available on workflow run page
2. **Linked in HUD**: Direct link from HUD interface to artifacts

Artifact naming: `power-l1-wheel-<run_id>`

## Monitoring & Debugging

### Check Workflow Status

1. **GitHub Actions**: https://github.com/TorchPowerCI/pytorch-power-backend/actions
2. **PyTorch HUD**: https://hud.pytorch.org (after allowlist addition)

### Verify Callbacks

Check workflow step summaries for:

- ✅ `Report in_progress to PyTorch HUD` - Should succeed
- ✅ `Report completed to PyTorch HUD` - Should succeed with test results

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| No dispatch received | Not in allowlist | Contact PyTorch team |
| Callback rejected | Missing OIDC permission | Already configured ✅ |
| HUD shows no data | ClickHouse delay | Wait 60 seconds, refresh |
| SSH connection fails | Wrong credentials | Verify secrets |
| Build fails on LPAR | Missing dependencies | Check LPAR setup |

## Troubleshooting

### SSH Connection Issues

**⚠️ IMPORTANT: Network Accessibility Limitation**

The POWER LPAR (`ailiblpar1.pperf.tadn.ibm.com`) is an **internal IBM system** that is **NOT accessible from GitHub Actions runners** (which run on public cloud infrastructure). This causes SSH connection failures like:

```
ssh: connect to host ailiblpar1.pperf.tadn.ibm.com port 22: Connection timed out
```

**Solutions:**

1. **Self-Hosted Runner (Recommended)**
   - Deploy a GitHub Actions self-hosted runner inside IBM's network
   - Runner can access internal LPAR systems
   - Update workflow to use: `runs-on: [self-hosted, linux, power]`
   - Setup guide: https://docs.github.com/en/actions/hosting-your-own-runners

2. **VPN/Bastion Host**
   - Set up a VPN gateway or bastion host accessible from GitHub
   - Route SSH connections through the gateway
   - Requires network infrastructure changes

3. **Expose LPAR (Not Recommended)**
   - Make LPAR accessible from GitHub's IP ranges
   - Security risk - requires firewall rules and hardening
   - GitHub IP ranges: https://api.github.com/meta

4. **Alternative Architecture**
   - Use a publicly accessible build server
   - Server pulls code and dispatches to internal LPAR
   - More complex but maintains security

**Current Status:**
- ❌ Configured a self hosted ppc64 runner.
- Testing if the CI flow runs on slef hiosted machine
- Next - Test if Workflow structure and HUD integration are correct


**Troubleshooting Steps:**
- Verify `POWER_SSH_USER` and `POWER_SSH_PRIVATE_KEY` secrets are set
- Check SSH key in `~/.ssh/authorized_keys` on LPAR
- Test connectivity: `ssh -v user@ailiblpar1.pperf.tadn.ibm.com` from runner
- Review firewall rules for GitHub Actions IP ranges

### Build Failures
- Check LPAR resources (CPU, memory, disk space)
- Verify all build dependencies are installed
- Review build logs in GitHub Actions

### HUD Integration Issues
- Verify repository is in allowlist
- Check workflow has `id-token: write` permission (already configured ✅)
- Confirm callbacks succeeded in step summaries
- Wait 60 seconds for ClickHouse propagation

### Cleanup Issues
- Workflow automatically cleans up after each run
- Manual cleanup: SSH to LPAR and remove `/tmp/pytorch-ci-*` directories

## Security Considerations

- SSH private keys stored as GitHub encrypted secrets
- OIDC tokens used for HUD callback authentication
- Workflow uses `StrictHostKeyChecking=no` for automation
- Remote workspace directories unique per run
- Cleanup steps run even on failure

## Example Workflow Run

**Successful Run**: https://github.com/TorchPowerCI/pytorch-power-backend/actions/runs/28492375238/job/84451575059

This run demonstrates:
- ✅ Dispatch received from pytorch/pytorch
- ✅ in_progress callback sent
- ✅ Build and test on POWER LPAR
- ✅ completed callback with test results
- ⏳ HUD integration (pending allowlist addition)

## Contributing

When modifying the workflow:

1. Test changes in a fork first
2. Ensure all secrets are properly referenced
3. Verify cleanup steps execute correctly
4. Test both callback steps succeed
5. Update this README if new secrets or prerequisites are added

## Related Resources

- **PyTorch HUD**: https://hud.pytorch.org
- **CRCR RFC**: https://github.com/pytorch/pytorch/issues/175022
- **CRCR Relay Lambda**: https://github.com/pytorch/test-infra/tree/main/aws/lambda/cross_repo_ci_relay
- **Callback Action**: https://github.com/pytorch/test-infra/tree/main/.github/actions/cross-repo-ci-relay-callback
- **Reference Implementation**: https://github.com/TorchedHat/pytorch-redhat-ci

## Status

- ✅ Workflow configured for L2 integration
- ✅ OIDC authentication enabled
- ✅ Callbacks implemented (in_progress + completed)
- ✅ Test results collection
- ✅ Real build/test status extraction
- ✅ Visual step summaries with pass/fail indicators
- 🔄 POWER LPAR network accessibility (in progress - adding to public network)
- ⏳ Awaiting allowlist addition to pytorch/pytorch
- ⏳ HUD integration pending allowlist approval

**Next Steps**: 
1. Complete POWER LPAR public network configuration (in progress)
2. Contact PyTorch maintainers to add `TorchPowerCI/pytorch-power-backend` to the L2 allowlist

**L2 Integration Checklist:**
- ✅ `id-token: write` permission configured
- ✅ `repository_dispatch` trigger configured
- ✅ `in_progress` callback implemented
- ✅ `completed` callback with test results
- ✅ Artifact URL reporting
- ✅ Build status extraction (pass/fail)
- ✅ Test status extraction (pass/fail)
- ✅ Smart conclusion logic
- ⏳ Network accessibility (planned)
- ⏳ Allowlist addition (pending)
