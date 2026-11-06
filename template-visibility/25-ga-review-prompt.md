# Review: Template Visibility for every organisation (go-live)

You are reviewing the pull requests linked with this message. Together they complete **Template Visibility** in
DoubleTick, a WhatsApp CRM. Until now the feature was a pilot, switched on for two organisations. It is now going
live for **every organisation, existing and new**. This is a **full review of the feature**, not just of the diff.

---

## 0. Repos and scope

| Repo | Stack | PR branch → base | What the PR adds |
|---|---|---|---|
| `whatsapp-crm-backend` | NestJS + TypeScript, MikroORM + raw SQL, pnpm, vitest | `shivanshuverma/dev1-18063-template-visibility-access` → `master` (merge-base `affbbe0bb9`, 22 files) | One audience per template, across all its WABAs (§2.4 rules 9–16) |
| `ai-bot-backend` | Express, ESM JavaScript, `Schema` ORM + raw SQL with `:named` params, npm, Node 22, vitest | `shivanshuverma/dev1-18120-template-visibility-ai-bot-backend-counterpart` → `main` (merge-base `fdd378f8d`) | Audience panel across all WABAs (`getTemplateWabaIds`); Templates-page visibility filter, its options endpoint and a rate limit on it (§2.4 rule 21) |

- Both services share **one Postgres and one Redis**. They integrate through shared tables, not HTTP.
- The rest of the feature is **already on master/main**: crm PR #5192 and ai-bot PR #2207, merged 2026-09-25. That
  covers the lists, the guards and the send-path wiring. Review the whole feature as it stands at the PR heads, using
  §3. The branches are a few commits behind their base, so review a local merge where that matters.
- The front ends consume the contract. Read them for context; they are not under review:
  - `whatsapp-crm-desktop`: the "Visible to" column, the slider summary and the Edit-visibility modal. The filter UI
    is merged but commented out under `TODO(DEV1-13028)`.
  - `whatsapp-crm-mini`: the create/duplicate card and the member picker.

## 1. What go-live changes (the premise of this review)

- **Today** the org switch is on for 2 orgs. When it is off, every path behaves exactly as before the feature: one
  flag query, then no scope query, no audience SQL, and nothing filtered.
- **At go-live** it is on for every org and every new signup. Everything the switch used to protect now runs for
  everyone: every template list, every send guard, every public-API send, the picker and the visibility save. Any
  cost or failure on a switch-on path now reaches every org.
- **Day one:** before the switch is turned on, production's existing audiences are reset to Channel (SQL in §6).
  So on day one every org is on and every template is Channel. Customers then start restricting templates.
  - Many guards cost nothing extra while a template is Channel (§3.4), so day one hides most of the cost. **Measure
    both states:** all Channel, and a realistic share of restricted templates.
- **How the switch is turned on for everyone is not decided.** Nothing in any repo writes the flag today; the pilot
  orgs were set by SQL. The two options are:
  - (a) write the org-level row for every existing org, write one at org creation and on on-prem installs, and keep
    the switch as a kill switch;
  - (b) remove the switch from both backends and from the front-end flag endpoint.

  Evaluate both (§5.4).

Review on four axes, in this priority order:
1. **The existing product is not affected.** With the switch on, every flow behaves as it does on master/main, at an
   acceptable cost.
2. **Performance.** Every query the feature runs (§3.4, §3.5), measured at the largest org's scale.
3. **Reliability.** Failure modes, consistency between the repos, caches, concurrency and automated callers.
4. **Feasibility of the rollout.**

---

## 2. The product

### 2.1 Problem (from the PRD)

Templates are created per WABA (WhatsApp Business Account). Everyone with view-templates on a WABA sees and can use
every template on it. One WABA often hosts several independent teams, so teams see and act on each other's templates.
This feature limits who can see a template.

The PRD is devmetrics "Template Visibility access". It is short. Product and the solutions team clarified the rules in
§2.4 after it was written, and the code follows those rules. Where the PRD and §2.4 differ, §2.4 is the intended
behaviour.

### 2.2 Terms

| Term | Meaning |
|---|---|
| Template | One logical template = **name + language within the org** |
| Copy | One `"Template"` row per phone number (integration). Copies on one WABA share `remoteId`; across WABAs they share only name + language |
| WABA | `Integration.metadata->>'wabaId'`. A WABA has several numbers |
| Owner | A `UserRole` on `Role(name = 'OWNER', type = 'ORG')` in this org. This is not `Organization.ownerId` |
| Direct manager | The REPORTING_MANAGER of any team where the user is a visible member (`TeamRole.visibility IS TRUE`, not deleted). A user can have several |
| Below / subtree | Recursively, the visible members of the teams the user visibly manages, up to 10 levels |
| Anchor | `visibilitySetBy`: whoever last set the audience. `createdBy` never changes |
| WABA gate | Whether the user holds READ_TEMPLATES on the number (rule 4) |

### 2.3 Modes

| Mode | Stored as | Who can see it |
|---|---|---|
| Channel (the default) | `visibility IS NULL` | Everyone who passes the WABA gate, as before the feature |
| My team | `visibility = 'MY_TEAM'` | The anchor, the anchor's direct manager(s), everyone below the anchor (up to 10 levels), and owners |
| Custom | `visibility = 'CUSTOM'` plus `visibleToUserIds text[]` | The anchor, the anchor's direct manager(s), the picked users, and owners |

### 2.4 Rules (the PRD plus the later clarifications)

**Seeing a template**
1. Each copy gets two checks, before grouping: the WABA gate and the audience. A user sees the template under
   exactly those WABAs they can access. Owners skip the audience check but **not** the WABA gate. In both repos the
   owner flag is computed inside the scope query, so owners still pay for it.
2. My team is only the anchor, the anchor's direct manager(s) and the anchor's whole reportee hierarchy. The
   manager's manager is **not** in it. Owners see every template in every mode because they are owners (rule 1), not
   because they belong to the team.
3. Implicit members (owners, managers, the subtree) are **never stored**. They are resolved live on each request from
   `TeamRole` and `UserRole`, so org-chart changes apply immediately. Hidden ("bubbled") `TeamRole` rows are ignored.
4. The two backends compute the WABA gate differently. This is a known, accepted difference.
   - **crm:** the user is assigned to the number (Redis-cached) and holds a role with READ_TEMPLATES on it. Users
     with READ_TEAM_ASSIGNED_CONVERSATIONS also get every number a reportee can read.
   - **ai-bot:** the user holds a role with READ_TEMPLATES on the number, checked live.
5. A hidden template is missing from lists and info. Sending, editing, deleting or duplicating it by id returns 403.

**Changing the audience**

6. The only route is `PATCH /v1/template/:templateId/visibility`, used by the desktop modal. Content edits ignore any
   audience sent with them.
7. Edit rights: the user holds CREATE_TEMPLATES on at least one connected number of any of the template's WABAs,
   **and** is an owner, the anchor, or the anchor's direct manager. The PRD's "original creator" means the anchor.
8. The `templateId` is only a handle for name + language. It must be a copy the editor can see.
9. A save writes **every live copy on every WABA**, including copies on disconnected numbers, in one `UPDATE`: the
   same mode, the anchor set to the editor, and the same picks. The editor's own WABA access never narrows the
   write. People outside the new audience lose access, including the creator.
10. Every save re-anchors on the editor, even an unchanged save or a save to Channel.
11. A save does not bump `dateUpdated`, so it doesn't trigger the 24-hour lock on content edits.
12. Custom picks must have template access on at least one number of any of the template's WABAs. Other picks are
    dropped and returned in `visibilityWarnings`. Owners, the editor and the editor's manager are implicit and are
    stripped before storing.
13. With the switch off, the save returns 400 for every mode.

**Copies arriving on a number or WABA** (all copies of a template must always share one audience)

14. **Create:**
    - If the name + language is already live in the org, the new copies join that audience and the request's
      choice is ignored.
    - Otherwise the request is resolved once and the same fields go on every WABA. A create on more than 10 WABAs is
      queued per WABA and carries the resolved fields.
15. **Other copy paths:**
    - Add-to-WABAs, the internal duplicate, and a recreate while a live copy exists all join the existing audience.
    - A template deleted everywhere and then recreated is Channel.
    - A duplicate under a new name gets its own audience.
16. **Meta sync:**
    - A new number on a WABA inherits from its sibling copy.
    - A template that appears on a new WABA joins the live audience.
    - A revived soft-deleted row takes the live audience, or Channel if no live copy exists.
17. **Name collision:** a create fails on a WABA where the name (in any language) belongs to a template the creator
    can't see. The error is "This name is already in use by a template you don't have access to. Please choose a
    different name." The other WABAs go ahead.

**Picker, display and filter**

18. `POST /v1/template/visibility/eligible-members`:
    - With a `templateId`, editors get all of the template's WABAs. Non-editors keep the requested WABAs (mini's
      duplicate flow).
    - It returns `fixed` (locked chips: self, owners, manager), `selected` and `members`.
    - It doesn't check the switch.
19. `GET /v1/template/:templateId/visibility` returns `{ canEditVisibility, mode, availableModes }`.
20. **Labels:**
    - My-team rows carry `visibilityLabel`: "My team" for the anchor, "Team <anchor name>" for everyone else.
    - Restricted rows on the ai-bot list and `/info` carry an `audience` block: who can see the template.
21. **Templates-page filter (ai-bot):**
    - The request field is `filters.visibility = { modes: ["CHANNEL" | "CUSTOM"], teamAnchorIds }`, with at most 5,000
      ids.
    - The picks are OR-ed together, then AND-ed with access, so the filter only narrows. It is ignored while the
      switch is off.
    - The options come from `GET /v1/templates/visibility-filter-options`: Channel, Custom, and one option per
      My-team anchor among the rows the viewer can list. The endpoint is rate limited to 60/min per user.
22. The front end learns the switch state from ai-bot `GET /v1/ai/organizations/feature-permissions`
    (`template_visibility_enabled`).

**Automated callers**

23. API keys made in the UI belong to the org's "Public api" system user, and bot-builder keys to the "Bot" system
    user. Both hold OWNER, so nothing is filtered for them. They still pay for the scope query on restricted
    templates (rule 1). Legacy member-bound keys are filtered as that member.
24. Scheduled sends are not re-checked when they fire. Bots, AI agents, sync, queue consumers and analytics are not
    filtered.

### 2.5 Known and accepted

Don't report these as bugs. If go-live changes their impact, report that and say how.

- **Access and edit rights:**
  - Owners need WABA access to see templates.
  - crm and ai-bot use different WABA gates (rule 4).
  - Every save re-anchors, so edit rights climb with each manager's save.
  - A template anchored on someone removed from the org stays anchored on them; only owners can then edit it.
  - Meta-synced and pre-feature templates are anchored on the integration owner (`createdBy`), so only owners, or
    that owner's direct manager, can restrict them.
  - Demoted owners and former managers lose Custom templates, because implicit members are never stored.
- **Separate tickets:**
  - Content edits and creates can act on WABAs the editor can't access.
- **Paths that don't filter:**
  - the old UI routes (v1/v2 chat send, v1/v2/v3 broadcast, v1 retarget);
  - public broadcast and broadcast-via-sheet;
  - forward and action templates;
  - `GET /v1/template/status/:status/wabas` and the bot-builder template resolver;
  - analytics and exports;
  - the template-status bell, which notifies the creator.
- **Pre-existing bugs, already ticketed:**
  - the CleverTap import lets the query string override `orgId`/`userId`;
  - a body `user` overrides the caller on the v1 send;
  - `PATCH /v1/scheduled-messages/:id` has no user check.

---

## 3. How it is coded

### 3.1 Storage and indexes

- **New `"Template"` columns:**
  - `visibility varchar(16)`: NULL means Channel; there is no CHECK constraint.
  - `visibilitySetBy varchar(15)`: no foreign key.
  - `visibleToUserIds text[]`.
- **Migrations:** ai-bot owns the DDL for the shared database, and all three have run in production:
  - `schema_migrations/20260826120000_add_template_visibility_columns.js`
  - `20260916120000_add_template_visibility_set_by.js`
  - `20260925120000_insert_template_visibility_update_audit_log_event.js`, which inserts an audit event row and has
    no DDL.
- **The switch:** an `"OrgVariable"` row with `name = 'template-visibility-enabled'`, at org level
  (`"integrationId" IS NULL`), with value exactly `'true'`.
- **Indexes known to the author.** Most base tables predate the migrations folders, so take the real list from
  `database.sql` or ask for production's.

| Table | Index | Source |
|---|---|---|
| Template | `Template_visibleToUserIds_gin`: GIN, partial `visibility = 'CUSTOM'` | ai-bot migration `20260826120000` |
| Template | `Template_visibility_visibilitySetBy_index`: btree, partial `visibility IS NOT NULL` | ai-bot migration `20260916120000` |
| Template | `integrationId`, `language`, each single-column | crm entity decorators only, not confirmed in production |
| Template | `(name, language)`, `(orgId, name, language)`, `remoteId` | **not found** anywhere (a test DDL creates `Template_remoteId_index`) |
| TeamRole | `TeamRole_pkey (orgId, teamId, userId, roleId)`, `TeamRole_orgId_userId_index`, `TeamRole_visibility_dateDeleted`, `idx_team_role_user_id`, `index_TeamRole_teamId` | production, as supplied in the previous review |
| UserRole | PK `(orgId, userId, roleId)`, `index_UserRole_orgId_roleId` | crm entities |
| OrgVariable | PK `(orgId, name)`. ai-bot's model also declares a unique `(orgId, name, integrationId)` | entities / models |
| DirectSendTemplate | PK `(remoteId)` | ai-bot migration `20260723120000` |
| Integration | no `orgId` or `metadata->>'wabaId'` index found | — |

### 3.2 The read path

Each request first reads the switch, in its own query, in both repos. No cache is used, although one exists in each
repo:
- crm `OrgVariablesService.getOrgVariableFromRedis`: key `variable:<orgId>:<name>`, 1 h TTL;
- ai-bot `libs/organization/org-variable.js`: 24 h TTL.

If the switch is off, the viewer is unrestricted and nothing else runs. If it is on, a recursive CTE computes the
viewer scope `{ viewerUserId, isOwner, directReportees, ancestors, unrestricted }`, and a predicate is built from it:

```sql
"Template"."visibility" IS NULL
OR "Template"."visibilitySetBy" = :viewer
OR "Template"."visibilitySetBy" = ANY(:directReportees)        -- the viewer is the anchor's direct manager
OR ("Template"."visibility" = 'MY_TEAM' AND "Template"."visibilitySetBy" = ANY(:ancestors))
OR ("Template"."visibility" = 'CUSTOM' AND :viewer = ANY("Template"."visibleToUserIds"))
```

- For an owner, or with the switch off, the predicate is `TRUE`. But the owner flag comes out of the same CTE, so
  owners still pay for it.
- It is AND-ed into each list query before `GROUP BY`. Every list scopes by `integrationId` first, so the predicate
  filters rows after the scan; it can't drive an index.
- **The CTE and the predicate must match between the two repos.** crm also has an in-memory twin of the predicate for
  single-template guards. The twin anchors on `visibilitySetBy ?? createdBy`; the SQL uses `visibilitySetBy` only.
- **crm `em.execute(em.raw(...))` writes every value into the SQL text** (MikroORM 5.9.8 `platform.formatQuery`). So
  `ANY(ARRAY[:directReportees])` reaches Postgres as a literal list, the SQL text differs for each viewer and array
  length, and no plan is reused.
  - Affected: Q2–Q8, Q13, Q19, Q22, Q23, Q25–Q27 in §3.4.
  - The bot-builder queries (Q9, Q10) and the sync queries (Q30–Q34) use `$n` bind parameters.

### 3.3 Where to read

**crm** (paths are under `src/modules/template/` unless noted):

| Area | Files |
|---|---|
| Rules | `domain/template-visibility.ts`: the predicate, the in-memory `canViewTemplate`, `canEditTemplateAudience` |
| Core service (TVS) | `services/template-visibility/index.service.ts`: the switch (`isVisibilityEnabledForOrg`), the `getViewerScope` CTE, eligibility (`getEligibleMembersForIntegrations`), guards (`assertCanUseTemplate*`), write access (`getAudienceWriteAccess*`), `resolveVisibilityForCreate`, `resolveVisibilityForWabas` |
| Data (TDS) | `services/template-data/index.service.ts`: where the predicate enters each list (`buildTemplateVisibilityClause` call sites), `updateTemplateAudience`, `getTemplatesForOrgByNameAndLanguage`, the name-collision finder |
| Sync | `services/sync-template/index.service.ts`: `getLiveAudiencesByNameLanguage`, `getSoftDeletedTemplates`, `setRevivedTemplateAudiences`, `inheritedAudience` |
| Visibility endpoints | `use-cases/v1/update-template-visibility`, `get-template-visibility-permissions`, `get-eligible-members` |
| Create and copies | `use-cases/v1/create-template` (plus `bulk-template-create*`), `duplicate-template-to-wabas`, `recreate-deleted-template` |
| Lists | `use-cases/v1/get-templates`, `get-templates-with-filter`, `get-templates-with-search-or-filter`, `v2/get-template` (with counts and folders); bot-builder `get-bot-builder-template-groups`, `get-bot-builder-template-integrations`; CleverTap `public-api/use-cases/v2/clever-tap/get-templates-for-clevertap` |
| Send guards | chat-message v3 controller, v4 broadcast, v2 retarget; `public-api/service/public-api-send-message.service.ts` (PASM), the v1 `send-template-message` use case, the v2 cloud-api send, the CleverTap and WebEngage consumers, the public-API queued sends, delete and bulk delete |
| Permissions | `src/shared/infra/auth/permission/services/template-access-permissions.service.ts` |

**ai-bot:**

| Area | Files |
|---|---|
| Visibility | `libs/templates/visibility.js`: `getViewerScope`, `buildVisibilityClause`, `buildVisibilityFilterClause`, `templateVisibilityLabel` |
| Templates | `libs/templates/index.js`: `getAllowedTemplateData` (`POST /v1/templates`), `getTemplateInfo` (`/info`), `getWabasToAssignTemplate` (`/wabas`), `getTemplateEditDetails` (`/is-editable`), `getVisibilityFilterOptions` (`/visibility-filter-options`). `getTemplates({ templateIds })` is the unfiltered read used by chat-message, group and the funnel-analytics consumer |
| Audience panel | `libs/templates/audience.js`: `resolveAudiences` |
| Switch | `libs/ai/feature-permission.js`, `shared/libs/org-variable/index.js` |
| Routing | `router/templates/v1/*`, `shared/constants/rate-limiter-config.js` |

**Front ends (context only):**
- desktop: `src/modules/templates/components/template-table/template-slider/edit-visibility-modal/*`
- mini: `src/modules/templates/v1/hooks/use-template-visibility.ts`, `use-cases/create-template.use-case.ts`

### 3.4 crm query inventory (the author's trace; verify every line, and add anything missed)

Short names: TVS and TDS as in §3.3; UPD, PERM, ELIG and CREATE are the use cases; SYNC is the sync service; PASM is
the public send service.

| ID | Where | What it does | When it runs |
|---|---|---|---|
| Q1 | TVS `isVisibilityEnabledForOrg` → `OrgVariablesService.getOrgVariables` (ORM find) | Switch read: `orgId`, `name`, `integrationId IS NULL` | Every list; every guard that needs a scope; **twice** in PATCH and GET-permissions; per WABA in create. Runs with the switch off too. Uncached |
| Q2 | TVS `getViewerScope` | Recursive CTE over `UserRole`/`Role` (owner) and `TeamRole` pairs (reportees, ancestors), depth < 10, `array_agg(DISTINCT)` | After Q1 when on, **owners included**. Termination relies on the depth cap: UNION includes `depth`, so it doesn't stop cycles (the comment says otherwise) |
| Q3 | TDS `getTemplatesForIntegration` | V1 list: `integrationId = ANY`, `type`, `NOT name = ANY`, `NOT EXISTS DirectSendTemplate`, predicate, optional `name ILIKE '%…%'`, `ORDER BY dateCreated DESC LIMIT` | `GET /v1/template` (limit 10000), `GET /v1/template/filter` (2000), `GET /v2/p/templates` (10000), `GET /v2/p/v2/templates`; bot-builder header path cached 1 min |
| Q4 | TDS `getTemplatesWithSearchOrFilters` | Search-or-filter list joined to `Integration.orgId`, predicate, `ORDER BY dateCreated DESC` | `POST /v1/template/filter`. Pre-existing on master: **no `LIMIT` in the SQL** (the limit is passed but unused), so it returns every match; also an unquoted `"Template".dateCreated` in the keyset and `name != ANY(...)` |
| Q5 | TDS `getTemplatesForIntegrationV2` | V2 grouped list: `GROUP BY wabaId, name, language`, user-chosen `ORDER BY`, `LIMIT` (2000), jsonb search. When on, adds `LEFT JOIN "User" AS "AnchorUser" ON id = COALESCE(visibilitySetBy, createdBy)` and `(array_agg(jsonb_build_object(...)))[1]` **for every row before `GROUP BY`**, owners included, although the label is used only on My-team rows | `POST /v2/template` |
| Q6, Q7 | TDS `getTotalTemplateCounts`, `getTemplateCountFolderWise` | Counts, each with the predicate | `POST /v2/template`, in parallel. The inlined reportee list appears 3 times per request (Q5–Q7) |
| Q8 | CleverTap `execute` | Import page `ORDER BY id LIMIT :limit+1`, predicate | `GET /v2/p/templates/import`. The `limit` query param has no maximum |
| Q9, Q10 | TDS `getBotBuilderTemplateGroups`, `getBotBuilderTemplateIntegrations` | Bot-builder grouped list (CTE, `GROUP BY name, language`, `LIMIT 10000`) and its channel picker (redundant `md5(jsonb_build_array(name, language))` filter); predicate with `$n` binds | `/v2/p/templates/grouped` (cache 1 min per org + user; a hit skips Q1/Q2), `/grouped/integrations` |
| Q11 | TVS `assertCanUseTemplateById` → `TDS.getTemplate({id})` | Template by PK for the guard | Every composer v3, broadcast v4 and retarget v2 request, in every org, whatever the switch |
| Q12 | TDS `getTemplateForWabaIntegrationRaw` | The send path's own template read, which feeds the guard | **Per message** in PASM `sendMessage`/`sendMessageRaw`/group sends. `sendMessages` and `sendMessagesRaw` run them in `Promise.all` over the messages. Also the queue consumer (one per message), CleverTap and WebEngage (via the queue), and cloud-api |
| Guard | TVS `assertCanUseTemplates` | No query if the template is Channel or anchored on the viewer. Otherwise **Q1 + Q2 once per call**, for owners too | So a public batch of N messages on a restricted template = N × (Q1 + Q2) in parallel, even for an owner key |
| Q13 | v1 `send-template-message` | `Template JOIN Integration WHERE orgId, name … ORDER BY dateCreated LIMIT 1` | `POST /public-api/send-template-message` |
| Q14–Q17 | delete, bulk delete, duplicate, edit | The lookup that feeds each guard | One guard scope per request |
| Q18 | UPD, PERM, TVS | Template by PK | PATCH, GET-permissions, picker |
| Q19 | TDS `getTemplatesForOrgByNameAndLanguage` | All live copies: `remoteId IN (SELECT … WHERE orgId, name, language, live)`. **The outer query has no `orgId`**, so other orgs' rows sharing a remoteId come back and are dropped in memory. It also selects `Integration.credentials`. Rows with NULL `remoteId` are excluded | **Every create in every org, switch on or off** (`resolveVisibilityForCreate`), including public-API create and the loops that create many templates (frictionless registration, default templates, internal duplicate and create-deleted). Also PATCH, GET-permissions, the picker and multi-edit, and **twice** in add-to-WABAs |
| Q20 | `getMultipleWhatsappIntegrationWithWabaAccountId` | Connected integrations of WABAs (`metadata.wabaId $in`) | PATCH, GET-permissions, picker (1–3×) |
| Q21 | `getIntegrationIdsWithCreateTemplates` | CREATE_TEMPLATES per integration (Redis hash first) | PATCH, GET-permissions, picker |
| Q22 | TVS `getImplicitAudienceUserIds` | Owners ∪ the editor's managers | PATCH (every mode), create whenever a mode is sent (**Channel included**), each bulk item |
| Q23 | TVS `getEligibleMembersForIntegrations` | Eligible members: 7-table join (Integration, UserWabaIntegration, UserWabaIntegrationRole, RolePolicy, Policy, User, UserOrganization), optional `u.name ILIKE '%…%'`, `ORDER BY name LIMIT 500`; an exact mode with `u.id = ANY(:userIds)` and no limit | Picker (2×); PATCH and create with Custom picks |
| Q24 | TDS `updateTemplateAudience` (`nativeUpdate`) | `UPDATE "Template" SET visibility, visibilitySetBy, visibleToUserIds WHERE name, language, integrationId IN (…), dateDeleted IS NULL` | PATCH, one statement |
| Q25–Q27 | TVS `getFixedAudienceCandidates`, `getSelectedMembersForTemplate`, `getPreviousAudienceAnchors` | Picker chips: owners and managers; stored picks via `unnest`; previous anchors and their managers | Picker with `templateId` |
| Q28 | TDS `getTemplateOccupyingName` | Name collision: `findOne(name, integrationId, live)`. If the occupant is restricted, then Q1 + Q2 | **Per WABA** of a create, in parallel; the scope is not reused across WABAs |
| Q29 | TDS `createNewTemplates` | `findOne(integrationId, name, language)` **sequentially per integration**, then one flush with the audience fields | Every create |
| Q30 | SYNC `getLiveAudiencesByNameLanguage` | `DISTINCT ON (name, language) … JOIN unnest($1, $2) … WHERE live AND orgId ORDER BY dateCreated, id`; joins on `t.language::text` | Once per integration synced, **every org, switch ignored** |
| Q31 | SYNC `getSoftDeletedTemplates` | `id = ANY($1) AND dateDeleted IS NOT NULL` | Per integration synced |
| Q32 | SYNC `setRevivedTemplateAudiences` | **One `UPDATE` per (name, language) group, sequential** | Per integration synced, when rows are revived |
| Q33 | SYNC sibling lookup | `DISTINCT ON (remoteId)` by `remoteId = ANY($1)`, same-WABA first | Per integration synced |
| Q34 | TDS `addNewTemplatesRaw` | Bulk insert including the audience columns | Per integration synced |

**Cost per crm route with the switch on**

| Route | Queries |
|---|---|
| `GET /v1/template`, `/v1/template/filter`, `/v2/p/templates`, `/v2/p/v2/templates` | Q1 → Q2 → Q3 |
| `POST /v1/template/filter` | Q1 → Q2 → Q4 (unbounded result) |
| `POST /v2/template` | Q1 → Q2 → Q5, then Q6 ‖ Q7 |
| CleverTap import | Q1 → Q2 → Q8 |
| Bot-builder grouped list / channel picker | cache, or Q1 → Q2 → Q9 / Q10 |
| Composer v3, broadcast v4, retarget v2 | Q11, plus Q1 + Q2 if the template is restricted and not the viewer's |
| Public sends, CleverTap, WebEngage, queue consumer, cloud-api | Q12 per message, plus Q1 + Q2 **per message** if restricted |
| `PATCH /:id/visibility` | 2×Q1, Q2, Q18, Q19, Q20, Q21 ‖ Q1+Q2, Q22, [Q23], Q24, one Redis delete per integration, the audit row. No transaction, no row lock |
| `GET /:id/visibility` | 2×Q1, Q2, Q18, Q19, Q20, Q21. No owner shortcut |
| `POST /visibility/eligible-members` | Up to ~14: Q11 twice, Q1 + Q2 twice, Q19, Q20 (2×), Q21, Q25, Q23 (2×), Q26, Q27. No Redis, no switch check, no maximum on `wabaIds` |
| Create | Q19 always; Q22 if a mode is sent; per WABA Q28 (+ Q1 + Q2 if the occupant is restricted) and Q29 per integration. More than 10 WABAs: each queue item repeats Q1, Q22, Q28, Q29 |
| Sync (any org, any switch state) | Per integration: Q30, Q31, Q32 × groups, Q33, Q34 |

**Arrays with no upper bound:**
- `visibleToUserIds` has no `ArrayMaxSize` in the create, duplicate or update-visibility DTOs. Q24 writes the same
  array to every copy.
- `wabaIds` has no maximum on the picker.
- `directReportees` is every member of every team the viewer manages, written into the SQL text.

### 3.5 ai-bot query inventory (the author's trace; verify)

None of these queries is cached.

| ID | Where | What it does | When it runs |
|---|---|---|---|
| A0 | `shared/libs/org-variable/index.js` `getOrgVariable` | Switch read: `SELECT * … WHERE orgId, name`, takes `rows[0]` with **no `ORDER BY` and no `integrationId IS NULL`**. `visibility.js` treats a row carrying an `integrationId` as off | Every template route, switch on or off |
| A1 | `libs/templates/visibility.js` `getViewerScope` | Recursive CTE (owner `EXISTS`, reportees, ancestors < 10); same UNION/`depth` caveat as Q2 | After A0 when on; owners included |
| A2 | `index.js` `getIntegrationIdsWithReadPermission` | READ_TEMPLATES integrations; `GROUP BY integration, role`, so duplicate ids appear | List, `/info`, filter options (in parallel with A0/A1) |
| A3 | `index.js` `getAllowedTemplateData` | List: `integrationId = ANY`, `type`, `NOT EXISTS DirectSendTemplate`, `NOT name = ANY`, predicate, the new filter clause, search (`ILIKE`, `jsonb_array_elements`), keyset; `GROUP BY name, language` over **every matching group before `ORDER BY … LIMIT 20`**; about 20 aggregates. When on, it adds `jsonb_agg(DISTINCT jsonb_build_object(...))`, `COUNT(DISTINCT COALESCE(...))` ×2, three `array_agg`s and the `AnchorUser` join on `COALESCE(...)`. `(array_agg(visibility))[1]` has no `ORDER BY` | `POST /v1/templates` |
| — | `visibility.js` `buildVisibilityFilterClause` | `COALESCE(visibilitySetBy, createdBy) = ANY(:teamAnchorIds)`, up to 5,000 ids | List, when the filter is sent and the switch is on |
| R1 | `libs/templates/pacing.js` | Redis `MGET` held marks, 300 ms timeout, fails open | List, `/info` |
| A5 | `audience.js` `getOwners` | Org owners | Audience panel: any page with a restricted row |
| A6 | `audience.js` `getDirectManagers` | Managers of up to 20 anchors | Audience panel |
| A7 | `audience.js` `getSubtrees` | Recursive CTE of My-team subtrees, **no `LIMIT`**. The 500-member cap trims the response afterwards | Audience panel with a My-team row |
| A8 | `audience.js` `getCustomPicks` | Picks via `unnest(names, languages)` and `unnest(visibleToUserIds)` | Audience panel with a Custom row |
| A9 | `audience.js` `getTemplateWabaIds` (**new in this PR**) | Every WABA each restricted (name, language) is live on: `unnest` JOIN `"Template"` on name + language with **no `Template` org or integration filter**; only the `Integration` join on `orgId` scopes it | Audience panel |
| A10 | `audience.js` `getWabaAccess` | Candidates' READ access: 7-table join, `metadata->>'wabaId' = ANY`, `a."userId" = ANY(:userIds)` with every candidate on the page (owners + managers + whole subtrees + picks; can be thousands) | After A5–A9 |
| A11 | `index.js` `getVisibilityFilterOptions` (**new in this PR**) | My-team anchors: the list's gates, `visibility = 'MY_TEAM'`, predicate, `GROUP BY COALESCE(...)`, no `LIMIT` | Filter options, when on |
| A12 | `index.js` `getTemplateInfo` | One template's rows, `jsonb_agg` per integration; `AnchorUser` join when on. Pre-existing: `if (!integrationIds)` is always false, and `throw BadRequestError(...)` is missing `new` | `/info` |
| A13 | `index.js` `getWabasToAssignTemplate` | Pre-existing: joins `UserWabaIntegrationRole` on `userId` only (no `orgId` or integration), which fans out across the user's roles in every org; the predicate is evaluated per fanned row | `/wabas` |
| A14 | `index.js` `getTemplateEditDetails` | Rows by name + language, predicate; no READ_TEMPLATES gate | `/is-editable` |
| A15 | `libs/ai/feature-permission.js` | Same SQL as A0, for `template_visibility_enabled` | `GET /v1/ai/organizations/feature-permissions` |
| A16 | `index.js` `getTemplates({ templateIds })` | `SELECT * WHERE id = ANY` with no visibility and no org filter (pre-existing) | Non-UI callers: chat-message, group, funnel consumer. Must stay unaffected |

**Cost per ai-bot route with the switch on**

| Route | Switch off | Switch on |
|---|---|---|
| `POST /v1/templates` | `[A2 ‖ A0]` → A3 → R1: 3 SQL + 1 Redis | `[A2 ‖ (A0 → A1)]` → A3 → `[A5 ‖ A6 ‖ A7? ‖ A8? ‖ A9]` → A10, with R1: up to 10 SQL in 5 sequential stages |
| `GET /info` | `[A2 ‖ A0]` → A12 → R1 | `[A2 ‖ (A0 → A1)]` → A12 → R1 → A5–A9 → A10 |
| `GET /wabas`, `GET /is-editable` | A0 → A13 / A14 | A0 → A1 → A13 / A14 |
| `GET /visibility-filter-options` | rate limit → `[A2 ‖ A0]` → return the modes | rate limit → `[A2 ‖ (A0 → A1)]` → A11 |

**Index notes from the trace:**
- `:viewer = ANY("visibleToUserIds")` can't use a GIN index, because GIN array operators are `@>` and `&&`. No query
  in either repo uses `@>` on that column. **`Template_visibleToUserIds_gin` is never used.**
- The `visibilitySetBy` partial index can't serve the OR predicate as a whole.

---

## 4. Scale to test at

- **Users:** the largest org has about 8,000.
- **Templates:** one org has about 30,000. Copies = templates × numbers.
  - The previous review measured the ai-bot list there: 325 → 467 ms of SQL with the audience aggregates. Today that
    cost is paid only with the switch on; after go-live it is paid by every org.
- **Hierarchy:** up to 10 levels (the cap), users with several managers, and possibly cycles in old data.
- **Owners:** typically 2–25 per org.
- **Earlier measurement:** the scope CTE cost about 7.5 ms per list request.
- **Public-API sends** are the highest-volume path: CleverTap and WebEngage campaigns, and batched sends.
- **Unknown to us:** production's `"Template"` row count and its spread across orgs, and per-org send volume. Take
  them from `database.sql` statistics, or ask.

---

## 5. What to check

### 5.1 The existing product is not affected

Compare base (master/main) with the PR head in three states: the switch on with every template Channel; the switch
on with a realistic share of restricted templates; and the switch off.

- **Every list:**
  - crm: `GET /v1/template`, `/filter`, search-or-filter, `POST /v2/template` with counts and folders;
  - ai-bot: the list, `/info`, `/wabas`, `/is-editable`;
  - bot-builder lists;
  - public `GET /v2/p/templates`, `/v2/p/v2/templates`, and the CleverTap import.

  Check that rows, counts, order, pagination/cursors and response shape (apart from the visibility fields) are the
  same. Check as an owner, a manager, an agent, and a user whose access comes through the reportee extension.
- **Every send:**
  - composer v3, broadcast v4, retarget v2;
  - public API v1, v2, cloud-api and frictionless; CleverTap, WebEngage, queued sends;
  - bots, scheduled sends, and AI agents (A16).

  Check that there is no new failure, and state the added DB work per message (§3.4 "Guard").
- **Writes:**
  - every create: Q19 now runs on all of them. Check that a multi-WABA create still succeeds partially, and cover
    the public-API create and the bulk registration loops;
  - add-to-WABAs, duplicate, recreate;
  - delete and bulk delete, content edit;
  - sync onto a new number and onto a new WABA.
- **Member-bound API keys with Channel templates:** unchanged.
- **Cost:** for each path above, base vs PR with the switch on.

### 5.2 Performance

**Method.**
- Use a throwaway Postgres 16 loaded from `database.sql` (production schema and indexes), seeded to the §4 scale,
  plus many small orgs and "other org" rows.
- Execute every query in §3.4 and §3.5 as written:
  - with real parameter shapes;
  - for crm, with the values inlined exactly as MikroORM sends them;
  - under `EXPLAIN (ANALYZE, BUFFERS)`, reporting the warm run.
- Compare base vs PR, and switch on vs off. Use five viewers: an owner, an agent, a deep manager, a manager of a
  large team, and a Custom pick.
- Then load-test the hot endpoints locally (1 → 10 → 50 → 200 concurrent): p50/p95/p99 latency, errors and pool
  saturation.
- Never run against a deployed URL or a shared database.

**Questions.**
1. **The switch read (Q1, A0, A15)** is uncached in both repos; PATCH and GET-permissions read it twice. After
   go-live it runs on every template request and on every guarded send.
   - Should it use the existing Redis caches (§3.2)? What invalidation would they need?
   - Or should the read go away along with the switch (§5.4)?
   - Also fix A0's `rows[0]` with no `ORDER BY` and no `integrationId IS NULL` filter.
2. **The scope CTE (Q2, A1):**
   - What does one call cost at 8,000 users, depth 10, wide teams and several managers?
   - What is the plan inside the recursion? `TeamRole_pkey` leads with `teamId`; is `(orgId, userId)` used?
   - Owners pay for it on every list and on every restricted guard. Would a cheap owner check first, or caching the
     scope per request or per user, pay off?
3. **Per-message guards (§3.4 "Guard", Q12):** a batch of N public messages on a restricted template runs N × (Q1 +
   Q2) in `Promise.all`, even for an owner key. Measure this at campaign volume, with pool usage, and propose
   resolving the scope once per batch or per key user.
4. **The predicate:**
   - Does it change any list plan at 30,000 templates?
   - The GIN index is never used (§3.5 index notes). Should the predicate use `@>`, or should the index be dropped
     to save write cost?
   - crm inlines `directReportees` into the SQL text (three times on `POST /v2/template`), so plans are never reused.
     What are the parse/plan cost and the pg_stat_statements impact? Should these queries bind arrays as parameters,
     as the bot-builder and sync queries do?
5. **List aggregates:**
   - ai-bot A3 aggregates every matching group before `LIMIT 20`, plus the switch-on aggregates and the `AnchorUser`
     join.
   - crm Q5 joins `AnchorUser` and builds a jsonb object for every row before `GROUP BY`, for owners too.

   What do these cost at 30,000 templates? Now that copies can't differ (no "Mixed"), which aggregates are still
   needed? Could the anchor name be fetched for My-team groups only, after the page is cut?
6. **The ai-bot audience panel (A5–A10):**
   - A7 returns whole subtrees with no `LIMIT`.
   - A10 takes every candidate as `= ANY(:userIds)`.
   - A9 (new) has no `Template` org filter: check for a cross-org scan by name + language, and whether an index or
     an `orgId` / `integrationId` filter fixes it.
7. **The ai-bot filter** applies `COALESCE(...) = ANY(:teamAnchorIds)` with up to 5,000 ids, and A11 groups by the
   same expression.
8. **Create path:**
   - Q19 now runs on every create in every org. Its outer query has no `orgId`, and no `(orgId, name, language)` or
     `remoteId` index was found. Measure it on the bulk registration loops.
   - Q28 runs per WABA with no shared scope, and Q29 runs per integration, sequentially.
   - Q22 runs even for Channel.
9. **The picker (Q23, Q25–Q27):** about 14 queries per call, the same template read twice, `ILIKE '%…%'` on
   `User.name` over 8,000 members, `LIMIT 500`.
10. **The save (Q24):** one `UPDATE` over every live copy, one row per number that has the template.
    - How long does it hold locks? Which triggers on `"Template"` fire?
    - How does it interact with Meta-webhook template updates, which run `UPDATE … WHERE "remoteId"` over the same
      rows, once per connected number in parallel?
11. **Sync (Q30–Q33)** runs for every org whatever the switch state, per integration synced. Q32 is one `UPDATE` per
    group, run sequentially. Q30 joins on `language::text`. Check the plans, the indexes, and the cost of a large
    WABA sync.
12. **Pre-existing hot spots that become reachable by more users:** Q4 has no `LIMIT`; A13 fans out across orgs.
    Report them separately, with plans.
13. **Indexes:** for each query, name the index it needs and recommend it with `CREATE INDEX CONCURRENTLY`. Where the
    plan depends on production statistics, ask for the production plan.

### 5.3 Reliability

1. **Errors:** a failed switch read or scope query throws. Classify each throw as an expected state or a real
   failure, and make sure expected 400s and 403s don't flood Sentry. A scope-query failure now fails sends (§3.4
   "Guard").
2. **Parity:**
   - crm and ai-bot must agree on the CTE and the predicate.
   - crm's in-memory twin uses `visibilitySetBy ?? createdBy`, while the SQL uses `visibilitySetBy` only.
   - The two repos' WABA gates differ (accepted). Say whether that difference matters more at go-live.
3. **One audience must hold across copies.** Find every way copies can still diverge:
   - **PATCH vs queued create:** a PATCH between enqueue and consumption of a create on more than 10 WABAs. Later
     items carry the old resolved audience.
   - **PATCH vs sync:** sync reads Q30 and writes later, with no transaction.
   - **PATCH vs create:** the copy set is read by Q19 and written by Q24, so a copy inserted in between is not
     covered.
   - **Rows missed by the save:** Q19 excludes rows with NULL `remoteId`.
   - **Two PATCHes:** last write wins.
4. **Data shapes found in production:**
   - hierarchy cycles: termination depends on the depth cap, not on `UNION`;
   - users in several teams;
   - removed anchors, whose `TeamRole` rows are hard-deleted;
   - soft-deleted users and rows with `createdBy` NULL;
   - disconnected numbers;
   - a template on 10 or more WABAs;
   - an org with both an org-level and an integration-level flag row (A0 then flips arbitrarily).
5. **Caches after a change:**
   - **public-API template cache** (`PUBLIC_API_TEMPLATE_*`): 1 day, cleared per integration on save. No guard reads
     it; guards re-read the database, and the cache is used for parameter validation only;
   - **bot-builder lists:** 60 s, per org + user, not cleared;
   - **crm WABA assignment:** Redis, 2 days;
   - **CREATE_TEMPLATES:** a Redis hash;
   - **the front end** reads the flag once per page load.
6. **Deploy:**
   - items already queued for a create on more than 10 WABAs carry no resolved audience;
   - old pods running against new data;
   - rollback.
7. **Automated callers at go-live:** member-bound API keys start failing sends of restricted templates.
   - v1 and cloud-api: 403.
   - v2 queued, frictionless, CleverTap and WebEngage: ENQUEUED, then FAILED plus a status webhook.

   Provide a read-only query that counts member-bound keys per org in production.
8. **Bounds:**
   - no `ArrayMaxSize` on `visibleToUserIds`;
   - no maximum on the picker's `wabaIds`;
   - no maximum on the CleverTap `limit`;
   - the subtree with no `LIMIT`;
   - 5,000 filter ids.

   State the worst case for each.
9. **Paths that don't filter (§2.5):** say which of them become a real leak at go-live, ranked by how reachable they
   are from current clients.

### 5.4 Feasibility of the rollout

1. **Turning it on for everyone:** compare (a), backfilling org rows and writing one at org creation and on on-prem
   installs, with (b), removing the switch. Cover:
   - cost per request (Q1, A0, A15);
   - the value of keeping a kill switch;
   - the front-end flag endpoint;
   - on-prem.
2. **Order:** deploy both backends (in either order), run the production reset (§6), then turn the switch on. Say
   what breaks if these happen in another order, and what the 1-day public-API template cache holds in between.
3. **Rollback:** confirm that turning the switch off returns behaviour to "as before" while stored audiences stay.
4. **On-prem:**
   - the same code lands on `onprem-master` / `onprem-main` after the sync;
   - each install reads its own `OrgVariable`;
   - the obfuscated `production-onprem` image must build and boot.
5. **Migrations:** they have all run. List anything needed at go-live, such as the indexes from §5.2 or dropping the
   unused GIN index.

---

## 6. Setup SQL (throwaway database only)

```sql
-- Switch on for one org (org-level row; integrationId must be NULL)
INSERT INTO "OrgVariable" ("orgId", "name", "value", "integrationId")
VALUES ('org_XXXXXXXXXX', 'template-visibility-enabled', 'true', NULL)
ON CONFLICT ("orgId", "name") DO UPDATE SET "value" = 'true', "integrationId" = NULL;

-- Switch off
DELETE FROM "OrgVariable" WHERE "orgId" = 'org_XXXXXXXXXX' AND "name" = 'template-visibility-enabled';

-- The production reset that runs before go-live: every template becomes Channel.
-- Soft-deleted rows are included, so a later revival can't bring an old audience back. "dateUpdated" is untouched.
UPDATE "Template"
   SET visibility = NULL, "visibilitySetBy" = NULL, "visibleToUserIds" = NULL
 WHERE visibility IS NOT NULL OR "visibilitySetBy" IS NOT NULL OR "visibleToUserIds" IS NOT NULL;
```

- The save writes an audit row with event `template.edit` and sub-event `template.visibility.update`. The row is
  seeded by the third migration.
- Audit logging is itself per org: `OrgVariable` `enable-audit-log` = `'true'`.

---

## 7. Tests in the PRs

**crm:** the committed visibility specs under `src/modules/template/**/__test__/`, plus:
- `public-api/use-cases/v2/clever-tap/__test__/get-templates-for-clevertap-visibility.use-case.spec.ts`
- `src/shared/infra/auth/permission/services/__test__/template-access-permissions.service.spec.ts`

**ai-bot:**
- `libs/templates/__test__/visibility.test.js`
- `libs/templates/__test__/audience.test.js`
- `libs/templates/__test__/visibility-filter.pg.test.js`: runs against a real Postgres. Set
  `TEMPLATE_VISIBILITY_PG_PORT` to a throwaway instance; the file header has the docker command.
- `router/templates/v1/__test__/schema.test.js`
- `shared/constants/__test__/templates-visibility-filter-options-rate-limit.test.js`
- `libs/ai/__test__/feature-permission.test.js`

**How to weigh them:**
- Apart from the pg test, these are unit tests that assert SQL as text against mocked database calls, so they can't
  show plans, row counts or races.
- Say which tests are missing for §5.1–5.3, written as Given / When / Then.

**Running them:**
- crm (pnpm): `npx vitest run <files>`. ESLint is broken repo-wide, so check with Prettier and `tsc --noEmit`, which
  has a baseline of 93 errors that already exist on master.
- ai-bot (npm, Node 22): `npx vitest run libs/templates/__test__ router/templates shared/constants/__test__`. Node 24
  fails to load the Kafka native module. Some suites already fail on main, so compare against base before blaming
  the PR.

---

## 8. Rules for this review

- **Never write to production, or to the shared dev database or Redis.** Use throwaway containers on loopback only.
  Use mock data only.
- **Execute, don't argue.** Every performance claim needs a plan or a measurement. Say what was executed, what was
  only read, and what was not checked.
- **§3.4 and §3.5 are the author's trace, not proof.** Confirm each line, correct it where it's wrong, and add any
  query it missed.
- **Inline comments** are only for what the author can change in these PRs. Put them at the cause, and at every site
  that breaks because of it. Rollout and ops notes go in the summary, not inline.
- **Pre-existing issues** (marked "pre-existing" above, or found on master/main) go in a separate list, not as PR
  blockers, unless go-live makes them reachable by more callers. Then say how.

## 9. Output

1. **Verdict:** ready for go-live / ready with fixes / not ready.
2. **Findings, most severe first.** For each finding give:
   - **Severity:**
     - `BLOCKER`: breaks, or materially slows, any org or automated caller at go-live;
     - `SHOULD-FIX`;
     - `NIT`.
   - **Location:** `repo:path:line`.
   - **Scenario:** switch state, all-Channel or restricted, caller type, viewer role.
   - **Evidence:** executed, or read.
   - **Smallest fix.**
   - **A test that would catch it.**
3. **Query performance table:** one row per query id (Q1–Q34, A0–A16, plus any you add), with route, runs per
   request, plan summary, base vs PR timing at the §4 scale, and the index you recommend.
4. **Load-test results**, including a public-API batch on a restricted template.
5. **Regression matrix for §5.1:** one result per path.
6. **Rollout recommendation for §5.4.**
7. **Pre-existing issues**, listed separately.
8. **Verified safe** (so it needn't be re-checked), and **not checked**.
