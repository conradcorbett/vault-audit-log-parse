
# Vault Audit Log Parsing Examples

## Setup

Before we start parsing Vault's audit log, make sure the following setup steps are completed.

### 1. **Enable Vault's Audit Log:**
To enable the audit log, run the following command:
```bash
vault audit enable file file_path=/tmp/my-file.txt
```
This will create an audit log file at `/tmp/my-file.txt`.

### 2. **Enable a Secret Engine and Create Secrets:**
Enable the `kv-v2` secret engine and create a couple of secrets:
```bash
vault secrets enable -path=app1 kv-v2
vault kv put /app1/apikeys api-key1=value1 api-key2=value2
vault kv put /app1/database-password username=app1 password=dbpass
```

### 3. **Create a User with a Policy:**
Create a policy that allows the user to read the secrets and associate the policy with a user:
```bash
vault policy write tester1 - <<EOF
path "app1/data/apikeys" {
  capabilities = ["read"]
}
EOF
```

Now enable user authentication and create a user `tester1`:
```bash
vault auth enable userpass
vault write auth/userpass/users/tester1     password=changeme     policies=tester1
```

### 4. **Generate Failed Logins:**
Generate some failed login attempts to simulate errors:
```bash
vault login -method=userpass username=tester password=wrongpassword
```

---

## Querying Audit Log for Specific Entries

### **Query for Invalid Credentials:**
To check for failed login attempts (invalid credentials), use the following command to filter the log:
```bash
cat my-file.txt | jq -r 'select(.error == "invalid credentials") | {time: .time, path: .request.path, remote_address: .request.remote_address, error: .error}'
```

#### Example Output:
```json
{
  "time": "2025-03-31T23:28:59.099067555Z",
  "path": "auth/userpass/login/tester",
  "remote_address": "127.0.0.1",
  "error": "invalid credentials"
}
```

This output shows the log entries where an invalid login attempt occurred, including the time, path, and remote address.

---

### **Example of User Display Name from Audit Log:**
To view the display name and other details for successful logins:
```bash
cat my-file.txt | jq -r 'select(.type == "response") | {display_name: .auth.display_name, remote_address: .request.remote_address, time: .time}'
```

#### Example Output:
```json
{
  "display_name": "userpass-tester",
  "remote_address": "127.0.0.1",
  "time": "2025-03-31T22:26:52.752195385Z"
}
{
  "display_name": "userpass-tester",
  "remote_address": "127.0.0.1",
  "time": "2025-03-31T22:37:19.911696134Z"
}
```

This shows the display name of the user along with the time and remote address of the login event.

---

### **Login and Read Secret Requests:**

1. **Login as `tester1`:**
```bash
vault login -method=userpass username=tester1 password=changeme
```

2. **Read Secrets:**
```bash
vault kv get /app1/apikeys
vault kv get /app1/database-password
```

---

### **Generate Read Requests Against Specific Secret Path:**
To filter for read operations against a specific secret path (`audittest/data/secret1`):
```bash
cat my-file.txt | jq -r 'select(.type == "response" and .request.operation == "read" and .request.path == "audittest/data/secret1") | {display_name: .auth.display_name, remote_address: .request.remote_address, time: .time}'
```

This will output details for all successful read requests on the `audittest/data/secret1` path.

---

### **View Entries Where User Did Not Have Access:**
To view entries where a user attempted to access a path they did not have permission for:
```bash
cat my-file.txt | jq -r 'select(.auth.policy_results.allowed == false and .type == "response") | {time: .time, remote_address: .request.remote_address, path: .request.path, display_name: .auth.display_name}'
```

This filters the log for entries where a user was denied access based on the policy configuration. The output will show the time, remote address, requested path, and the user's display name.

---

## Summary of Commands

- **Enable audit log**: `vault audit enable file file_path=/tmp/my-file.txt`
- **Enable secret engine & create secrets**: 
  - `vault secrets enable -path=app1 kv-v2`
  - `vault kv put /app1/apikeys api-key1=value1 api-key2=value2`
- **Create user with policy**: `vault policy write tester1 - <<EOF`
- **Generate failed login attempts**: `vault login -method=userpass username=tester password=wrongpassword`
- **Query audit log**: Use `jq` commands to filter based on various criteria like error messages or specific paths.

This guide should help you query Vault's audit logs effectively to monitor authentication events, errors, and access requests.
