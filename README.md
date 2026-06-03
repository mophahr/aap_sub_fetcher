# AAP Subscription Fetcher

Ansible playbooks to manage Red Hat Ansible Automation Platform subscription manifests.

## Playbooks

| Playbook | Description |
|----------|-------------|
| `list_available_pools.yml` | List available subscription pools to find pool IDs (requires portal credentials) |
| `create_aap_manifest.yml` | Create manifest via Portal API (requires portal credentials) |
| `upload_existing_manifest.yml` | Upload pre-downloaded manifest to Automation Gateway |

## Requirements

- Ansible 2.9+
- `redhat.satellite` collection (for `create_aap_manifest.yml` and `list_available_pools.yml`)
- `ansible.controller` collection (for `upload_existing_manifest.yml`)

### Configure Automation Hub token

1. Copy `ansible.cfg.example` to `ansible.cfg`
2. Log in to [console.redhat.com](https://console.redhat.com/ansible/automation-hub/token)
3. Generate or copy your API token
4. Edit `ansible.cfg` and replace `<your-automation-hub-token>` with your token

Alternatively, set the token via environment variable:

```bash
export ANSIBLE_GALAXY_SERVER_CERTIFIED_HUB_TOKEN="your-token-here"
```

### Install the collections

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

## Usage

### Create manifest via API (requires portal username/password)

```bash
ansible-playbook create_aap_manifest.yml -e @vars/credentials.yml
```

You will be prompted for:
- Red Hat Customer Portal username
- Red Hat Customer Portal password
- AAP subscription pool ID

#### Idempotent behavior

The playbook is **idempotent** - running it multiple times will not create duplicate manifests:

- If a manifest with the same `manifest_name` exists, it will be updated (not recreated)
- Subscription quantities will be adjusted to match the desired value
- The manifest UUID remains the same across runs

#### Using an existing manifest from the portal

If you already created a manifest manually at [console.redhat.com/subscriptions/manifests](https://console.redhat.com/subscriptions/manifests), you can manage it with this playbook by setting `manifest_name` to match the existing manifest name:

```bash
ansible-playbook create_aap_manifest.yml -e manifest_name="Your Existing Manifest Name" -e @vars/credentials.yml
```

Alternatively, you can use the manifest UUID directly by adding it to your `vars/credentials.yml`:

```yaml
manifest_uuid: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

### Upload existing manifest

1. Run `create_aap_manifest.yml` as documented above or download your manually created manifest from [console.redhat.com/subscriptions/manifests](https://console.redhat.com/subscriptions/manifests) and place it as `aap_sub_fetcher-test.zip` in this directory
3. Copy `vars/aap_credentials.yml.example` to `vars/aap_credentials.yml` and fill in your AAP details
4. Run:

```bash
ansible-playbook upload_existing_manifest.yml -e @vars/aap_credentials.yml
```

### Example: Using a variables file

Create a `vars/credentials.yml` file (see `vars/credentials.yml.example`), then run:

```bash
ansible-playbook create_aap_manifest.yml -e @vars/credentials.yml
```

### Example: Using Ansible Vault for secrets

```bash
ansible-vault create vars/credentials.yml
ansible-playbook create_aap_manifest.yml -e @vars/credentials.yml --ask-vault-pass
```

## Finding your Pool ID

Use the list_available_pools playbook (requires portal credentials):

```bash
ansible-playbook list_available_pools.yml -e @vars/credentials.yml
```

This queries the Red Hat Subscription API and displays all available AAP pools. By default, pools with 0 availability are hidden. To show all pools:

```bash
ansible-playbook list_available_pools.yml -e @vars/credentials.yml -e hide_empty_pools=false
```

## Output

The manifest will be downloaded to `./aap_sub_fetcher-test.zip` by default.

## Variables

### Portal Credentials (vars/credentials.yml)

| Variable | Description |
|----------|-------------|
| `rhsm_username` | Red Hat Customer Portal username |
| `rhsm_password` | Red Hat Customer Portal password |
| `aap_pool_id` | Subscription pool ID for AAP |

### AAP Credentials (vars/aap_credentials.yml)

| Variable | Description |
|----------|-------------|
| `aap_host` | AAP hostname |
| `aap_username` | AAP username |
| `aap_password` | AAP password |

### create_aap_manifest.yml

| Variable | Default | Description |
|----------|---------|-------------|
| `manifest_name` | `aap_sub_fetcher-test` | Name of the manifest in Red Hat portal |
| `manifest_uuid` | (empty) | UUID of existing manifest (optional, overrides name) |
| `manifest_path` | `./aap_sub_fetcher-test.zip` | Local path to save the manifest |
| `subscription_quantity` | `10` | Number of subscriptions to attach |
| `content_access_mode` | `org_environment` | Content access mode (use `entitlement` for legacy) |

### list_available_pools.yml

| Variable | Default | Description |
|----------|---------|-------------|
| `pool_filter` | `Ansible Automation Platform` | Filter pools by product name |
| `hide_empty_pools` | `true` | Hide pools with 0 availability |
| `validate_certs` | `false` | SSL certificate validation |

## Known Limitations

### Satellite version is hardcoded

The `redhat.satellite.redhat_manifest` module has the Satellite/distributor version **hardcoded to `sat-6.3`** in its source code. This cannot be changed via playbook variables. While it does work with AAP, it doesn't comply with the documentation at https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/containerized_installation/assembly-gateway-licensing#proc-create-subscription-allocation_obtain-manifest

If you need a specific Satellite version (e.g., 6.16), you have two options:

1. **Create the manifest manually** at [console.redhat.com/subscriptions/manifests](https://console.redhat.com/subscriptions/manifests) where you can select the correct Satellite version, then use `upload_existing_manifest.yml` to upload it.

2. **Patch the module locally** by editing `collections/ansible_collections/redhat/satellite/plugins/modules/redhat_manifest.py` and changing:
   ```python
   'facts': {'distributor_version': 'sat-6.16',
             'system.certificate_version': '3.2'}
   ```
