# Certificate checker

Ansible playbook that reads the TLS certificate presented by configured
services, calculates its validity period, and writes a semicolon-separated
report to `/tmp/result_cert.csv` on the Ansible controller.

## Requirements

- Ansible, OpenSSL, and `date` on the controller
- For `ssh` checks, SSH access and OpenSSL on the managed hosts

## Inventory

Define `cert_check_method` and a `cert_check_ports` list on each certificate
check group. Every checked host should belong to exactly one such group. If a
host needs a distinct set of ports or a different method, place it in a
dedicated group with that complete configuration.

See [`inventories/example.yml`](inventories/example.yml) for an anonymized
inventory showing direct TLS checks and both SSH target choices. Copy and adapt
it for your environment; the working inventory is intentionally ignored by
Git.

### SSH credentials with Ansible Vault

Keep SSH usernames and passwords in one encrypted Vault file. This lets
different hosts use different accounts and credentials. Create its directory
and file:

```bash
mkdir -p inventories/group_vars/all
ansible-vault create inventories/group_vars/all/vault.yml
```

In the editor, add `ansible_users` and `ansible_passwords` mappings like those in
[`inventories/ssh-credentials.example.yml`](inventories/ssh-credentials.example.yml),
replacing the example names with exact SSH inventory hostnames and the values
with the account and password for each host. The SSH groups map both connection
variables by `inventory_hostname`. Add one entry to each mapping for every new
SSH host.

The Vault file is encrypted and ignored by Git.

Run the playbook and enter the Vault password when prompted:

```bash
ansible-playbook -i inventories/home-lab.yml check_cert.yml --ask-vault-pass
```

Password-based SSH authentication requires `sshpass` installed on the
controller. Ansible prompts for the Vault password once per run. Do not put
the Vault password itself in the inventory or commit it.

`openssl` connects from the controller to the inventory host and port.
`ssh` connects to the managed host and runs OpenSSL there. By default,
`cert_check_server_localhost: true` checks `localhost:<port>`. Set it to
`false` to connect from the managed host to `cert_check_server_name:<port>`;
that name is also sent as SNI. The false setting requires
`cert_check_server_name`.

For direct `openssl` checks, `cert_check_host` optionally overrides the
inventory hostname used for the connection. `cert_check_server_name` optionally
overrides the SNI name; it defaults to the inventory hostname.

Ports are checked sequentially for each host. Hosts without a valid method or a
non-empty `cert_check_ports` list fail with a configuration error.

## Run

From the project directory:

```bash
ansible-playbook -i inventories/example.yml check_cert.yml
```

The report contains the host, port, certificate validity dates, remaining days,
subject, and serial number. The report is recreated for each run.
