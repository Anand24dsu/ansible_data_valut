# Ansible Vault for YugabyteDB Data Import

This repository contains Ansible playbooks for securely importing data into YugabyteDB using Ansible Vault for credential management.

## Prerequisites

- Ansible (version 2.9 or later) installed on your control node
- SSH access to your target server(s)
- The `community.postgresql` collection installed:
  ```bash
  ansible-galaxy collection install community.postgresql
  ```

## Project Structure

```
project/
├── ansible.cfg
├── group_vars/
│   └── all/
│       ├── vars.yml       # Unencrypted variables
│       └── vault.yml      # Encrypted vault file with credentials
├── inventory.ini
├── northwind_ddl.sql      # SQL for creating tables
├── northwind_data.sql     # SQL for inserting data
└── yugabytedb_data.yml    # Main playbook
```

## Step-by-Step Setup Instructions

### 1. Create the Project Structure

Create the basic directory structure:

```bash
mkdir -p ansible_data_vault/group_vars/all
cd ansible_data_vault
```

### 2. Create the Vault File

Create an encrypted vault file to store sensitive credentials:

```bash
ansible-vault create group_vars/all/vault.yml
```

When prompted, enter a secure password that you'll remember.

Add the following credentials to the vault file:

```yaml
vault_db_user: yugabyte
vault_db_password: yugabyte
```

### 3. Create the Variables File

Create an unencrypted variables file that references the vault variables:

```bash
touch group_vars/all/vars.yml
```

Add the following content to `vars.yml`:

```yaml
# Reference to vault variables
db_user: "{{ vault_db_user }}"
db_password: "{{ vault_db_password }}"

# Other non-sensitive variables
db_name: testdb
db_host: "10.128.15.223"
db_port: 5433
create_db_if_not_exists: true
ddl_sql_local_path: ./northwind_ddl.sql
data_sql_local_path: ./northwind_data.sql
ddl_sql_remote_path: /tmp/northwind_ddl.sql
data_sql_remote_path: /tmp/northwind_data.sql
```

### 4. Create the Inventory File

Create an inventory file to define your target hosts:

```bash
touch inventory.ini
```

Add your servers to the inventory:

```ini
[all]
34.133.178.124 ansible_user=anand ansible_python_interpreter=/usr/bin/python
```

Replace `anand` with your actual SSH username.

### 5. Create the Ansible Configuration File

Create an ansible.cfg file:

```bash
touch ansible.cfg
```

Add the following configuration:

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
```

### 6. Create the Main Playbook

Create the main playbook file:

```bash
touch yugabytedb_data.yml
```

Add the following content to `yugabytedb_data.yml`:

```yaml
---
- name: Insert data into YugabyteDB using Ansible
  hosts: all
  gather_facts: false
  
  tasks:
    - name: Check if YugabyteDB port is reachable
      wait_for:
        host: "{{ db_host }}"
        port: "{{ db_port }}"
        timeout: 10
      register: port_check
      ignore_errors: true
    
    - name: Display connection status
      debug:
        msg: "YugabyteDB port status: {{ 'Reachable' if port_check.state is defined and port_check.state == 'started' else 'Unreachable' }}"
    
    - name: Ensure database exists
      community.postgresql.postgresql_db:
        name: "{{ db_name }}"
        login_user: "{{ db_user }}"
        login_password: "{{ db_password }}"
        login_host: "{{ db_host }}"
        login_port: "{{ db_port }}"
        state: present
      when: create_db_if_not_exists and (port_check.state is defined and port_check.state == 'started')
      register: db_creation
      ignore_errors: true
      no_log: true  # Hide sensitive output
    
    - name: Copy DDL SQL file to remote server
      copy:
        src: "{{ ddl_sql_local_path }}"
        dest: "{{ ddl_sql_remote_path }}"
        mode: '0644'
      when: port_check.state is defined and port_check.state == 'started'
    
    - name: Copy Data SQL file to remote server
      copy:
        src: "{{ data_sql_local_path }}"
        dest: "{{ data_sql_remote_path }}"
        mode: '0644'
      when: port_check.state is defined and port_check.state == 'started'
    
    - name: Execute DDL SQL (create tables)
      community.postgresql.postgresql_script:
        db: "{{ db_name }}"
        login_user: "{{ db_user }}"
        login_password: "{{ db_password }}"
        login_host: "{{ db_host }}"
        login_port: "{{ db_port }}"
        path: "{{ ddl_sql_remote_path }}"
      when: port_check.state is defined and port_check.state == 'started'
      register: ddl_query_result
      no_log: true  # Hide sensitive output
    
    - name: Execute Data SQL (insert rows)
      community.postgresql.postgresql_script:
        db: "{{ db_name }}"
        login_user: "{{ db_user }}"
        login_password: "{{ db_password }}"
        login_host: "{{ db_host }}"
        login_port: "{{ db_port }}"
        path: "{{ data_sql_remote_path }}"
      when: port_check.state is defined and port_check.state == 'started'
      register: data_query_result
      no_log: true  # Hide sensitive output
    
    - name: Query all required tables
      community.postgresql.postgresql_query:
        db: "{{ db_name }}"
        login_user: "{{ db_user }}"
        login_password: "{{ db_password }}"
        login_host: "{{ db_host }}"
        login_port: "{{ db_port }}"
        query: "SELECT * FROM public.{{ item }};"
      loop:
        - categories
        - employees
        - territories
        - orders
        - products
        - us_states
        - order_details
      register: query_results
      when: port_check.state is defined and port_check.state == 'started'
      ignore_errors: true
      no_log: true  # Hide sensitive output
    
    - name: Display query results
      debug:
        msg: "{{ item.query_result }}"
      loop: "{{ query_results.results }}"
      when: item.query_result is defined
```

### 7. Prepare Your SQL Files

Place your SQL files in the project directory:
- `northwind_ddl.sql` - Contains table creation statements
- `northwind_data.sql` - Contains data insertion statements

### 8. Running the Playbook

To run the playbook with vault password:

```bash
ansible-playbook yugabytedb_data.yml --ask-vault-pass
```

When prompted, enter the vault password you set in step 2.

### 9. Troubleshooting SSH Connection Issues

If you encounter SSH connection issues like:

```
TASK [Check if YugabyteDB port is reachable] ***********************************************************
fatal: [hostname]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ssh: connect to host X.X.X.X port 22: Connection timed out", "unreachable": true}
```

Try these troubleshooting steps:

1. **Verify direct SSH access:**
   ```bash
   ssh username@host_ip
   ```

2. **Check firewall settings:** Ensure port 22 is open on the server and any network firewalls.

3. **Update inventory with explicit SSH details:**
   ```ini
   [all]
  34.133.178.124 ansible_user=anand ansible_python_interpreter=/usr/bin/python

   ```

4. **Test basic connectivity:**
   ```bash
   ansible -i inventory.ini yugabytedb-node -m ping -vvv
   ```

### 10. Additional Vault Commands

- **Edit vault file:**
  ```bash
  ansible-vault edit group_vars/all/vault.yml
  ```

- **View vault file content:**
  ```bash
  ansible-vault view group_vars/all/vault.yml
  ```

- **Encrypt an existing file:**
  ```bash
  ansible-vault encrypt path/to/file.yml
  ```

- **Decrypt a vault file:**
  ```bash
  ansible-vault decrypt path/to/file.yml
  ```

- **Change vault password:**
  ```bash
  ansible-vault rekey group_vars/all/vault.yml
  ```

### 11. Using Vault Password File (Optional)

For automation purposes, you can store the vault password in a file:

```bash
echo "your_vault_password" > ~/.vault_pass.txt
chmod 600 ~/.vault_pass.txt
```

Update your `ansible.cfg`:
```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
vault_password_file = ~/.vault_pass.txt
```

Then run without `--ask-vault-pass`:
```bash
ansible-playbook yugabytedb_data.yml
```

## Security Best Practices

1. Never commit the vault password file to version control
2. Rotate vault passwords regularly
3. Use `no_log: true` for tasks containing sensitive data
4. Consider using different vaults for different environments
5. Limit access to vault passwords to authorized personnel only