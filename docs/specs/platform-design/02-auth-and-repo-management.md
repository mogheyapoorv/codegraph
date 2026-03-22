# CodeGraph Platform — Authentication & Repo Management

## 3. Authentication & Admin Bootstrap

### Auth Modes

| Mode | When | How |
|---|---|---|
| **Built-in email/password** | Small teams, quick setup, air-gapped | bcrypt in PostgreSQL, CodeGraph issues JWTs |
| **Keycloak (bundled)** | Self-hosted orgs wanting SSO | Ships in docker-compose, pre-configured OIDC |
| **External OIDC** | Orgs with existing Okta/Auth0/Azure AD | Point at any OIDC discovery URL |
| **SAML (via Keycloak)** | Enterprise IdP requirement | Keycloak bridges SAML to OIDC |

Our auth middleware validates JWTs regardless of issuer. Same middleware, different token source.

### First-Time Admin Bootstrap

1. First boot detects empty user table
2. Setup wizard at `http://localhost:3000/setup`
3. Wizard asks:
   - Admin email + password (break-glass admin, always built-in)
   - Auth mode: "Email/password" or "Configure OIDC"
   - First repo credential (GitHub App / PAT)
   - First repo to index (optional — gets you to value fast)
4. Admin logs in, lands on Admin Dashboard

**Automated deployment override (Helm/Terraform/Ansible):**

```
CODEGRAPH_ADMIN_EMAIL=admin@company.com
CODEGRAPH_ADMIN_PASSWORD=<secure-password>
CODEGRAPH_AUTH_MODE=oidc
CODEGRAPH_OIDC_ISSUER_URL=https://keycloak.company.com/realms/codegraph
```

If these are set, first boot skips the wizard and auto-configures.

### Roles

| Role | Can Do |
|---|---|
| **Superadmin** | Everything. Created at bootstrap. Break-glass account. |
| **Admin** | Manage repos, groups, credentials, LLM config, view analytics. |
| **Member** | Browse wikis, chat, view diagrams, view API catalog. |

### LLM Configuration (Admin Only)

Admins configure this in the dashboard:

| Setting | Options |
|---|---|
| Mode | Direct (provider APIs) or Proxy (LiteLLM/OpenAI-compatible) |
| Providers | Anthropic, OpenAI, Google, Ollama, Bedrock, Azure |
| Operation routing | Different model per operation (chat, wiki gen, research, embedding, etc.) |
| Cost cap | Daily/monthly LLM spend cap. Pauses LLM operations when hit. |

Users never see any of this. Admins control cost, quality, and provider choices centrally.

### User Connected Accounts

For external doc access (Confluence/Google Docs) when a user pastes a link in chat:

- User connects their Google/Atlassian account via OAuth (Profile > Connected Accounts)
- We use their token ONLY when they paste a link. Never proactively.
- Token stored encrypted per-user. Doc content is never stored — fetched live, used for that single response.
- User hasn't connected? Graceful degradation. We answer with code and wiki only.

---

## 4. Repo Management & Repo Groups

### Private Repo Access

| Method | Platform | How |
|---|---|---|
| **GitHub App** (recommended) | GitHub Cloud / Enterprise Server | Install on org, auto-discovers repos, tokens auto-rotate |
| **Service Account PAT** | GitHub / GitLab / Bitbucket | Machine user with read-only scope |
| **Deploy Key** | Any (per-repo) | SSH key, most restrictive |

### Repo Discovery Flow

1. Admin adds a credential (GitHub App / PAT)
2. CodeGraph discovers repos via the platform API
3. All discovered repos appear in Admin Dashboard as "Unassigned"
4. Admin assigns repos to groups or leaves them ungrouped
5. All repos queue for indexing regardless

### Repo Change Detection — Three Equal Triggers

| Trigger | Use Case | How |
|---|---|---|
| **Webhook** | Orgs that allow inbound hooks | GitHub/GitLab/Bitbucket push events, signature validated |
| **Polling** | Enterprises blocking inbound webhooks (first-class, not a fallback) | `git ls-remote` on configurable interval (default 6h), lightweight |
| **Manual** | On-demand, force full re-index | UI button + API endpoint |

All three produce the same `repo.pushed` event via the event bus. Downstream workers don't know or care how the trigger happened.

**Git platform adapter interface:** Each git hosting platform (GitHub, GitLab, Bitbucket, Gitea, etc.) is implemented as an adapter behind a common interface covering: repo discovery, webhook parsing, credential management, PR comment posting, and API access. Adding support for a new git platform (e.g., Gitea, Azure DevOps) means implementing this adapter — the rest of CodeGraph (indexing, extraction, wiki generation) works unchanged.

### Repo Groups

**Purpose:** Logical architecture grouping for exactly two things:
1. **Scoped Q&A** — chat searches within a group's repos
2. **Cross-repo diagrams** — architecture diagrams across repos in a group

**Key rules:**
- Repos work fully without any group (wiki, per-repo chat, API catalog, diagrams)
- A repo can belong to multiple groups
- Deleting a group doesn't delete repos
- Any authenticated user can browse any group
- Groups are created manually by admin — no HR/team system integration
- Grouping is optional. Can happen before, during, or after indexing.

### Bulk Indexing (1000 repos)

When admin adds many repos at once:

- **Priority:** P0 webhook (real-time) > P1 manual > P2 bulk discovery
- **Concurrency:** Admin-configurable max concurrent jobs (default: 5)
- **Scheduling:** Small repos first within queue (fast wins)
- **Cost control:** Daily LLM cost cap. Pipeline extraction (no LLM) keeps going regardless.
- **Failure handling:** Failed jobs don't block the queue. Retry separately.
- **Admin controls:** Pause, resume, cancel, reorder from the dashboard.
- **No dependency on groups:** Every repo indexes independently. Groups are labels, not scheduling units.

### Scoped Search

| Scope | What It Searches |
|---|---|
| **Repo** | Single repo |
| **Group** | All repos in one group |
| **Cross-Group** | User selects multiple groups |
| **Org** | All indexed repos |

All search tools (entity graph, vector, trigram, wiki, API catalog) respect the selected scope via SQL WHERE filters.

### Cross-Repo Entity Resolution

Runs after individual repos are indexed. Triggers when:
- A repo finishes indexing that belongs to a group
- Admin assigns repos to a group

This doesn't run as part of individual repo indexing — it's a group-level concern handled by the background consistency job.

---

**Prev:** [Goals & Architecture](01-goals-and-architecture.md) | **Next:** [Entity Graph & Extraction](03-entity-graph-and-extraction.md) | [Index](00-index.md)
