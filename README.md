
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
vault auth enable -path=userpass1 userpass
vault write auth/userpass1/users/tester1     password=changeme     policies=tester1
```

### 4. **Generate Failed Logins:**
Generate some failed login attempts to simulate errors:
```bash
vault login -method=userpass -path=userpass1 username=tester1 password=wrongpassword
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
  "path": "auth/userpass/login/tester1",
  "remote_address": "127.0.0.1",
  "error": "invalid credentials"
}
```

This output shows the log entries where an invalid login attempt occurred, including the time, path, and remote address.

---

### **Login as User and Read Secrets:**

1. **Login as `tester1`:**
```bash
vault login -method=userpass -path=userpass1 username=tester1 password=changeme
```

2. **Read some Secrets:**
```bash
vault kv get /app1/apikeys
vault kv get /app1/database-password
```

---

### **View Read Requests Against Specific Secret Path:**
To filter for read operations against a specific secret path (`app1/data/apikeys`):
```bash
cat my-file.txt | jq -r 'select(.type == "response" and .request.operation == "read" and .request.path == "app1/data/apikeys") | {display_name: .auth.display_name, remote_address: .request.remote_address, time: .time}'
```

This will output details for all successful read requests on the `app1/data/apikeys` path.

---

### **View Entries Where User Attempted To Access Path They Did Not Have Access:**
```bash
cat my-file.txt | jq -r 'select(.auth.policy_results.allowed == false and .type == "response") | {time: .time, remote_address: .request.remote_address, path: .request.path, display_name: .auth.display_name}'
```

This filters the log for entries where a user was denied access based on the policy configuration. The output will show the time, remote address, requested path, and the user's display name.

## Cleanup

Delete the lab setup

### 1. **Disable Vault's Audit Log:**
To disable the audit log, run the following command:
```bash
vault audit disable file
rm /tmp/my-file.txt
```

### 2. **Disable secrets, delete user, and auth mounts:**
Enable the `kv-v2` secret engine and create a couple of secrets:
```bash
vault secrets disable app1
vault auth disable userpass1
vault policy delete tester1
```