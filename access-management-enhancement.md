# Users Access Management Enhancement

## 1. Initiatives

Our RDS shards sit on public IPs, authenticated by static passwords in `.env` and YAML, with no
record of who connects or what they run. This initiative puts every shard behind one
identity-aware access plane on self-hosted Teleport Community and moves RDS fully private. The
goals: close the public exposure, replace shared passwords with per-person short-lived
certificates, log every session and query, scope each person to only the client shard they were
granted, and serve internal staff, external client users, and our Heroku apps through the same
plane. It stays cheap (free Community license, no VPN). Per-client shards, version-controlled
roles, and dry-run apply are unchanged.

---

## 2. Current state architecture

```mermaid
flowchart TD
    TL["Tech lead<br/>runs role_manager.py manually"] -->|"psycopg2, static creds in .env"| PUB
    IU["Internal staff"] -->|direct connect| PUB
    EU["External client users"] -->|direct connect over internet| PUB
    APP["Heroku apps (v3_settings, scrapers)<br/>outside the AWS VPC"] -->|"static creds in .env"| PUB

    subgraph PUB ["RDS on PUBLIC IPs, open to the internet"]
        DB[("customore-v2 main shards<br/>+ per-client DW shards")]
    end

    GAPS["No SSO, no MFA, no audit.<br/>Standing accounts, static passwords in .env and YAML."]
    PUB -.-> GAPS
```

Everything authenticates with a long-lived password that lives in plaintext config, anyone with
the string and the public endpoint is in, and nothing is recorded.

---

## 3. Target architecture

One plane. Teleport (auth + proxy) is the only way in. RDS has no public IP. The DB agent runs
inside the VPC and is the only thing that talks to the shards directly.

```mermaid
flowchart TB
    Manager["Manager (ops/platform)<br/>assigns roles via admin console / tctl"]
    User["Staff & external users<br/>local Teleport account + MFA"]
    Heroku["Heroku apps<br/>tbot (Machine ID) on the dyno"]

    subgraph plane["Teleport Community — the access plane"]
        TP["Auth + Proxy (public DNS)<br/>users, roles, audit log"]
        AG["DB Agent (in VPC)<br/>provisions the DB user per session"]
    end

    subgraph VPC ["Private VPC — RDS has no public IP"]
        RDS[("Per-client RDS shards")]
        SVC["In-VPC services (if any)"] -->|"IAM DB auth, direct"| RDS
    end

    Manager -->|"assign the database role"| TP
    User -->|"Teleport Connect, then DBeaver"| TP
    Heroku -->|"tbot local DB tunnel, join token from Doppler"| TP
    TP --> AG
    AG -->|"connect as the granted role"| RDS
    TP --> AUDIT["Audit log<br/>who connected, every query"]
    RM["role_manager<br/>group roles = the databases"] -.->|defines| RDS
```

Key points:

- **The Teleport proxy stays public on purpose.** It is the secure front door. Everything behind
  it goes private. RDS security groups allow only the DB agent's security group.
- **Humans** log in through the Teleport Connect desktop app and query from DBeaver over a local
  proxy. They never hold a DB password; the certificate is their identity.
- **Heroku apps** run `tbot` on the dyno, which opens a local Postgres tunnel. The app's
  `DATABASE_URL` points at `localhost`, no app code change beyond the connection string. The join
  token is delivered through Doppler (already in our stack) and rotates, so no long-lived secret
  anywhere.
- **The agent provisions the Postgres user on the fly** as `teleport_admin`, grants it the group
  roles the person's Teleport role names, then connects as that user.

---

## 4. How each actor connects

| Actor | Path | Auth | Scope |
|---|---|---|---|
| **Staff** | Teleport Connect app → DBeaver on a local proxy | Local Teleport account + MFA, short-lived cert | The shards the manager granted |
| **External client user** | same | Local Teleport account + MFA | The one client shard granted |
| **Heroku app (machine)** | `tbot` local DB tunnel → public proxy → in-VPC agent | Machine ID cert, join token via Doppler | The shards its bot identity was granted |
| **Teleport DB agent** | Inside the VPC | mTLS / IAM as per-shard `teleport_admin` | Provisions the session user with the granted role |
| **In-VPC service** (if any) | Direct, no Teleport | IAM DB auth via its task role | Its own database |


### End-to-end diagram:

```mermaid
sequenceDiagram
    autonumber
    participant U as You (DBeaver + Teleport Connect)
    participant TP as Teleport proxy (public DNS)
    participant AG as DB Agent (private VPC)
    participant PG as Client shard (private, no public IP)

    Note over U,TP: Once a day
    U->>TP: Log in (username, password, MFA)
    TP-->>U: Short-lived certificate (identity, role, traits)

    Note over U,PG: Every connect
    U->>TP: Open the client shard
    TP->>TP: Authorize (role: client label, db user, db name)
    U->>U: Teleport Connect opens a loopback proxy for DBeaver
    U->>TP: DBeaver session (mTLS, no DB password)
    TP->>AG: Route into the VPC
    AG->>PG: As teleport_admin, ensure your user exists, GRANT the role's db_roles
    AG->>PG: Connect as you (now holding the group roles)
    U->>PG: SELECT ... through the proxy
    PG-->>U: Rows (only what the group roles allow)
    TP->>TP: Audit log records the session and every query
```

---

## 5. User and role management

![alt text](image.png)
**Lifecycle (manual, tied to the joiner/leaver process):**

| Step | Action |
|---|---|
| Onboard | Ops creates the account: `tctl users add` (staff and external alike), MFA enrolled on first login. Baseline = no DB access. |
| Revoke | Manager removes the role. To cut an active session instantly: `tctl lock`. |
| Offboard | `tctl lock` (instant "access denied") then `tctl users rm`. |
| Review | Periodic access reviews, since there is no IdP to sync from. |

**How a grant becomes table access (two independent axes):**

```mermaid
flowchart LR
    subgraph A["Axis 1 — WHICH database"]
        direction TB
        a1["trait: allowed_clients = nestle-ph"] --> a2["analyst role, db_labels.client = trait"]
        a2 --> a3["matches shard label client = nestle-ph"]
        a3 --> a4["reachable shard: dw-nestle-ph"]
    end
    subgraph B["Axis 2 — WHAT you can do"]
        direction TB
        b1["analyst role, db_roles"] --> b2["Postgres group role, analytic_read_role_table_level"]
        b2 --> b3["SELECT on ALL tables in public"]
    end
    a4 --> R["Your session: read-only on dw-nestle-ph"]
    b3 --> R
```

Traits and labels pick *which* database, `db_roles` pick *what* you can do. Add a client by
adding it to the person's `allowed_clients` trait; make them read-write by adding a write group
role. The two never tangle, so **one templated `analyst` role covers everyone**. Onboarding a new
client adds zero Teleport roles.

**`role_manager` becomes group-roles-only.** It keeps `CREATE ROLE` and the privilege grants for
the group roles (`analytic_*`, `product_*`, `tech_*`, `read_dw_*`) plus the per-shard
`teleport_admin` the agent uses. It drops `CREATE USER`, per-user grants, and passwords. Two
tiers: a coarse analytics tier (`databases: [ALL]`, all tables in `public`) that is shard- and
table-count independent, and a fine per-client tier for subset access. `ALTER DEFAULT PRIVILEGES`
keeps new tables readable without editing any role.

```mermaid
flowchart LR
    NT["New table in public"] -->|"auto via ALTER DEFAULT PRIVILEGES"| GR["Group role"]
    GR -->|"you are already a member"| SU["Your session user"]
    SU --> V["Queryable on next connect, no grant edited"]
```

Note: `tables: [ALL]` makes `public` the trust boundary. Hold sensitive tables in a separate
schema or in `except_tables`.

---

## 6. Next steps

Ordered so nothing breaks during the cutover (register and test through Teleport while RDS is
still public, then flip it private last).

1. **Inventory** every shard, group role, and current user.
2. **Stand up the Teleport cluster** (single node to start, HA later). Create a local account per
   person, no DB access yet, turn on Teleport MFA.
3. **Run the DB Agent in the VPC**, register every shard with `teleport_admin` and
   auto-provisioning, **while RDS is still public**.
4. **Convert `role_manager`** to group-roles-only, add `teleport_admin`, apply.
5. **Cut humans over.** Grant one person a shard end to end, connect via Teleport Connect, confirm
   the audit log. Then roll across staff and external users.
6. **Get one Heroku app on Machine ID.** `tbot` on the dyno, join token via Doppler, app
   `DATABASE_URL` → local tunnel. Verify against the still-public shard.
7. **Flip private.** Set `PubliclyAccessible = false`, move to private subnets, lock security
   groups to the agent's SG only.
8. **Remove the old standing accounts** and strip `.env`/YAML passwords. Keep a restricted
   break-glass path straight to RDS over IAM for emergencies.


### Cost

Teleport Community license is free. Recurring: the node(s) running auth and proxy, the audit and
session backend (S3 + DynamoDB), a NAT gateway, and a public DNS name with a TLS certificate. No
VPN, no per-user access license.
