# 🧩 SECTION 1: CORE USER MANAGEMENT

# USER MANAGEMENT – DATABASE SCHEMA DOCUMENTATION

## Complete Column Use Cases & Business Logic

**Responsibility:** Authentication, Identity Management, Profiles, and Access Initialization
**Sections:** 1
**Total Tables:** 4 tables
**Complexity:** High

---

## 📋 TABLE OF CONTENTS

### Section 1: Core User Management

1. [htx_user](#1-htx_user)
2. [users_userprofile](#2-users_userprofile)
3. [users_userbankdetails](#3-users_userbankdetails)
4. [users_magiclink](#4-users_magiclink)

---

## 🎯 KEY USE CASES FOR USER MANAGEMENT

### High-Level Business Flows:

1. **User Authentication & Authorization** – central login and permission management
2. **User Lifecycle Tracking** – registration, onboarding, deactivation
3. **Identity Localization** – country & language for multilingual support
4. **Payment Enablement** – bank details for global payouts
5. **Passwordless Access** – secure magic-link login system
6. **Cross-Module Dependency** – foundation for organizations, workspaces, invitations, and projects
7. **Activity Auditing** – track login and annotation activity
8. **Compliance Data Capture** – retain legal information for KYC and payments

---

## SECTION 1: CORE USER MANAGEMENT

---

## 1. htx_user

**Purpose:**
Primary user identity and authentication table integrated with Django’s `AbstractBaseUser`.

**Table Name:** `htx_user`
**Business Context:**
Acts as the root for every other module (user profiles, organizations, workspaces, payments, and invitations).

### Column Details

| Column                                | Type                | Description                                    | Constraints       | Use Case & Business Logic Purpose      |
| ------------------------------------- | ------------------- | ---------------------------------------------- | ----------------- | -------------------------------------- |
| **id**                                | integer (AutoField) | Primary User ID                                | PRIMARY KEY       | Unique identifier for each user record |
| **username**                          | varchar(256)        | Public/internal username                       | INDEXED           | Used for display and search            |
| **email**                             | varchar(256)        | Login email address                            | UNIQUE, NOT NULL  | Primary login credential               |
| **codename**                          | varchar(256)        | Auto-generated user alias (e.g., associate_42) | DEFAULT NULL      | Used for privacy in multi-org views    |
| **first_name** / **last_name**        | varchar(256)        | Personal names                                 | NULL              | Used in UI displays                    |
| **country**                           | varchar(256)        | Residence country                              | NULL              | Localization and legal use             |
| **native_lang** / **additional_lang** | varchar(256)        | Languages                                      | NULL              | Used for language-specific projects    |
| **phone**                             | varchar(256)        | Contact number                                 | NULL              | Optional                               |
| **avatar**                            | image               | Profile photo                                  | NULL              | Display avatar in dashboard            |
| **is_staff**                          | boolean             | Admin flag                                     | DEFAULT FALSE     | Allows Django admin login              |
| **is_active**                         | boolean             | Active status                                  | DEFAULT TRUE      | Soft disable instead of delete         |
| **onboarded**                         | boolean             | Onboarding complete                            | DEFAULT FALSE     | Tracks user progress                   |
| **date_joined**                       | timestamp           | Join time                                      | AUTO_NOW_ADD      | Used in reports                        |
| **activity_at**                       | timestamp           | Last annotation activity                       | AUTO_NOW          | Updated per task event                 |
| **last_activity**                     | timestamp           | Last login or action                           | AUTO_NOW          | Used for engagement metrics            |
| **active_workspace_id**               | integer             | Active workspace                               | FK → workspace    | Current workspace context              |
| **active_organization_id**            | integer             | Active organization                            | FK → organization | Tenant context for data scoping        |
| **allow_newsletters**                 | boolean             | Newsletter opt-in                              | NULL              | GDPR compliance flag                   |

### Business Logic Use Cases

**Primary Use Cases:**

* Authentication and token generation
* Organization and workspace linking
* User activity tracking
* Codename auto-generation for privacy
* Email normalization for duplicate prevention

**Workflow Example:**

```
1. User registers → creates htx_user record  
2. post_save signal fires → creates Auth Token  
3. Codename auto-generates as associate_<id>  
4. User logs in and joins organization workspace  
```

---

## 2. users_userprofile

**Purpose:**
Extended user profile for personal and demographic information.

**Table Name:** `users_userprofile`
**Business Context:**
Stores personal metadata for HR, KYC, and diversity reporting.

### Column Details

| Column                | Type         | Description       | Constraints                 | Use Case & Business Logic Purpose |
| --------------------- | ------------ | ----------------- | --------------------------- | --------------------------------- |
| **id**                | integer      | Profile ID        | PRIMARY KEY                 | Unique record                     |
| **user_id**           | integer      | Linked user       | OneToOne → htx_user         | Profile owner                     |
| **birth_date**        | date         | Date of birth     | NULL                        | For age verification              |
| **gender**            | varchar(10)  | Gender            | Choices (male/female/other) | Optional demographics             |
| **sexual_preference** | varchar(50)  | Sexual preference | NULL                        | Optional                          |
| **religion**          | varchar(50)  | Religion          | NULL                        | Optional                          |
| **ethnicity**         | varchar(50)  | Ethnicity         | NULL                        | Used for D&I analysis             |
| **country_of_origin** | varchar(100) | Birth country     | NULL                        | Localization support              |
| **company_name**      | varchar(255) | Employer          | NOT NULL                    | Identifies company link           |
| **role**              | varchar(255) | Job title         | NOT NULL                    | Determines responsibility         |

### Business Logic Use Cases

* Enhances user data for compliance and reporting
* Used in analytics and filtering (e.g., role, language, region)
* Populated during onboarding forms

**Workflow Example:**

```
1. User joins organization → Profile created automatically  
2. Fills out company_name and role  
3. Data used in dashboard filtering and HR reports  
```

---

## 3. users_userbankdetails

**Purpose:**
Stores global banking and payment information for users.

**Table Name:** `users_userbankdetails`
**Business Context:**
Used by payment modules for contractor and vendor payouts.

### Column Details

| Column                  | Type         | Description    | Constraints         | Use Case & Business Logic Purpose |
| ----------------------- | ------------ | -------------- | ------------------- | --------------------------------- |
| **id**                  | integer      | Record ID      | PRIMARY KEY         | Unique entry                      |
| **user_id**             | integer      | Linked user    | OneToOne → htx_user | Owner of bank details             |
| **account_holder_name** | varchar(100) | Account name   | REQUIRED            | Verification of ownership         |
| **account_number**      | varchar(34)  | Account / IBAN | REQUIRED            | Wire transfer identifier          |
| **bank_name**           | varchar(100) | Bank name      | REQUIRED            | Displayed in payment records      |
| **swift_code**          | varchar(11)  | SWIFT/BIC      | NULL                | International payments            |
| **country**             | varchar(100) | Bank country   | REQUIRED            | Determines transfer network       |
| **routing_number**      | varchar(9)   | US Routing     | NULL                | ACH payments                      |
| **sort_code**           | varchar(6)   | UK Sort Code   | NULL                | Domestic UK payouts               |
| **bsb_number**          | varchar(6)   | AU BSB         | NULL                | Australian payouts                |
| **ifsc_code**           | varchar(11)  | Indian IFSC    | NULL                | India bank transfer               |

### Business Logic Use Cases

* Ensures secure storage of payout information
* Validates account formats for each region
* Referenced by payment_rate and invoice modules

**Workflow Example:**

```
1. User updates bank details in profile  
2. System validates SWIFT/IFSC codes  
3. Finance team uses these for scheduled payouts  
```

---

## 4. users_magiclink

**Purpose:**
Implements password-less authentication through secure tokens.

**Table Name:** `users_magiclink`
**Business Context:**
Used for temporary login links in email-based sign-in flows.

### Column Details

| Column      | Type         | Description       | Constraints   | Use Case & Business Logic Purpose |
| ----------- | ------------ | ----------------- | ------------- | --------------------------------- |
| **id**      | integer      | Record ID         | PRIMARY KEY   | Unique magic-link entry           |
| **user_id** | integer      | Linked user       | FK → htx_user | Token owner                       |
| **token**   | varchar(255) | Unique UUID token | UNIQUE        | Used for one-time login URL       |

### Business Logic Use Cases

* Generates temporary login links via email
* Token expires after first use
* Improves user experience for external contractors

**Workflow Example:**

```
1. User clicks “Send Magic Link” in login form  
2. System creates unique UUID token record  
3. Email sent → “Click to Sign In”  
4. User clicks link → system verifies token → logs user in → deletes token  
```

---

## ✅ SECTION SUMMARY

| Table                   | Purpose                                     | Relationships                               | Key Usage                             |
| ----------------------- | ------------------------------------------- | ------------------------------------------- | ------------------------------------- |
| `htx_user`              | Core user authentication and identity table | Organizations, Workspaces, Profiles, Tokens | Login, permissions, activity tracking |
| `users_userprofile`     | Personal and demographic details            | 1-1 with User                               | Onboarding, HR analytics              |
| `users_userbankdetails` | Bank and payment data                       | 1-1 with User                               | Payroll & payouts                     |
| `users_magiclink`       | One-time token login records                | M-1 with User                               | Passwordless authentication           |




 **Section 2: User Profiles & Extended Info**,


# USER PROFILES & EXTENDED INFO – DATABASE SCHEMA DOCUMENTATION

## Complete Column Use Cases & Business Logic

**Responsibility:** Extended user personalization, preferences, and detailed profile storage
**Sections:** 2
**Total Tables:** 1 table
**Complexity:** Medium

---

## 📋 TABLE OF CONTENTS

### Section 2: User Profiles & Extended Info

1. [users_newuserprofile](#1-users_newuserprofile)


## 🎯 KEY USE CASES FOR USER EXTENDED PROFILES

### High-Level Business Flows:

1. **Personalized User Preferences** – Store and apply user-specific UI settings.
2. **Extended Metadata Storage** – Save job titles and optional demographic details.
3. **Profile Integration** – Link additional information directly to authenticated users.
4. **Localization Support** – Manage preferred languages for the platform interface.
5. **User Experience Optimization** – Automatically render theme and language on login.

---

## SECTION 2: USER PROFILES & EXTENDED INFO

---

## 1. users_newuserprofile

**Purpose:**
Extends the base user model with additional personalization options such as UI theme, language preference, and professional designation.

**Table Name:** `users_newuserprofile`
**Business Context:**
Enables user-centric customization and provides extended metadata for dashboards, analytics, and UI rendering.

---

### Column Details

| Column                 | Type                | Description       | Constraints                          | Use Case & Business Logic Purpose                                          |
| ---------------------- | ------------------- | ----------------- | ------------------------------------ | -------------------------------------------------------------------------- |
| **id**                 | integer (AutoField) | Profile ID        | PRIMARY KEY, AUTO_INCREMENT          | Unique identifier for the extended user profile                            |
| **user_id**            | integer             | Linked user       | OneToOne → `htx_user`                | Ensures one-to-one relation with main user record                          |
| **designation**        | varchar(256)        | Job title or role | NULL                                 | Identifies user’s professional designation (e.g., “Annotator”, “Reviewer”) |
| **birth_date**         | date                | Date of birth     | NULL                                 | Used for optional demographic data                                         |
| **gender**             | varchar(10)         | Gender            | Choices: male/female/other           | Optional for analytics or compliance                                       |
| **theme**              | varchar(10)         | UI display mode   | Choices: light/dark, DEFAULT 'light' | Stores preferred interface mode                                            |
| **preferred_language** | varchar(10)         | UI language       | NULL                                 | Determines language shown in application UI                                |

---

### Business Logic Use Cases

**Primary Use Cases:**

* **User Experience Customization:** Retains theme and language selections across sessions.
* **Professional Context Storage:** Stores job title or function for internal categorization.
* **Analytics Enhancement:** Allows administrators to segment users by language or role.
* **Smooth Onboarding:** Simplifies preference setup during account creation.

---

### Relationships

* 1️⃣ **One-to-One → htx_user**
* 🔗 Referenced by UI session and preference management systems.

---

### Workflow Example

```
1. User logs in for the first time → system creates default NewUserProfile.
2. Default theme = 'light'; preferred_language empty.
3. User switches to dark mode and selects English.
4. Preferences persist; on next login UI loads dark theme automatically.
```

---

## ✅ SECTION SUMMARY

| Table                  | Purpose                                                | Relationships | Key Usage                                    |
| ---------------------- | ------------------------------------------------------ | ------------- | -------------------------------------------- |
| `users_newuserprofile` | Stores extended user preferences and professional info | 1-1 with User | UI theme, language, and role personalization |





# 💰 SECTION 3: USER PAYMENTS & COMPENSATION



# USER PAYMENTS & COMPENSATION – DATABASE SCHEMA DOCUMENTATION

## Complete Column Use Cases & Business Logic

**Responsibility:** User-level compensation, currency standardization, and payment configuration
**Sections:** 3
**Total Tables:** 2 tables
**Complexity:** Medium

---

## 📋 TABLE OF CONTENTS

### Section 3: User Payments & Compensation

1. [payment_rate](#1-payment_rate)
2. [currency](#2-currency)

---

## 🎯 KEY USE CASES FOR PAYMENTS & COMPENSATION

### High-Level Business Flows:

1. **Define User Payment Structure** – Store hourly or per-label payment rates.
2. **Global Currency Mapping** – Manage currency codes and names for financial consistency.
3. **Automated Payroll Integration** – Connect user payment rates with payout systems.
4. **Task-based Payment Models** – Support flexible compensation (per task, per hour, per label).
5. **Data Auditing & Reporting** – Provide structured financial records per user for compliance.

---

## SECTION 3: USER PAYMENTS & COMPENSATION

---

## 1. payment_rate

**Purpose:**
Defines how much a user is compensated and in which currency.

**Table Name:** `payment_rate`
**Business Context:**
Serves as the foundation for payroll and billing logic. Each user has an associated rate that determines earnings per task or per hour.

---

### Column Details

| Column          | Type                | Description       | Constraints                   | Use Case & Business Logic Purpose                |
| --------------- | ------------------- | ----------------- | ----------------------------- | ------------------------------------------------ |
| **id**          | integer (AutoField) | Payment rate ID   | PRIMARY KEY, AUTO_INCREMENT   | Unique record identifier                         |
| **user_id**     | integer             | Linked user       | OneToOne → `htx_user`         | Each user has a single active rate configuration |
| **global_rate** | decimal(10,2)       | Compensation rate | REQUIRED                      | Defines numeric rate (e.g., 10.50 = $10.50/hour) |
| **currency**    | varchar(3)          | Currency code     | Choices: USD/EUR/GBP/IND      | Specifies payment currency per user              |
| **rate_type**   | varchar(10)         | Type of rate      | Choices: per_hour / per_label | Indicates how compensation is calculated         |

---

### Business Logic Use Cases

**Primary Use Cases:**

* **Hourly Rate Payments:** Used for employees or hourly-based contracts.
* **Per-Label Payment System:** Used for annotation tasks, where each labeled item earns a fixed amount.
* **Global Rate Definition:** Administrators configure a base rate per user, converted via `Currency` table as needed.
* **Automated Invoicing:** Used by payroll systems to compute monthly or project-level payouts.

---

### Relationships

* 1️⃣ **One-to-One → htx_user** (each user has one defined rate)
* 🔗 **Uses → Currency (by code)** for standardized currency representation.

---

### Workflow Example

```
1. Admin sets payment_rate(user=John, rate_type='per_hour', global_rate=15.00, currency='USD').
2. John logs 20 hours of task activity.
3. Payroll system calculates payout = 15 * 20 = 300 USD.
4. Record logged for monthly invoice and stored in accounting logs.
```

---

## 2. currency

**Purpose:**
Stores global currency definitions used across user payment and financial modules.

**Table Name:** `currency`
**Business Context:**
Defines standardized currency codes and names for system-wide use in transactions, reports, and user-specific rates.

---

### Column Details

| Column   | Type                | Description            | Constraints                 | Use Case & Business Logic Purpose                              |
| -------- | ------------------- | ---------------------- | --------------------------- | -------------------------------------------------------------- |
| **id**   | integer (AutoField) | Currency ID            | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for each currency record                     |
| **code** | varchar(3)          | ISO 4217 currency code | UNIQUE, DEFAULT “USD”       | Ensures consistent use of standard currency codes              |
| **name** | varchar(50)         | Currency full name     | REQUIRED                    | Used for UI displays and reporting (e.g., “US Dollar”, “Euro”) |

---

### Business Logic Use Cases

**Primary Use Cases:**

* **Standardization:** Ensures consistent financial reporting and avoids mismatched currency strings.
* **Dropdown Population:** Powers currency selection fields across payment and invoice UIs.
* **Cross-Rate Integration:** Can be extended to hold exchange rate data for multi-currency billing.

---

### Relationships

* 🔗 **Referenced by:** `payment_rate` table (`currency` field)
* Used system-wide for monetary formatting and validation.

---

### Workflow Example

```
1. Admin adds Currency(code='INR', name='Indian Rupee').
2. User’s payment_rate(currency='INR', global_rate=500.00, rate_type='per_label').
3. Invoices and reports automatically display rates in Indian Rupees.
```

---

## ✅ SECTION SUMMARY

| Table          | Purpose                                       | Relationships                      | Key Usage                                    |
| -------------- | --------------------------------------------- | ---------------------------------- | -------------------------------------------- |
| `payment_rate` | Defines user-specific payment rate and method | 1-1 with User, references Currency | Determines how users are compensated         |
| `currency`     | Stores ISO-standard currency codes and names  | Referenced by payment_rate         | Enables consistent multi-currency operations |

---



# ✉️ SECTION 5: INVITATIONS & ONBOARDING

---

# INVITATIONS & ONBOARDING – DATABASE SCHEMA DOCUMENTATION

## Complete Column Use Cases & Business Logic

**Responsibility:** User onboarding, organization/workspace access management, and secure invitation handling
**Sections:** 5
**Total Tables:** 4 tables
**Complexity:** High

---

## 📋 TABLE OF CONTENTS

### Section 5: Invitations & Onboarding

1. [users_invitation](#1-users_invitation)
2. [invitationgroupmember](#2-invitationgroupmember)
3. [invitationprojectmember](#3-invitationprojectmember)
4. [invitationworkspacemember](#4-invitationworkspacemember)

---

## 🎯 KEY USE CASES FOR INVITATIONS & ONBOARDING

### High-Level Business Flows:

1. **Token-Based Invitations** – Allow secure onboarding via one-time tokens.
2. **Role-Specific Memberships** – Define member roles across organizations, projects, and workspaces.
3. **Collaborative Access Control** – Enable cross-level invites with precise access types (Annotator, Reviewer, Admin).
4. **Expiration-Based Security** – Automatically invalidate old or unused invites.
5. **Multi-Entity Onboarding** – Invite users to multiple scopes (organization, workspace, project, or group).
6. **Auditability** – Track who invited whom, when, and with what permissions.

---

## SECTION 4: INVITATIONS & ONBOARDING

---

## 1. users_invitation

**Purpose:**
Manages all system invitations — including organization, workspace, and project onboarding.

**Table Name:** `users_invitation`
**Business Context:**
Centralized record of pending, accepted, or rejected invitations. Acts as the foundation for the onboarding workflow.

---

### Column Details

| Column              | Type                | Description            | Constraints                             | Use Case & Business Logic Purpose               |
| ------------------- | ------------------- | ---------------------- | --------------------------------------- | ----------------------------------------------- |
| **id**              | integer (AutoField) | Invitation ID          | PRIMARY KEY, AUTO_INCREMENT             | Unique identifier for each invitation           |
| **inviter_id**      | integer             | Inviting user          | FK → `htx_user`                         | Tracks who initiated the invitation             |
| **invitee_email**   | varchar(256)        | Email of invited user  | NOT NULL                                | Used to match with new or existing user account |
| **token**           | UUID                | Unique token           | UNIQUE, AUTO-GENERATED                  | Used for secure invitation links                |
| **organization_id** | integer             | Organization reference | FK → `organizations_organization`, NULL | Links invite to specific organization           |
| **member_type**     | varchar(10)         | Organization role      | Choices from `OrganizationMemberTypes`  | Defines user’s role upon acceptance             |
| **status**          | varchar(10)         | Invitation state       | Choices: pending/accepted/rejected      | Tracks lifecycle of the invitation              |
| **is_valid**        | boolean             | Validity flag          | DEFAULT FALSE                           | Becomes true after validation rules pass        |
| **created_at**      | datetime            | Created timestamp      | AUTO_NOW_ADD                            | When the invitation was generated               |
| **updated_at**      | datetime            | Updated timestamp      | AUTO_NOW                                | Tracks changes in invitation (status, validity) |
| **expires_at**      | datetime            | Expiration datetime    | NULL                                    | Invitation becomes invalid after this time      |
| **sent_at**         | datetime            | When email was sent    | NULL                                    | Used for re-sending logic                       |
| **responded_at**    | datetime            | When invitee responded | NULL                                    | Marks acceptance/rejection time                 |

---

### Business Logic Use Cases

**Primary Use Cases:**

* **Organization Onboarding:** Invite new members to specific roles in an organization.
* **Project Access Management:** Invite collaborators with scoped access rights.
* **Token-Based Validation:** Securely validate invite links with expiry times.
* **Email Integration:** Send automatic email invites via background jobs.
* **Status Tracking:** Track pending vs accepted vs rejected invitations for audit.

---

### Relationships

* 🔗 **Many-to-Many → Workspace** (via `InvitationWorkspaceMember`)
* 🔗 **Many-to-Many → Project** (via `InvitationProjectMember`)
* 🔗 **Many-to-Many → Group** (via `InvitationGroupMember`)
* 1️⃣ **Many-to-One → Organization** (defines organization scope)

---

### Workflow Example

```
1. Admin invites jane@company.com to organization “Acme Corp”.
2. System creates invitation record with token and expiry date.
3. Email sent with link: /invite/<token>.
4. Jane clicks link → verifies token → system marks as accepted.
5. User is automatically added to organization, workspace, and project roles.
```

---

## 2. invitationgroupmember

**Purpose:**
Defines group-level roles assigned via invitations.

**Table Name:** `invitationgroupmember`
**Business Context:**
Manages pending or confirmed invitations to user groups, ensuring group-level access and role assignment upon acceptance.

---

### Column Details

| Column            | Type                | Description       | Constraints                     | Use Case & Business Logic Purpose                          |
| ----------------- | ------------------- | ----------------- | ------------------------------- | ---------------------------------------------------------- |
| **id**            | integer (AutoField) | Record ID         | PRIMARY KEY, AUTO_INCREMENT     | Unique mapping entry                                       |
| **invitation_id** | integer             | Linked invitation | FK → `users_invitation`         | Associates invitation to a specific group context          |
| **group_id**      | integer             | Target group      | FK → `user_group`               | Specifies which group the invite applies to                |
| **member_type**   | varchar(100)        | Role in group     | Choices from `GroupMemberTypes` | Defines the invited user’s role: Annotator, Reviewer, etc. |

---

### Business Logic Use Cases

**Primary Use Cases:**

* Assign group-level access during the invitation process.
* Predefine user role before they even join.
* Enable batch invitations for groups.

**Workflow Example:**

```
1. Admin sends group invite to “QA Reviewers”.
2. InvitationGroupMember record created with member_type='REVIEWER'.
3. Upon acceptance, user auto-added as reviewer in that group.
```

---

### Relationships

* 🔗 **Many-to-One → users_invitation**
* 🔗 **Many-to-One → user_group**

---

## 3. invitationprojectmember

**Purpose:**
Handles project-level access during invitations.

**Table Name:** `invitationprojectmember`
**Business Context:**
Used to predefine user roles in projects (Annotator, Reviewer, etc.) when inviting new members.

---

### Column Details

| Column            | Type                | Description       | Constraints                | Use Case & Business Logic Purpose       |
| ----------------- | ------------------- | ----------------- | -------------------------- | --------------------------------------- |
| **id**            | integer (AutoField) | Record ID         | PRIMARY KEY                | Unique invitation-project mapping       |
| **invitation_id** | integer             | Linked invitation | FK → `users_invitation`    | Parent invitation                       |
| **project_id**    | integer             | Project reference | FK → `projects_project`    | Which project the invite applies to     |
| **member_type**   | varchar(15)         | Role in project   | Choices from `MemberTypes` | Defines user’s role within that project |

---

### Business Logic Use Cases

**Primary Use Cases:**

* Manage project onboarding via invitation tokens.
* Control project role assignment automatically.
* Separate permissions for annotation and review flows.

**Workflow Example:**

```
1. Project Manager invites new user to Project “LabelVision”.
2. InvitationProjectMember record created with role='ANNOTATOR'.
3. Once accepted, user joins project as Annotator automatically.
```

---

### Relationships

* 🔗 **Many-to-One → users_invitation**
* 🔗 **Many-to-One → projects_project**

---

## 4. invitationworkspacemember

**Purpose:**
Handles workspace-level invitations and access roles.

**Table Name:** `invitationworkspacemember`
**Business Context:**
Controls which workspace a user is invited to, along with their specific role within that workspace.

---

### Column Details

| Column            | Type                | Description         | Constraints                         | Use Case & Business Logic Purpose                     |
| ----------------- | ------------------- | ------------------- | ----------------------------------- | ----------------------------------------------------- |
| **id**            | integer (AutoField) | Record ID           | PRIMARY KEY                         | Unique identifier for workspace-invitation mapping    |
| **invitation_id** | integer             | Linked invitation   | FK → `users_invitation`             | Parent invitation reference                           |
| **workspace_id**  | integer             | Workspace reference | FK → `workspaces_workspace`         | Defines workspace scope of invitation                 |
| **member_type**   | varchar(10)         | Role in workspace   | Choices from `WorkspaceMemberTypes` | Determines access level (Admin, Member, Viewer, etc.) |

---

### Business Logic Use Cases

**Primary Use Cases:**

* Invite users to specific workspaces within an organization.
* Control access scope without granting org-wide rights.
* Predefine workspace roles via invitations.

**Workflow Example:**

```
1. Admin sends invitation for Workspace “Production”.
2. InvitationWorkspaceMember created with member_type='MEMBER'.
3. User accepts → gains access only to that workspace.
```

---

### Relationships

* 🔗 **Many-to-One → users_invitation**
* 🔗 **Many-to-One → workspaces_workspace**

---

## ✅ SECTION SUMMARY

| Table                       | Purpose                                   | Relationships                                    | Key Usage                                     |
| --------------------------- | ----------------------------------------- | ------------------------------------------------ | --------------------------------------------- |
| `users_invitation`          | Core invitation management for onboarding | Links to Organization, Group, Project, Workspace | Tracks invite lifecycle and roles             |
| `invitationgroupmember`     | Maps invitations to user groups           | FK to Invitation, Group                          | Defines group-level onboarding roles          |
| `invitationprojectmember`   | Maps invitations to projects              | FK to Invitation, Project                        | Assigns project-level roles during onboarding |
| `invitationworkspacemember` | Maps invitations to workspaces            | FK to Invitation, Workspace                      | Assigns workspace-level roles securely        |

---



