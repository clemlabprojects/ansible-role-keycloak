# ansible-role-keycloak

Installs and configures Keycloak 25.0.6 as the SSO provider for an ODP Hadoop cluster. The role is a single play that covers installation through post-install federation setup.

---

## What This Role Does

Tasks execute in the following order:

### 1. Install

- Installs OpenJDK 17
- Downloads Keycloak 25.0.6 and extracts it to `/opt/keycloak/`
- Creates a `keycloak` system user
- Writes a systemd service unit and starts the service
- Copies SSL certificates to the configured cert folder

### 2. Realm Bootstrap

- Waits for the Keycloak API to become available
- Obtains an admin token from the master realm
- Creates the production realm
- Creates a service account user and grants it `realm-admin` role-mappings

### 3. OIDC Client

Creates or updates a confidential OIDC client in the production realm.

`directAccessGrantsEnabled: true` is set on this client. This is required so Ambari can register its own sub-clients via the Resource Owner Password Credentials flow during cluster Kerberos setup. Without it, Ambari's keytab provisioning will fail with a 401.

### 4. LDAP User Federation — Path A

**Active when:** `freeipa_servers` is defined, or `keycloak_ldap_enabled: true`

- Creates or updates a generic LDAP federation component (idempotent: POST if absent, PUT if present)
- Attaches a `group-ldap-mapper` sub-component that imports groups and user memberships from the LDAP server into Keycloak
- Triggers a full sync

All LDAP settings are fully parametrized via `keycloak_ldap_*` variables. Any LDAP-compatible directory works; FreeIPA is not required.

### 5. Manual Users and Groups — Path B

**Active when:** `freeipa_servers` is NOT defined AND `keycloak_ldap_enabled` is not true

- Creates local Keycloak users from the `cluster_users` list
- Creates groups from `cluster_groups` and from `cluster_users[*].groups`
- Assigns user-to-group memberships

**Path A and Path B are mutually exclusive.** Only one runs per deploy, selected automatically based on whether `freeipa_servers` is in scope.

### 6. Ambari JWT SSO

**File:** `tasks/ambari_sso.yaml`  
**Guard:** `ambari_sso_enabled | default(false)`

- Registers a confidential OIDC client for Ambari (authorization-code flow)
- Retrieves the client secret and realm RSA public key
- Configures Ambari JWT SSO and the OIDC callback via the Ambari REST API

The Ambari `OidcCallbackFilter` exchanges the authorization code server-side and issues an HttpOnly `hadoop-jwt` cookie.

### 7. Active Directory LDAP Federation

**File:** `tasks/ad_ldap.yaml`  
**Guard:** `ad_servers is defined`

- Federates AD users into the realm in `READ_ONLY` mode
- Uses `sAMAccountName` as the Keycloak username
- Triggers a full sync after creation

AD federation is additive and can run alongside FreeIPA federation. Both federations are idempotent (check-then-create).

---

## Variables

### Required

Set these in your `external_vars` file and vault.

| Variable | Description |
|---|---|
| `keycloak_sso_hostname` | FQDN of the Keycloak server (also used for SSL cert filenames) |
| `keycloak_server_port` | HTTPS port (typically `8443`) |
| `keycloak_admin_user` | Keycloak master realm admin username |
| `keycloak_admin_password` | Keycloak master realm admin password (vault) |
| `keycloak_realm` | Name of the production realm to create |
| `keycloak_service_user` | Service account username created in the realm |
| `keycloak_service_user_password` | Service account password (vault) |
| `jdk_home` | Path to JDK home (e.g. `/usr/lib/jvm/java-17-openjdk`) |
| `security_ssl_cert_folder` | Remote path where certs are placed |
| `source_cert_folder` | Local path where the role copies certs from |
| `keycloak_database_name` | PostgreSQL database name |
| `keycloak_database_hostnames` | List of DB hostnames (first entry is used) |
| `keycloak_database_user` | DB username |
| `keycloak_database_user_password` | DB password (vault) |

### Optional — General

| Variable | Default | Description |
|---|---|---|
| `keycloak_ssl_enabled` | — | Enable HTTPS (sets ssl_opts) |
| `keycloak_proxy_enabled` | — | Add `--proxy-headers forwarded` |
| `firewalld_enabled` | — | Open port 8443 in firewalld |
| `keycloak_validate_certs` | `true` | Validate TLS certs in API calls |
| `keycloak_admin_roles` | `['realm-admin']` | Roles granted to the service user |
| `keycloak_client_redirect_uris` | existing value | Redirect URIs for the OIDC client |
| `keycloak_client_web_origins` | existing value | Web origins for the OIDC client |

### Optional — LDAP Federation (Path A)

| Variable | Default | Description |
|---|---|---|
| `keycloak_ldap_enabled` | auto (true if `freeipa_servers` defined) | Force-enable or force-disable LDAP federation |
| `keycloak_ldap_federation_name` | `freeipa` | Name of the federation component in Keycloak |
| `keycloak_ldap_vendor` | `other` | LDAP vendor (`other`, `rhds`, `ad`) |
| `keycloak_ldap_url` | `ldap://<freeipa_servers[0]>` | LDAP connection URL |
| `keycloak_ldap_users_dn` | `ipaserver_domain_suffix_users` | Users base DN |
| `keycloak_ldap_bind_dn` | `uid=admin,cn=users,cn=accounts,<suffix>` | Bind DN |
| `keycloak_ldap_bind_credential` | `ipaadmin_password` | Bind password (vault) |
| `keycloak_ldap_username_attr` | `uid` | Username LDAP attribute |
| `keycloak_ldap_uuid_attr` | `ipaUniqueID` | UUID LDAP attribute |
| `keycloak_ldap_edit_mode` | `READ_ONLY` | Federation edit mode |
| `keycloak_ldap_sync_on_setup` | `true` | Trigger full sync after setup |
| `keycloak_ldap_group_mapper_enabled` | auto (same as LDAP enabled) | Enable the `group-ldap-mapper` sub-component |
| `keycloak_ldap_groups_dn` | `ipaserver_domain_suffix_groups` | Groups base DN |
| `keycloak_ldap_group_name_attr` | `cn` | Group name attribute |
| `keycloak_ldap_group_object_classes` | `groupOfNames` | Group object class |
| `keycloak_ldap_membership_attr` | `member` | Membership attribute |
| `keycloak_ldap_membership_type` | `DN` | Membership attribute type |

### Optional — Ambari SSO (`ambari_sso_enabled: true`)

| Variable | Default | Description |
|---|---|---|
| `ambari_server_host` | — | Ambari server hostname or LB FQDN |
| `ambari_server_port` | `8442` | Ambari HTTPS port |
| `ambari_admin_user` | — | Ambari admin username |
| `ambari_admin_password` | — | Ambari admin password (vault) |
| `keycloak_ambari_client_id` | `ambari` | OIDC client ID for Ambari |
| `ambari_sso_cookie_name` | `hadoop-jwt` | JWT cookie name |
| `ambari_sso_oidc_callback_url` | derived from host:port | Override for load-balanced deployments |

### Optional — AD LDAP Federation (`ad_servers` defined)

| Variable | Description |
|---|---|
| `ad_servers` | List of AD domain controller hostnames or IPs |
| `ad_bind_dn` | Bind DN for the read-only service account |
| `ad_bind_password` | Bind password (vault) |
| `ad_users_dn` | Base DN for user searches |
| `ad_use_ldaps` | Use LDAPS port 636 instead of 389 (default: `false`) |

---

## Example Playbook

### FreeIPA-backed cluster (typical ODP deployment)

```yaml
- name: Keycloak SSO
  hosts: keycloak_hosts
  become: true
  vars_files:
    - vars/external_vars_dev_arm.yml
    - vaults/vault-dev.yml
  roles:
    - role: ansible-role-keycloak
```

Key variables in `external_vars_dev_arm.yml`:

```yaml
keycloak_sso_hostname: sso.dev21.hadoop.clemlab.com
keycloak_server_port: 8443
keycloak_realm: odp
keycloak_service_user: keycloak-svc
keycloak_ssl_enabled: true
keycloak_proxy_enabled: false
firewalld_enabled: true
jdk_home: /usr/lib/jvm/java-17-openjdk

# LDAP federation auto-enabled because freeipa_servers is defined
freeipa_servers:
  - admin21.dev21.hadoop.clemlab.com
ipaserver_domain_suffix: dc=dev21,dc=hadoop,dc=clemlab,dc=com
ipaserver_domain_suffix_users: cn=users,cn=accounts,dc=dev21,dc=hadoop,dc=clemlab,dc=com
ipaserver_domain_suffix_groups: cn=groups,cn=accounts,dc=dev21,dc=hadoop,dc=clemlab,dc=com

# Ambari SSO
ambari_sso_enabled: true
ambari_server_host: master21.dev21.hadoop.clemlab.com
ambari_server_port: 8442
ambari_admin_user: admin
```

### Standalone deployment (no FreeIPA — Path B active)

```yaml
# external_vars snippet
keycloak_ldap_enabled: false   # disables Path A, activates Path B
cluster_users:
  - givenname: John
    sn: Doe
    password: "{{ vault_john_password }}"
    groups: [hadoop-users, admins]
cluster_groups:
  - hadoop-users
  - admins
```

### Active Directory federation added to an existing cluster

```yaml
# external_vars snippet — add alongside FreeIPA vars
ad_servers:
  - dc01.corp.example.com
ad_bind_dn: "CN=svc-keycloak,OU=ServiceAccounts,DC=corp,DC=example,DC=com"
ad_bind_password: "{{ vault_ad_svc_password }}"
ad_users_dn: "DC=corp,DC=example,DC=com"
ad_use_ldaps: true
```

---

## License

Proprietary — ODP / DPH-BUILDER internal use.
