# Metabase + OpenLDAP (Dev Stack)

Spin up a tiny lab to test **LDAP sign-in + just-in-time (JIT) user provisioning** in Metabase.

This stack includes:

- **Metabase Enterprise** (for LDAP auth)
- **Postgres** (Metabase application DB)
- **OpenLDAP** (Bitnami)
- **phpLDAPadmin** (web UI for LDAP)

> Metabase creates a user **the first time they successfully sign in via LDAP** (no background import).  
> _This README intentionally omits any anonymization/deletion flows._

---

## ⚙️ Services & Ports

| Service        | Purpose                | URL / Port                          |
|----------------|------------------------|-------------------------------------|
| Postgres       | Metabase app DB        | `localhost:5433` → container `5432` |
| Metabase (EE)  | Web UI                 | `http://localhost:3001`             |
| OpenLDAP       | Directory              | `localhost:1389` → container `389`  |
| phpLDAPadmin   | LDAP UI (HTTPS)        | `https://localhost:8081`            |

> **Apple Silicon (M1/M2):** this compose is ARM-friendly. If you ever see a platform warning, add `platform: linux/arm64/v8` to the affected service(s).

---

## 🐳 Quickstart

Start everything:

```bash
docker compose up -d
```

- Open **Metabase** → `http://localhost:3001` and complete the setup wizard with a **local admin** (email/password).
- Open **phpLDAPadmin** → `https://localhost:8081` (accept the self‑signed cert)  
  Login DN: `cn=admin,dc=example,dc=org` • Password: `adminpass`

---

## 👤 Create LDAP Users

You can use **phpLDAPadmin** (UI) or **CLI** inside the LDAP container.

### Option A — UI (phpLDAPadmin)

1. Create `ou=users,dc=example,dc=org`  
2. Add users (objectClasses: `inetOrgPerson`, `organizationalPerson`, `person`, `top`)

Example entries:

- **John Doe**  
  DN: `uid=jdoe,ou=users,dc=example,dc=org`  
  `uid=jdoe`, `givenName=John`, `sn=Doe`, `cn=John Doe`  
  `mail=jdoe@example.org`, `userPassword=secret123`

- **Jane Doe**  
  DN: `uid=jane,ou=users,dc=example,dc=org`  
  `uid=jane`, `givenName=Jane`, `sn=Doe`, `cn=Jane Doe`  
  `mail=jane@example.org`, `userPassword=secret123`

- **Alice Smith**  
  DN: `uid=alice,ou=users,dc=example,dc=org`  
  `mail=alice@example.org`, etc.

### Option B — CLI (fast)

```bash
docker compose exec -T ldap bash -lc "/opt/bitnami/openldap/bin/ldapadd -x -H ldap://localhost:389 \
  -D 'cn=admin,dc=example,dc=org' -w adminpass << 'LDIF'
dn: ou=users,dc=example,dc=org
objectClass: organizationalUnit
ou: users

dn: uid=jdoe,ou=users,dc=example,dc=org
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
objectClass: top
uid: jdoe
cn: John Doe
sn: Doe
givenName: John
mail: jdoe@example.org
userPassword: secret123

dn: uid=jane,ou=users,dc=example,dc=org
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
objectClass: top
uid: jane
cn: Jane Doe
sn: Doe
givenName: Jane
mail: jane@example.org
userPassword: secret123

dn: uid=alice,ou=users,dc=example,dc=org
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
objectClass: top
uid: alice
cn: Alice Smith
sn: Smith
givenName: Alice
mail: alice@example.org
userPassword: secret123
LDIF"
```

Verify the users exist:

```bash
docker compose exec ldap bash -lc "/opt/bitnami/openldap/bin/ldapsearch -x -H ldap://localhost:389 -b 'ou=users,dc=example,dc=org' '(objectClass=inetOrgPerson)' uid mail"
```

---

## 🔐 Configure LDAP in Metabase

Metabase → **Admin** → **Settings** → **Authentication** → **LDAP**

- **User provisioning**: **Enabled**
- **LDAP host**: `ldap`   (Docker service name)
- **LDAP port**: `389`
- **LDAP security**: `None` (for local dev)
- **Username or DN**: `cn=admin,dc=example,dc=org`
- **Password**: `adminpass`

**User schema**

- **User search base**: `ou=users,dc=example,dc=org`
- **User filter**:
  ```
  (&(objectClass=inetOrgPerson)(|(uid={login})(mail={login})))
  ```
- **Email attribute**: `mail`
- **First name attribute**: `givenName`
- **Last name attribute**: `sn`

Click **Test** → **Save**.

---

## 🚪 Provisioning (JIT) Flow

1. Sign out of Metabase.
2. Click **Sign in with LDAP**.
3. Log in as any user you created, e.g.:
   - `jdoe` / `secret123`
   - or `jdoe@example.org` / `secret123`

✅ On first successful bind, Metabase **auto‑creates** the account.  
Check **Admin → People** to see the new users after their first login.

---

## 👥 Optional: Group Sync (map LDAP groups → Metabase groups)

1) Create groups via LDAP (e.g., `groupOfNames`):

```bash
docker compose exec -T ldap bash -lc "/opt/bitnami/openldap/bin/ldapadd -x -H ldap://localhost:389 \
  -D 'cn=admin,dc=example,dc=org' -w adminpass << 'LDIF'
dn: ou=groups,dc=example,dc=org
objectClass: organizationalUnit
ou: groups

dn: cn=metabase-users,ou=groups,dc=example,dc=org
objectClass: groupOfNames
cn: metabase-users
member: uid=jdoe,ou=users,dc=example,dc=org
member: uid=jane,ou=users,dc=example,dc=org
LDIF"
```

2) In Metabase LDAP settings:
- **Synchronize Group Memberships**: **On**
- **Group search base**: `ou=groups,dc=example,dc=org`
- **Group membership filter**: `(member={dn})`
- Add a mapping: **metabase-users** → a Metabase group (e.g., _All Users_ or a custom group).

> Group membership is applied at **login time** (no background polling).

---

## 🧪 Useful Admin Checks

**List users via API (requires Metabase admin session):**

```bash
# login (replace admin email + password you set on first-run)
export MB_URL="http://localhost:3001"
export MB_USER="[email protected]"
export MB_PASS="your-admin-password"

export MB_SESSION=$(curl -s -X POST "$MB_URL/api/session" \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"$MB_USER\",\"password\":\"$MB_PASS\"}" | jq -r .id)

curl -s "$MB_URL/api/user" -H "X-Metabase-Session: $MB_SESSION" | jq '.[] | {id,email,first_name,is_active}'
```

**Connect to the app DB (from host):**

```bash
psql "postgresql://metabase:metabase@localhost:5433/metabase"
-- or inside container:
docker compose exec db psql -U metabase -d metabase

-- then:
SELECT id, email, first_name, last_name, is_active FROM core_user ORDER BY id DESC LIMIT 20;
```

---

## 🧰 Troubleshooting

- **phpLDAPadmin “Bad Request / speaking HTTP to SSL”**  
  Use **https://localhost:8081** (we map host 8081 → container 443).

- **phpLDAPadmin “Can’t contact LDAP server”**  
  - Ensure it points to service `ldap`, **port 389**, **tls: false** (we set via `PHPLDAPADMIN_LDAP_HOSTS`).
  - Confirm Docker DNS from inside the container:  
    `docker compose exec phpldapadmin getent hosts ldap`
  - Test LDAP from LDAP container:  
    `docker compose exec ldap bash -lc "/opt/bitnami/openldap/bin/ldapsearch -x -H ldap://localhost:389 -b 'dc=example,dc=org' -s base '(objectClass=*)'"`

- **Metabase “Test” fails in LDAP settings**  
  Double-check host `ldap`, port `389`, bind DN/password, and that your user filter matches real entries with `mail` set.

- **Apple Silicon: platform mismatch**  
  Add `platform: linux/arm64/v8` to the service(s) or force `linux/amd64` (slower).

---

## 🧹 Cleanup

```bash
docker compose down -v
```

---

## 🧭 Variants

- **OSS path (no LDAP)**: use Metabase OSS + **Google Sign‑In** to test SSO without Enterprise features.
- **StartTLS/LDAPS**: once you’re done with the basics, switch LDAP security and wire certs as needed.
