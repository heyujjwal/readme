# ORGANIZATIONAL STRUCTURE - DATABASE SCHEMA DOCUMENTATION
## Complete Column Use Cases & Business Logic

**Responsibility:** Organizations, Workspaces, User Groups, and System Migrations
**Sections:** 4, 6, 7, 27
**Total Tables:** 10 tables
**Complexity:** Medium

---

## 📋 TABLE OF CONTENTS

### Section 4: User Groups & Teams
1. [user_group](#1-user_group)
2. [user_group_membership](#2-user_group_membership)
3. [user_group_project](#3-user_group_project)
4. [user_group_tag](#4-user_group_tag)

### Section 6: Organizations
5. [organization](#5-organization)
6. [OrganizationMember](#6-organizationmember)

### Section 7: Workspaces
7. [workspace](#7-workspace)
8. [WorkspaceMember](#8-workspacemember)
9. [WorkspaceBillingEmbeddings](#9-workspacebillingembeddings)

### Section 27: System & Migrations
10. [AsyncMigrationStatus](#10-asyncmigrationstatus)

---

## 🎯 KEY USE CASES FOR ORGANIZATIONAL STRUCTURE

### High-Level Business Flows:
1. **Multi-tenant Organization Creation** - Creating isolated organizational spaces for different companies
2. **Workspace-based Project Isolation** - Separating projects within organizations by workspace (e.g., dev/staging/prod, or by team)
3. **User Group Management for Task Distribution** - Organizing annotators into teams for specific project types
4. **Cross-Organization Collaboration** - Enabling multiple organizations to work on shared projects with privacy controls
5. **Hierarchical Permission Management** - Organization → Workspace → Project → Group permission inheritance
6. **Soft Deletion with Audit Trail** - Removing members without data loss, maintaining history
7. **Context Switching** - Users switching between organizations/workspaces seamlessly
8. **Billing and Usage Tracking** - Per-workspace billing dashboards and usage monitoring
9. **Invitation-based Onboarding** - Adding users to organizations/workspaces via secure token links
10. **Database Migration Tracking** - Monitoring long-running schema changes per project

---

## SECTION 4: USER GROUPS & TEAMS

---

## 1. user_group

**Purpose:** Organizes users into teams for collaborative annotation work within workspaces
**Table Name:** `user_group`
**Business Context:** Groups allow project managers to assign batches of tasks to specific teams (e.g., "Medical Annotators", "QA Reviewers", "Spanish Speakers")

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Group ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for the group. Used in all group-related foreign keys (memberships, project assignments, tags) |
| **name** | varchar(255) | Group name | UNIQUE, NOT NULL | **Group identification** - Display name in UI (e.g., "Senior Annotators", "Medical Specialists"). Must be globally unique to prevent confusion. Used in dropdowns for task assignment |
| **description** | text | Group description | NULL | **Group purpose documentation** - Free-text explanation of group's role, required skills, or assignment criteria. Helps admins understand group purpose months later. Example: "Annotators with 5+ years medical background for radiology images" |
| **created_by_id** | integer | Creator user | FK to htx_user, SET_NULL | **Ownership tracking** - Who created this group. SET_NULL allows group to persist even if creator leaves. Used for permission checks (creator has admin rights) |
| **created_at** | timestamp | Creation timestamp | AUTO_NOW_ADD | **Audit trail** - When group was created. Used for sorting groups by age, lifecycle analysis |
| **updated_at** | timestamp | Last update timestamp | AUTO_NOW | **Change tracking** - Last modification time. Auto-updates on any change (name, description, membership changes). Helps identify stale groups |
| **workspace_id** | integer | Parent workspace | FK to workspace, CASCADE, NULL | **CRITICAL: Workspace scoping** - Groups belong to workspaces, not organizations. Enables different team structures per workspace (e.g., "Dev Team" in dev workspace, "QA Team" in staging). CASCADE delete removes group when workspace deleted |

### Business Logic Use Cases

**Primary Use Cases:**
- **Task Assignment to Teams:** Project manager assigns 500 tasks to "Spanish Annotators" group instead of individual users
- **Skill-based Routing:** Medical images automatically routed to "Medical Specialists" group
- **Quality Control Teams:** "QA Reviewers" group sees only tasks marked for review
- **Language-specific Teams:** "French Team" gets only French-language annotation tasks
- **Adjudication Panels:** "Adjudicators" group resolves disagreements between annotators

**Workflow Example:**
```
1. Admin creates group "Medical Annotators" in workspace "Production"
2. Adds 10 annotators with medical backgrounds to group
3. Links group to project "Radiology Images"
4. System automatically shows radiology tasks only to those 10 annotators
5. Can track team performance metrics separately
```

---

## 2. user_group_membership

**Purpose:** Links users to groups with specific roles and soft-delete capability
**Table Name:** `user_group_membership`
**Business Context:** Manages who belongs to which group and their role within that group. Supports soft deletion to preserve history when members leave

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Membership ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for this membership record |
| **group_id** | integer | Parent group | FK to user_group, CASCADE | **Group reference** - Which group this membership belongs to. CASCADE delete removes membership when group deleted |
| **user_id** | integer | Member user | FK to htx_user, CASCADE | **User reference** - Which user is in this group. CASCADE delete removes membership when user deleted |
| **role** | varchar(20) | Member role | CHOICES: ADMIN/ANNOTATOR/REVIEWER/ADJUDICATOR, DEFAULT 'ANNOTATOR' | **CRITICAL: Role-based permissions** - Determines what user can do within group:<br>- **ADMIN:** Manage group, add/remove members, assign tasks<br>- **ANNOTATOR:** Create annotations on assigned tasks<br>- **REVIEWER:** Review and approve/reject annotations<br>- **ADJUDICATOR:** Resolve conflicts when annotators disagree |
| **joined_at** | timestamp | Join timestamp | AUTO_NOW_ADD | **Membership start date** - When user joined group. Used for tenure tracking, "member since" display |
| **created_at** | timestamp | Creation timestamp | AUTO_NOW_ADD | **Audit trail** - When membership record created (usually same as joined_at) |
| **updated_at** | timestamp | Update timestamp | AUTO_NOW | **Change tracking** - Last modification time. Updates when role changes or membership restored |
| **is_deleted** | boolean | Soft delete flag | DEFAULT FALSE | **CRITICAL: Soft deletion** - TRUE when member removed from group. Preserves historical data (who was in group when task X was annotated). Can be restored by setting FALSE |
| **deleted_at** | timestamp | Deletion timestamp | NULL | **Deletion audit** - When membership was soft-deleted. NULL if active. Used to calculate "was member from X to Y" |
| **deleted_by_id** | integer | User who deleted | FK to htx_user, SET_NULL | **Deletion accountability** - Who removed this member. Tracks admin actions for audit purposes |

**Unique Constraint:** (group_id, user_id) - One membership record per user per group

### Business Logic Use Cases

**Primary Use Cases:**
- **Team Assignment:** Add user to "QA Team" as REVIEWER
- **Role Promotion:** Change annotator to ADMIN when they become team lead
- **Soft Removal:** Mark membership deleted when user moves to different team (preserves work history)
- **Restoration:** Reactivate soft-deleted membership when user returns
- **Audit Trail:** Track who was in group during specific time period

**Soft Deletion Workflow:**
```
1. User annotates 100 tasks as member of "Medical Team"
2. User moves to different project
3. Admin soft-deletes membership (is_deleted=TRUE, deleted_at=now)
4. User's annotations remain linked to "Medical Team"
5. Historical reports show user was team member during that period
6. Can restore membership if user returns (set is_deleted=FALSE)
```

---

## 3. user_group_project

**Purpose:** Links groups to specific projects for task routing
**Table Name:** `user_group_project`
**Business Context:** Determines which groups have access to which projects. Enables targeted task distribution

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Link ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for this group-project relationship |
| **group_id** | integer | Group reference | FK to user_group, CASCADE | **Group side of relationship** - Which group gets access. CASCADE delete removes link when group deleted |
| **project_id** | integer | Project reference | FK to project, CASCADE | **Project side of relationship** - Which project group can access. CASCADE delete removes link when project deleted |
| **created_at** | timestamp | Creation timestamp | AUTO_NOW_ADD | **Link establishment time** - When group was assigned to project |
| **updated_at** | timestamp | Update timestamp | AUTO_NOW | **Change tracking** - Last modification (rarely changes once created) |

**Unique Constraint:** (group_id, project_id) - One link per group-project pair

### Business Logic Use Cases

**Primary Use Cases:**
- **Project Assignment:** Link "Medical Annotators" group to "Radiology Project"
- **Multi-project Teams:** Link "Senior Reviewers" to multiple projects
- **Specialized Access:** Only "Spanish Speakers" group sees Spanish translation project
- **Task Queue Filtering:** Users only see tasks from projects their groups are linked to

**Assignment Workflow:**
```
1. Create project "Medical Image Classification"
2. Link "Medical Specialists" group to project
3. Link "QA Team" group to same project
4. Medical Specialists see annotation tasks
5. QA Team sees only review tasks
6. Other groups don't see this project at all
```

---

## 4. user_group_tag

**Purpose:** Associates tags with groups for categorization and filtering
**Table Name:** `user_group_tag`
**Business Context:** Enables grouping of groups by characteristics (e.g., language, skill level, certification)

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Link ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for this group-tag association |
| **group_id** | integer | Group reference | FK to user_group, CASCADE | **Group side** - Which group is being tagged. CASCADE delete removes tag link when group deleted |
| **tag_id** | integer | Tag reference | FK to projects_tag, CASCADE | **Tag side** - Which tag applies to group. CASCADE delete removes link when tag deleted |
| **created_at** | timestamp | Creation timestamp | AUTO_NOW_ADD | **Tag application time** - When tag was added to group |
| **updated_at** | timestamp | Update timestamp | AUTO_NOW | **Change tracking** - Last modification time |

**Unique Constraint:** (group_id, tag_id) - One tag per group (no duplicates)

### Business Logic Use Cases

**Primary Use Cases:**
- **Language Categorization:** Tag groups with "Spanish", "French", "Mandarin"
- **Skill Level Tagging:** "Junior", "Senior", "Expert" tags
- **Certification Tracking:** "Medical Certified", "Legal Certified" tags
- **Filtering & Search:** Find all "Senior" level groups across workspace
- **Auto-routing:** Tasks tagged "Medical" automatically go to groups tagged "Medical Certified"

**Tagging Workflow:**
```
1. Create tags: "Spanish", "Medical", "Senior"
2. Tag "Medical Team A" with: Medical, Senior
3. Tag "Medical Team B" with: Medical, Junior
4. Create task tagged "Medical", "Complex"
5. System routes to "Medical Team A" (Senior level)
6. Admin can filter "Show all Senior Medical groups"
```

---

## SECTION 6: ORGANIZATIONS

---

## 5. organization

**Purpose:** Top-level tenant isolation for multi-company deployments
**Table Name:** `organization`
**Business Context:** Highest level of data separation. Each organization is a separate company/client with complete data isolation

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Organization ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for organization across entire system |
| **title** | varchar(1000) | Organization name | NOT NULL | **Company name** - Display name in UI (e.g., "Acme Corp", "Medical Research Institute"). Up to 1000 chars to support long official names |
| **token** | varchar(256) | Invitation token | UNIQUE, NULL, DEFAULT hash | **CRITICAL: Invite link security** - UUID/hash token for invite URLs like `app.com/invite/abc123`. Unique to prevent collision. Regeneratable via reset_token() if leaked. Used when inviting new members without sending email |
| **created_by_id** | integer | Creator/owner | FK to htx_user, CASCADE | **Organization owner** - Founder user who created org. Has ultimate admin privileges. Used for "contact owner" features. CASCADE because org shouldn't exist without creator |
| **created_at** | timestamp | Creation timestamp | AUTO_NOW_ADD | **Org birthday** - When organization was established. Used for billing calculations, tenure displays, analytics |
| **updated_at** | timestamp | Last update timestamp | AUTO_NOW | **Change tracking** - Last modification to org settings (name change, etc.) |
| **contact_info** | varchar(254) | Contact email | NULL | **Support contact** - Email for billing/support inquiries. May differ from created_by email. Used for sending invoices, usage alerts |

### Business Logic Use Cases

**Primary Use Cases:**
- **Multi-tenancy:** Isolate "Company A" data completely from "Company B" data
- **Workspace Ownership:** Each org can have multiple workspaces (dev, staging, prod)
- **User Scoping:** User's active_organization_id determines which data they see
- **Billing Entity:** Invoices generated per organization
- **Invite Management:** Share invite link for new members to join org
- **Cross-org Collaboration:** Users can be members of multiple orgs, switch via active_organization_id

**Permission Hierarchy:**
```
Organization (Company)
  ├─ Workspace (Dev)
  │    ├─ Project A
  │    └─ Project B
  ├─ Workspace (Staging)
  │    └─ Project C
  └─ Workspace (Production)
       └─ Project D
```

**Critical Methods:**
- **has_user(user):** Check if user is member (includes BILLING_ACCESS hardcoded list)
- **has_permission(user):** Main permission gate - checks membership or BILLING_ACCESS
- **remove_user(user):** Soft deletes membership, updates user.active_organization_id
- **reset_token():** Regenerates invite token if compromised

**Privacy Use Case:**
```
Scenario: User from Org A viewing annotation by user from Org B
- Instead of showing real email "john@orgb.com"
- System shows codename "associate_123"
- Protects cross-organization privacy
```

---

## 6. OrganizationMember

**Purpose:** Links users to organizations with member types and soft deletion
**Table Name:** `OrganizationMember` (through table for many-to-many)
**Business Context:** Manages organization membership roster with role-based permissions

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Membership ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for this org membership |
| **user_id** | integer | Member user | FK to htx_user, CASCADE | **User reference** - Which user is member. CASCADE delete removes membership when user deleted |
| **organization_id** | integer | Parent organization | FK to organization, CASCADE | **Org reference** - Which org user belongs to. CASCADE delete removes membership when org deleted |
| **member_type** | integer | Member type | DEFAULT 1 | **Role-based access** - Permission level within org:<br>- **1 (ADMIN):** Full control, can add/remove members, create workspaces<br>- **2 (MEMBER):** Regular access, can use existing workspaces<br><br>Determines who can invite others, modify org settings |
| **created_at** | timestamp | Creation timestamp | AUTO_NOW_ADD | **Membership start** - When user joined organization. Used for "member since" displays |
| **updated_at** | timestamp | Update timestamp | AUTO_NOW | **Change tracking** - Last modification (role changes, etc.) |
| **deleted_at** | timestamp | Soft delete timestamp | NULL, INDEXED | **CRITICAL: Soft deletion** - Non-NULL when membership removed. Indexed for fast "show active members" queries. Preserves historical membership for audit trails. User's work remains but they can't access org anymore |

### Business Logic Use Cases

**Primary Use Cases:**
- **Org Membership:** Add user to organization
- **Role Management:** Promote member to admin
- **Access Revocation:** Soft-delete membership when user leaves company
- **Multi-org Users:** User can be member of multiple organizations simultaneously
- **Audit Trail:** Track who was member when specific work was done

**Soft Deletion Details:**
- **is_deleted property:** Returns TRUE if deleted_at is not NULL
- **is_owner property:** Returns TRUE if user_id == organization.created_by_id
- **soft_delete() method:**
  1. Sets deleted_at = now()
  2. Clears user's task locks
  3. Updates user.active_organization_id if this was active org
  4. Preserves all user's annotations/work

**Permission Check Workflow:**
```
1. User requests access to organization resource
2. System calls organization.has_permission(user)
3. Checks:
   - Is user in OrganizationMember with deleted_at=NULL?
   - OR is user email in BILLING_ACCESS hardcoded list?
4. If yes → grant access
5. If no → show "Access Denied"
```

**Multi-org Scenario:**
```
User: john@example.com
- Member of Org A (ADMIN, active)
- Member of Org B (MEMBER, active)
- Member of Org C (MEMBER, deleted_at='2024-01-15')

User switches org via UI → updates active_organization_id
- When active_org = A → sees all Org A data, has admin rights
- When active_org = B → sees all Org B data, member rights
- Cannot switch to Org C (soft-deleted)
```

---

## SECTION 7: WORKSPACES

---

## 7. workspace

**Purpose:** Project containers within organizations for logical separation
**Table Name:** `workspace`
**Business Context:** Second-level isolation. Organizations have multiple workspaces to separate projects by environment, team, or purpose

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Workspace ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for workspace across system |
| **title** | varchar(1000) | Workspace name | NOT NULL | **Workspace name** - Display name (e.g., "Development", "Production", "Q4 Campaign", "Medical Team"). Up to 1000 chars for descriptive names |
| **token** | varchar(256) | Invitation token | UNIQUE, NULL, DEFAULT hash | **Invite link security** - UUID/hash for workspace invite URLs. Independent of org token. Used for adding members to specific workspace without org-wide access |
| **created_by_id** | integer | Creator/owner | FK to htx_user, CASCADE | **Workspace owner** - User who created workspace. Auto-becomes admin. Often org admin but can delegate workspace creation. CASCADE ensures workspace removed if creator deleted (rare) |
| **created_at** | timestamp | Creation timestamp | AUTO_NOW_ADD | **Workspace birthday** - Creation date. Used for lifecycle tracking, sorting by age |
| **updated_at** | timestamp | Last update timestamp | AUTO_NOW | **Change tracking** - Last modification to workspace settings |
| **contact_info** | varchar(254) | Contact email | NULL | **Workspace-specific contact** - Email for questions about this specific workspace (may differ from org contact). Used for workspace-level notifications |

### Business Logic Use Cases

**Primary Use Cases:**
- **Environment Separation:** Separate "Development", "Staging", "Production" workspaces
- **Team Separation:** "Marketing Team" vs "Sales Team" workspaces in same org
- **Project Isolation:** "Q1 Projects" vs "Q2 Projects" workspaces
- **Client Separation:** Agency creates separate workspace per client within their org
- **Access Control:** User can be member of Org but only see specific workspaces

**Common Workspace Structures:**

**By Environment:**
```
Organization: Acme Corp
  ├─ Workspace: Development (test data, experiments)
  ├─ Workspace: Staging (pre-production validation)
  └─ Workspace: Production (live customer projects)
```

**By Team:**
```
Organization: Research Institute
  ├─ Workspace: Radiology Team
  ├─ Workspace: Pathology Team
  └─ Workspace: Clinical Trials Team
```

**By Client (Agency Model):**
```
Organization: Annotation Agency
  ├─ Workspace: Client A Projects
  ├─ Workspace: Client B Projects
  └─ Workspace: Internal R&D
```

**Critical Methods:**
- **has_user(user):** Check if user is workspace member (with deleted_at check)
- **has_permission(user):** Permission gate for workspace access
- **remove_user(user):** Soft-delete membership, update user.active_workspace_id
- **save() override:** Auto-creates WorkspaceMember record for creator as ADMIN

**Context Switching:**
```
User workflow:
1. User belongs to multiple workspaces
2. Clicks workspace switcher in UI
3. Updates user.active_workspace_id
4. All queries now filtered by new active_workspace_id
5. User sees only projects/tasks in that workspace
```

---

## 8. WorkspaceMember

**Purpose:** Manages workspace membership with roles and soft deletion
**Table Name:** `WorkspaceMember` (through table)
**Business Context:** Controls who can access each workspace and their permission level

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Membership ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for workspace membership |
| **user_id** | integer | Member user | FK to htx_user, CASCADE | **User reference** - Which user is workspace member. CASCADE delete removes membership when user deleted |
| **workspace_id** | integer | Parent workspace | FK to workspace, CASCADE | **Workspace reference** - Which workspace user can access. CASCADE delete removes membership when workspace deleted |
| **member_type** | integer | Member type | DEFAULT 1 | **Role-based permissions:**<br>- **1 (ADMIN):** Full workspace control, manage members, create/delete projects<br>- **2 (MEMBER):** Access projects, annotate tasks, create drafts<br><br>Inherited from WorkspaceMemberTypes enum |
| **created_at** | timestamp | Creation timestamp | AUTO_NOW_ADD | **Membership start** - When user added to workspace. Used for tenure tracking |
| **updated_at** | timestamp | Update timestamp | AUTO_NOW | **Change tracking** - Last modification (role promotions, etc.) |
| **deleted_at** | timestamp | Soft delete timestamp | NULL, INDEXED | **CRITICAL: Soft deletion** - Non-NULL when membership revoked. Indexed for fast active member queries:<br>- Preserves history (who had access when)<br>- Maintains annotation attribution<br>- Enables restoration if user returns<br>- Updates user.active_workspace_id on deletion |

### Business Logic Use Cases

**Primary Use Cases:**
- **Workspace Access:** Grant user access to specific workspace
- **Role Management:** Promote member to admin for workspace management
- **Access Revocation:** Soft-delete when user no longer needs workspace access
- **Multi-workspace Users:** User sees only workspaces they're member of
- **Audit Trail:** Track workspace access over time

**Permission Hierarchy:**
```
User has OrganizationMember record (org-level access)
  AND
User has WorkspaceMember record (workspace-level access)
  = Can access workspace's projects
```

**Soft Deletion Workflow:**
```
1. User annotates tasks in "Production Workspace"
2. User moves to different team/leaves company
3. Admin soft-deletes WorkspaceMember record
   - Sets deleted_at = now()
   - If user.active_workspace_id == this workspace:
     - System clears active_workspace_id or switches to another workspace
4. User's annotations remain linked to workspace
5. Historical reports show user was member during annotation period
6. User no longer sees workspace in switcher
7. Can restore by setting deleted_at = NULL
```

**Multi-workspace Scenario:**
```
User: jane@company.com
Organization: TechCorp (member)

Workspace Memberships:
- Development Workspace (ADMIN, active)
- Staging Workspace (MEMBER, active)
- Old Project Workspace (MEMBER, deleted_at='2024-01-01')

Current State:
- active_workspace_id = Development
- Can see Dev + Staging workspaces in switcher
- Cannot access Old Project Workspace (soft-deleted)
- All queries filtered by active_workspace_id
```

**Access Control Example:**
```
Request: GET /api/projects/

Authorization Check:
1. User authenticated? ✓
2. User.active_workspace_id = 5
3. WorkspaceMember.objects.filter(
     user=user,
     workspace_id=5,
     deleted_at__isnull=True
   ).exists() ? ✓
4. Return projects where project.workspace_id = 5
```

---

## 9. WorkspaceBillingEmbeddings

**Purpose:** Embedded billing dashboards for workspace-level usage tracking
**Table Name:** `WorkspaceBillingEmbeddings`
**Business Context:** Enables embedding external billing/analytics dashboards (e.g., Stripe, Tableau) directly in workspace UI

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Embedding ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for billing dashboard configuration |
| **workspace_id** | integer | Parent workspace | FK to workspace, CASCADE | **Workspace scope** - Which workspace this dashboard belongs to. CASCADE delete removes dashboard config when workspace deleted. One workspace can have multiple dashboards |
| **title** | varchar(50) | Dashboard title | NULL | **Display name** - Label shown in UI (e.g., "Monthly Usage", "Annotation Costs", "Team Performance"). Appears as tab/button text |
| **iframe_url** | varchar(1000) | Embed URL | NULL | **Dashboard source** - Full URL to external dashboard:<br>- Stripe billing portal: `https://billing.stripe.com/session/xyz`<br>- Looker dashboard: `https://looker.company.com/embed/dashboards/42`<br>- Tableau: `https://tableau.company.com/views/usage`<br>Embedded via iframe in workspace settings |
| **button_text** | varchar(50) | Button label text | NULL | **Call-to-action text** - Text on button to open dashboard (e.g., "View Billing", "See Usage", "Export Report") |
| **visible** | boolean | Visibility flag | DEFAULT FALSE | **Dashboard toggle** - TRUE to show dashboard in UI, FALSE to hide:<br>- Admins can enable/disable without deleting config<br>- Useful for beta testing new dashboards<br>- Hide during billing setup/testing |
| **width** | integer (PositiveIntegerField) | Iframe width in pixels | DEFAULT 1200 | **Dashboard dimensions** - Width of embedded iframe. Common values:<br>- 1200px: Standard desktop<br>- 800px: Compact view<br>- 1920px: Full-width displays |
| **height** | integer (PositiveIntegerField) | Iframe height in pixels | DEFAULT 600 | **Dashboard dimensions** - Height of embedded iframe:<br>- 600px: Standard dashboard<br>- 400px: Summary view<br>- 1000px: Detailed analytics |

### Business Logic Use Cases

**Primary Use Cases:**
- **Billing Integration:** Embed Stripe portal for subscription management
- **Usage Analytics:** Embed Looker/Tableau dashboard for annotation metrics
- **Cost Tracking:** Show real-time workspace spending
- **Performance Metrics:** Embed charts showing annotator productivity
- **Custom Reporting:** Link to workspace-specific analytics tools

**Example Configurations:**

**Stripe Billing Portal:**
```json
{
  "workspace_id": 5,
  "title": "Billing & Subscription",
  "iframe_url": "https://billing.stripe.com/portal/session_abc123",
  "button_text": "Manage Billing",
  "visible": true,
  "width": 1000,
  "height": 800
}
```

**Usage Dashboard:**
```json
{
  "workspace_id": 5,
  "title": "Annotation Metrics",
  "iframe_url": "https://looker.company.com/embed/42?workspace=5",
  "button_text": "View Analytics",
  "visible": true,
  "width": 1200,
  "height": 600
}
```

**Admin Workflow:**
```
1. Workspace admin goes to Settings → Billing
2. Clicks "Add Dashboard"
3. Enters Stripe portal URL
4. Sets button text "Manage Subscription"
5. Sets visible=true
6. Saves config
7. All workspace admins now see "Manage Subscription" button
8. Clicking opens embedded Stripe portal in iframe
9. Can toggle visible=false to hide during maintenance
```

**Multi-dashboard Setup:**
```
Workspace: Production

Dashboard 1:
- Title: "Monthly Billing"
- URL: Stripe portal
- Button: "View Invoices"
- Visible: true

Dashboard 2:
- Title: "Usage Analytics"
- URL: Tableau dashboard
- Button: "See Metrics"
- Visible: true

Dashboard 3:
- Title: "Cost Projections"
- URL: Custom analytics tool
- Button: "Forecast Costs"
- Visible: false (in testing)
```

---

## SECTION 27: SYSTEM & MIGRATIONS

---

## 10. AsyncMigrationStatus

**Purpose:** Tracks long-running database schema migrations per project
**Table Name:** `core_asyncmigrationstatus`
**Business Context:** Database migrations that would lock tables for hours are run asynchronously. This table tracks their progress

### Column Details

| Column | Type | Description | Constraints | Use Case & Business Logic Purpose |
|--------|------|-------------|-------------|-----------------------------------|
| **id** | integer (AutoField) | Migration ID | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for this migration run |
| **meta** | jsonb | Migration metadata | NOT NULL | **Migration details** - JSON blob containing:<br>- Migration parameters (batch size, timeout)<br>- Progress metrics (rows migrated, rows remaining)<br>- Error details if failed<br>- Start/end timestamps per phase<br>- Custom migration-specific data<br><br>Example: `{"batch_size": 1000, "rows_migrated": 45000, "rows_total": 100000}` |
| **project_id** | integer | Related project | FK to project, CASCADE, NULL | **Project scope** - Which project this migration applies to:<br>- NULL: System-wide migration (affects all projects)<br>- Non-NULL: Project-specific migration (e.g., rebuilding project annotations index)<br><br>CASCADE delete removes migration status when project deleted |
| **name** | text | Migration name | NOT NULL | **Migration identifier** - Human-readable name describing what migration does:<br>- "add_task_overlap_index_2024_01"<br>- "migrate_annotations_to_jsonb"<br>- "rebuild_project_summary_cache"<br><br>Used for tracking which migrations ran, debugging |
| **status** | varchar(100) | Migration status | CHOICES: STARTED/IN PROGRESS/FINISHED/ERROR, NOT NULL | **CRITICAL: Migration state tracking:**<br>- **STARTED:** Migration job created, not yet running<br>- **IN PROGRESS:** Currently executing, worker processing<br>- **FINISHED:** Completed successfully, safe to use new schema<br>- **ERROR:** Failed, needs investigation/retry<br><br>UI shows status to admins, blocks schema changes if IN PROGRESS |
| **created_at** | timestamp | Creation timestamp | AUTO_NOW_ADD | **Migration start time** - When migration job was created. Used to calculate total duration |
| **updated_at** | timestamp | Update timestamp | AUTO_NOW | **Last progress update** - Auto-updates when worker reports progress. Used to detect stuck migrations (if not updated in 10+ minutes) |

### Business Logic Use Cases

**Primary Use Cases:**
- **Safe Schema Changes:** Run database migrations without locking production tables
- **Progress Monitoring:** Admins see real-time migration progress
- **Error Recovery:** Track failed migrations for retry
- **Downtime Avoidance:** Asynchronous migrations prevent site outages
- **Project-specific Updates:** Rebuild indexes for large projects without affecting others

**Migration Workflow:**

**System-wide Migration:**
```
Scenario: Add new index to task table (100M rows)

1. Developer creates migration script
2. System creates AsyncMigrationStatus:
   - name: "add_task_inner_id_index_2024_03"
   - status: "STARTED"
   - project_id: NULL (affects all)
   - meta: {"table": "task", "index": "idx_inner_id"}

3. Background worker picks up job:
   - status → "IN PROGRESS"
   - meta: {"batch_size": 10000, "rows_processed": 0}

4. Worker processes in batches:
   - Batch 1: 10k rows indexed
     - meta: {"rows_processed": 10000, "rows_total": 100000000}
   - Batch 2: 10k rows indexed
     - meta: {"rows_processed": 20000, ...}
   - ... continues ...

5. Completion:
   - status → "FINISHED"
   - meta: {"rows_processed": 100000000, "duration_seconds": 3600}

6. Admin sees: "Migration completed in 1 hour"
```

**Project-specific Migration:**
```
Scenario: Rebuild annotation cache for large project

1. Project "Medical Images" has 1M annotations
2. Cache rebuild needed after schema change
3. System creates AsyncMigrationStatus:
   - name: "rebuild_annotation_cache_project_42"
   - status: "STARTED"
   - project_id: 42
   - meta: {"total_annotations": 1000000}

4. Worker processes project 42 only:
   - Doesn't affect other projects
   - Updates meta with progress every 1000 annotations

5. If error occurs:
   - status → "ERROR"
   - meta: {"error": "Timeout at annotation 500000", "can_retry": true}

6. Admin can retry from last successful point
```

**Admin Dashboard View:**
```
Migrations in Progress:

1. "add_task_overlap_index_2024_03"
   Status: IN PROGRESS (45% complete)
   Started: 2024-03-15 14:30:00
   Elapsed: 27 minutes
   ETA: 33 minutes

2. "rebuild_project_summary_project_18"
   Status: IN PROGRESS (80% complete)
   Project: Radiology Dataset
   Started: 2024-03-15 15:00:00
   Elapsed: 12 minutes
   ETA: 3 minutes

Recent Completions:

3. "migrate_predictions_to_v2_schema"
   Status: FINISHED
   Duration: 2 hours 14 minutes
   Completed: 2024-03-14 09:45:00
```

**Error Handling:**
```
Migration Failed Scenario:

1. Migration starts: "add_complex_index"
2. Worker processes 60% of data
3. Database connection timeout occurs
4. System records:
   - status: "ERROR"
   - meta: {
       "error": "Connection timeout",
       "last_successful_batch": 600000,
       "rows_remaining": 400000,
       "can_resume": true
     }
5. Admin alerted via email/Slack
6. Admin investigates, increases timeout config
7. Admin clicks "Retry Migration"
8. Worker resumes from batch 600000
9. Completes successfully
```

**Migration Meta Examples:**

**Progress Tracking:**
```json
{
  "batch_size": 1000,
  "total_rows": 1000000,
  "rows_processed": 450000,
  "percentage_complete": 45.0,
  "batches_processed": 450,
  "batches_remaining": 550,
  "current_batch_start_time": "2024-03-15T14:55:23Z",
  "estimated_completion_time": "2024-03-15T15:23:00Z"
}
```

**Error Details:**
```json
{
  "error_type": "DatabaseTimeout",
  "error_message": "Connection lost after 30 minutes",
  "error_timestamp": "2024-03-15T15:00:00Z",
  "last_successful_batch": 600000,
  "failed_batch": 600001,
  "can_retry": true,
  "suggested_fix": "Increase database timeout to 60 minutes"
}
```

---

## 🔄 BUSINESS WORKFLOWS & DATA FLOWS

### Workflow 1: Creating New Organization with Workspaces

```
Step 1: User Signs Up
- User creates account (htx_user record)
- User.email = "admin@company.com"

Step 2: Create Organization
- organization.create(
    title="Acme Corp",
    created_by=user,
    contact_info="billing@acme.com"
  )
- Auto-generates organization.token for invite links
- Creates OrganizationMember(user=user, org=org, member_type=ADMIN)

Step 3: User Creates Workspaces
- workspace.create(title="Development", created_by=user)
  - Auto-creates WorkspaceMember(user=user, workspace=dev, member_type=ADMIN)
  - Links workspace to organization

- workspace.create(title="Production", created_by=user)
  - Auto-creates WorkspaceMember for creator

Step 4: User's Active Context
- user.active_organization_id = organization.id
- user.active_workspace_id = development_workspace.id
- User now sees only Development workspace data
```

### Workflow 2: Inviting Team Members to Workspace

```
Step 1: Admin Generates Invite Link
- Admin in workspace "Production"
- Clicks "Invite Members"
- System generates: https://app.com/invite/workspace/{workspace.token}

Step 2: New User Accepts Invite
- New user clicks link
- System validates workspace.token
- Creates htx_user account if needed
- Creates WorkspaceMember(user=new_user, workspace=prod, member_type=MEMBER)

Step 3: Admin Creates User Groups
- group.create(name="QA Reviewers", workspace=prod)
- group_membership.create(user=new_user, group=qa_reviewers, role=REVIEWER)

Step 4: Admin Assigns Group to Projects
- user_group_project.create(group=qa_reviewers, project=medical_project)
- New user now sees medical project tasks
- Can perform review actions only (based on REVIEWER role)
```

### Workflow 3: User Switching Between Workspaces

```
Current State:
- user.active_workspace_id = 5 (Development)
- User sees 10 projects in Development

User Clicks Workspace Switcher:
1. UI shows workspaces where WorkspaceMember(user=user, deleted_at=NULL)
2. User selects "Production" workspace (id=10)

Backend Updates:
3. user.active_workspace_id = 10
4. user.save()

UI Refreshes:
5. All queries now filtered by workspace_id=10
6. User sees Production projects only
7. Sidebar updates with Production workspace name
8. Billing dashboard shows Production usage
```

### Workflow 4: Soft-Deleting User from Organization

```
Scenario: Employee leaves company

Step 1: Admin Removes User
- Admin finds user in member list
- Clicks "Remove from Organization"

Step 2: System Soft-Deletes
- OrganizationMember.soft_delete(deleted_by=admin)
  - Sets deleted_at = now()
  - Sets deleted_by_id = admin.id
  - Clears user's task locks in this org

- If user.active_organization_id == this_org:
  - System sets user.active_organization_id = NULL
  - OR switches to another org if user is multi-org member

Step 3: Cascade Effects
- User can no longer access organization
- User's annotations remain linked (historical data)
- Reports show user as "Former Member" with tenure dates
- User can still access other organizations they're member of

Step 4: Restoration (if user returns)
- Admin clicks "Restore Member"
- Sets deleted_at = NULL
- User regains full access
```

### Workflow 5: Running Async Migration

```
Scenario: Adding new index to large project

Step 1: Developer Triggers Migration
- Deploy new code with migration script
- Migration creates AsyncMigrationStatus:
    name: "add_annotation_index_project_42"
    status: "STARTED"
    project_id: 42
    meta: {}

Step 2: Background Worker Picks Up Job
- Worker queries AsyncMigrationStatus where status="STARTED"
- Claims job by setting status="IN PROGRESS"
- Begins processing in batches

Step 3: Progress Updates
- Every 1000 rows processed:
    status: "IN PROGRESS"
    updated_at: now()
    meta: {"rows_processed": 10000, "rows_total": 500000}

- Admin dashboard polls updated_at to show live progress

Step 4: Completion
- Worker finishes all batches
- Updates:
    status: "FINISHED"
    meta: {"rows_processed": 500000, "duration_seconds": 1800}

- System notification: "Migration completed successfully"

Step 5: Error Handling (if failure)
- If worker crashes:
    status: "ERROR"
    meta: {"error": "Worker timeout", "last_batch": 300000}

- Admin investigates, retries migration
- Worker resumes from last_batch
```

---

## 🔍 CRITICAL BUSINESS RULES

### Multi-Tenancy Rules

1. **Data Isolation:**
   - Users in Org A NEVER see data from Org B
   - Exception: Codename shown for cross-org collaboration (privacy)

2. **Permission Hierarchy:**
   ```
   Organization Member → Can potentially access org
   +
   Workspace Member → Can access workspace
   +
   Project Member → Can work on project
   +
   Group Member → Gets assigned specific tasks
   ```

3. **Active Context:**
   - User must have active_organization_id and active_workspace_id set
   - All queries filtered by these IDs
   - Switching updates these IDs, changes entire view

### Soft Deletion Rules

1. **When to Soft Delete:**
   - User leaves organization/workspace/group
   - Want to preserve historical attribution
   - May need to restore access later

2. **What Soft Delete Does:**
   - Sets deleted_at timestamp
   - Records deleted_by user (accountability)
   - Clears task locks (prevent blocking)
   - Updates active_* IDs if needed
   - Preserves all work done by user

3. **What Soft Delete Doesn't Do:**
   - Doesn't remove user's annotations
   - Doesn't break foreign keys
   - Doesn't affect other memberships
   - Doesn't delete user account

### Invitation Token Rules

1. **Token Security:**
   - UUIDs prevent guessing
   - Unique per organization/workspace
   - Can be regenerated if leaked
   - Should expire after use (not enforced in schema)

2. **Token Usage:**
   - Share via secure channel (email, Slack)
   - One token per org, one per workspace
   - Anyone with token can join (if validated)
   - Should implement rate limiting (not in schema)

### Migration Rules

1. **Status Transitions:**
   ```
   STARTED → IN PROGRESS → FINISHED (success)
                       ↓
                     ERROR (failure) → can retry → IN PROGRESS
   ```

2. **Progress Tracking:**
   - Update meta every N rows (e.g., 1000)
   - Update updated_at to show worker alive
   - Include ETA calculations in meta
   - Store error details for debugging

3. **Project-specific vs System-wide:**
   - project_id=NULL: Affects all projects
   - project_id=X: Affects only project X
   - Multiple migrations can run in parallel if different projects

---

## 📊 PERFORMANCE & INDEXING NOTES

### Critical Indexes

```sql
-- Organization membership lookups (hot path)
CREATE INDEX idx_org_member_active ON OrganizationMember(user_id, organization_id)
  WHERE deleted_at IS NULL;

-- Workspace membership lookups (hot path)
CREATE INDEX idx_workspace_member_active ON WorkspaceMember(user_id, workspace_id)
  WHERE deleted_at IS NULL;

-- Group membership with soft delete
CREATE INDEX idx_group_member_active ON user_group_membership(user_id, group_id)
  WHERE is_deleted = FALSE;

-- Active migrations monitoring
CREATE INDEX idx_migration_status ON AsyncMigrationStatus(status, updated_at)
  WHERE status IN ('STARTED', 'IN PROGRESS');
```

### Query Patterns

**Most Common Query (runs on every API request):**
```python
# Check if user can access workspace
WorkspaceMember.objects.filter(
    user_id=user.id,
    workspace_id=user.active_workspace_id,
    deleted_at__isnull=True
).exists()

# Cached in request context to avoid N+1
```

**Group-based Task Assignment:**
```python
# Find tasks for user based on group memberships
user_groups = user_group_membership.objects.filter(
    user=user,
    is_deleted=False
).values_list('group_id', flat=True)

projects = user_group_project.objects.filter(
    group_id__in=user_groups
).values_list('project_id', flat=True)

tasks = Task.objects.filter(
    project_id__in=projects,
    project__workspace_id=user.active_workspace_id
)
```

---

## ✅ VALIDATION RULES

### Business Validation

1. **Organization:**
   - Title cannot be empty
   - Token must be unique across system
   - Creator must be valid user
   - Contact email must be valid format

2. **Workspace:**
   - Must belong to organization
   - Title cannot be empty
   - Token unique across system
   - Creator must be org member

3. **Groups:**
   - Name unique across system (consider scoping to workspace?)
   - Must belong to workspace
   - Creator must be workspace member

4. **Memberships:**
   - User cannot be added twice to same org/workspace/group (unless soft-deleted)
   - member_type must be valid choice
   - Cannot soft-delete last admin (implement in code)

5. **Migrations:**
   - Name should follow convention: {action}_{table}_{date}
   - Status must be valid choice
   - Meta must be valid JSON
   - Cannot have multiple migrations with same name IN PROGRESS

---

## 🚨 COMMON PITFALLS & GOTCHAS

### 1. Forgetting Soft Delete Checks
```python
# ❌ WRONG - includes deleted members
members = OrganizationMember.objects.filter(organization=org)

# ✅ CORRECT - exclude soft-deleted
members = OrganizationMember.objects.filter(
    organization=org,
    deleted_at__isnull=True
)
```

### 2. Active Context Not Set
```python
# ❌ WRONG - user.active_workspace_id is None
projects = Project.objects.filter(workspace_id=user.active_workspace_id)
# Returns empty set, user sees nothing

# ✅ CORRECT - check and set default
if not user.active_workspace_id:
    user.active_workspace_id = user.workspaces.first().id
    user.save()
```

### 3. Cross-Org Privacy Leaks
```python
# ❌ WRONG - shows real email
annotation.completed_by.email

# ✅ CORRECT - check org context
if annotation.completed_by.active_organization_id != request.user.active_organization_id:
    return annotation.completed_by.codename  # "associate_123"
else:
    return annotation.completed_by.email
```

### 4. Migration Status Stale
```python
# ❌ WRONG - migration shows "IN PROGRESS" for 2 hours
# Worker crashed but status not updated

# ✅ CORRECT - implement heartbeat check
stale_migrations = AsyncMigrationStatus.objects.filter(
    status='IN PROGRESS',
    updated_at__lt=now() - timedelta(minutes=10)
)
# Alert admins, mark as ERROR
```

### 5. Cascade Delete Confusion
```python
# Workspace deleted → CASCADE deletes WorkspaceMember
# User's active_workspace_id still points to deleted workspace!

# ✅ CORRECT - implement pre_delete signal
@receiver(pre_delete, sender=Workspace)
def clear_active_workspace(sender, instance, **kwargs):
    User.objects.filter(
        active_workspace_id=instance.id
    ).update(active_workspace_id=None)
```

---

## 📈 SCALABILITY CONSIDERATIONS

### Horizontal Scaling

1. **Sharding by Organization:**
   - Each organization's data on separate database shard
   - organization_id as shard key
   - Enables independent scaling per tenant

2. **Read Replicas:**
   - Membership checks read from replica
   - Writes go to primary
   - Heavy read:write ratio (100:1)

3. **Caching Strategy:**
   ```python
   # Cache user's org/workspace memberships
   cache_key = f"user:{user.id}:memberships"
   memberships = cache.get(cache_key)
   if not memberships:
       memberships = fetch_from_db()
       cache.set(cache_key, memberships, timeout=300)  # 5 min
   ```

### Large Organizations

**Problem:** Organization with 10,000 members
- Listing members page slow
- Soft delete queries expensive

**Solution:**
```python
# Paginate member lists
members = OrganizationMember.objects.filter(
    organization=org,
    deleted_at__isnull=True
).select_related('user').order_by('-created_at')[:50]

# Add compound index
CREATE INDEX idx_org_members_list ON OrganizationMember(
    organization_id, deleted_at, created_at DESC
);
```

---

## 🔗 INTEGRATION POINTS

### External Systems

1. **Billing Systems (Stripe, etc.):**
   - WorkspaceBillingEmbeddings stores iframe URLs
   - Webhook callbacks update organization/workspace usage
   - organization_id or workspace_id in Stripe metadata

2. **Analytics (Looker, Tableau):**
   - Embed dashboards per workspace
   - Pass workspace_id in URL parameters
   - SSO for seamless authentication

3. **Identity Providers (Okta, Auth0):**
   - Map IDP groups to user_group
   - Sync membership on login
   - Use organization.token for SAML relay state

4. **Notification Services:**
   - Send alerts when AsyncMigrationStatus = ERROR
   - Notify org admins when workspace created
   - Email digest of workspace activity

---

## 🎯 TESTING CHECKLIST

### Unit Tests

- [ ] Organization creation with auto-member
- [ ] Workspace creation with auto-member
- [ ] Group creation and membership
- [ ] Soft delete preserves data
- [ ] Restore from soft delete
- [ ] Token uniqueness
- [ ] Permission checks with deleted members
- [ ] Active context switching
- [ ] Migration status transitions
- [ ] Cross-org privacy (codename display)

### Integration Tests

- [ ] Full onboarding flow (org → workspace → group → project)
- [ ] Multi-workspace user switching
- [ ] Invite link acceptance
- [ ] Soft delete cascading effects
- [ ] Migration progress tracking
- [ ] Billing dashboard embed
- [ ] Group-based task assignment
- [ ] Cross-org annotation attribution

### Performance Tests

- [ ] 10,000 org members list load time < 200ms
- [ ] Active workspace check load time < 10ms
- [ ] Group membership query with 100 groups < 50ms
- [ ] Migration status check < 5ms
- [ ] Soft delete check with 1M records < 100ms

---




