---
title: Row-Level Security (RLS) Overview
summary: Restrict access to specific rows of data based on user roles, permissions, or other criteria
toc: true
keywords: security, row level security, RLS
docs_area: develop
---

Row Level Security (_RLS_) is a security feature that allows organizations to restrict access to specific rows of data in a database based on user [roles]({% link {{ page.version.version }}/security-reference/authorization.md %}#roles), [permissions]({% link {{ page.version.version }}/security-reference/authorization.md %}#authorization-model), or other criteria.

Row-level security complements standard SQL privileges ([`GRANT`]({% link {{ page.version.version }}/grant.md %})/[`REVOKE`]({% link {{ page.version.version }}/revoke.md %})) by allowing administrators to define policies that determine precisely which rows users can view or modify within a specific table.

## Use cases

Use cases for row-level security include:

- [Restricting access to sensitive data for compliance](#restricting-access-to-sensitive-data-for-compliance)
- [Designing multi-tenant applications](#designing-multi-tenant-applications)

### Restricting access to sensitive data for compliance

In industries like finance or healthcare, organizations are required to ensure that only authorized users access sensitive data. Row-level security (RLS) addresses this requirement directly within the database.

For example, RLS allows a financial institution to restrict access to customer records based on roles or departments. In healthcare, RLS can be used to enforce policies ensuring patient records are visible only to the medical staff involved in their care.

RLS embeds access control logic directly into the database and eliminates the need for manual filtering in application code. This centralized enforcement prevents inconsistencies, reduces security attack surface, and simplifies compliance with data access regulations.

### Designing multi-tenant applications

In multi-tenant applications such as typical Software-as-a-Service (SaaS) deployments, isolating data between tenants within shared tables is a requirement. Row-Level Security (RLS) provides a database-level mechanism for enforcing this isolation. SaaS providers can utilize RLS policies to ensure tenants can only access their own data, eliminating the need for complex and potentially insecure application-layer filtering logic based on tenant IDs.

[XXX](XXX): add note re: index design with custom prefix (tenant ID?), and the fact that we don't automatically create indexes for you in this case (if I heard correctly at bug bash)???

## How to use row-level security

At a high level, the steps for using row-level security (RLS) are as follows:

1. Create [schema objects]({% link {{ page.version.version }}/schema-design-overview.md %}) and [insert data]({% link {{ page.version.version }}/insert-data.md %}). ([`CREATE TABLE`]({% link {{ page.version.version }}/create-table.md %}), [`INSERT`]({% link {{ page.version.version }}/insert.md %}))
2. [Create roles]({% link {{ page.version.version }}/create-role.md %}) & [grant access]({% link {{ page.version.version }}/grant.md %}) to schema objects by those roles. ([`CREATE ROLE`]({% link {{ page.version.version }}/create-role.md %}), [`GRANT`]({% link {{ page.version.version }}/grant.md %}))
3. Enable row-level security on the schema objects. ([`ALTER TABLE ... ENABLE ROW LEVEL SECURITY`]({% link {{ page.version.version }}/alter-table.md %}#enable-disable-row-level-security))
4. Define row-level security policies on the schema objects which are assigned to specific roles. ([`CREATE POLICY`]({% link {{ page.version.version }}/create-policy.md %}))

For detailed examples showing how to use row-level security, see the [examples](#examples).

## How row-level security policies are evaluated

Policies function as filters or constraints applied automatically by CockroachDB during [query execution]({% link {{ page.version.version }}/architecture/sql-layer.md %}#query-execution). They are based on boolean expressions evaluated in the context of the current [user]({% link {{ page.version.version }}/security-reference/authorization.md %}#roles), [session properties]({% link {{ page.version.version }}/show-sessions.md %}), and the row data itself.

When row-level security is enabled on a table:

1. Existing [SQL privileges]({% link {{ page.version.version }}/security-reference/authorization.md %}#supported-privileges) still determine **if** a user can access the table at all (e.g., `SELECT`, `INSERT`).
2. [Row-level security policies]({% link {{ page.version.version }}/show-policies.md %}) determine **which rows** within the table are accessible or modifiable for specific commands.

Further details about RLS evaluation include:

- All [policies]({% link {{ page.version.version }}/show-policies.md %}) apply to a specific set of [roles]({% link {{ page.version.version }}/security-reference/authorization.md %}#roles). For a policy to be applicable, it must match at least one of the roles assigned to it. If the policy is associated with the `PUBLIC` role, it applies to all roles. In order for reads or writes to succeed, there must be at least one permissive policy for the user's role. 
- If RLS is enabled but no policies apply to a given combination of user and SQL statement, **access is denied by default**.
- Permissive policies are combined using `OR` logic, while restrictive policies are combined using `AND` logic. The overall policy enforcement is determined by evaluating a logical expression of the form: `(permissive policies) AND (restrictive policies)`.
- The `USING` clause of [`CREATE POLICY`]({% link {{ page.version.version }}/create-policy.md %}) filters rows during reads; the `WITH CHECK` clause validates writes, and defaults to `USING` if absent.

## Considerations

### Performance

Complex [policy expressions]({% link {{ page.version.version }}/create-policy.md %}) evaluated per-row can impact query performance. To limit the performance impacts of row-level security, optimize your policy expressions and consider [indexing]({% link {{ page.version.version }}/indexes.md %}) relevant columns.

According to internal testing, row-level security had the following performance impacts on a write-heavy [`sysbench`](https://github.com/akopytov/sysbench) workload:

| Number of policies | Percentage slowdown of the workload (approx.) |
|--------------------|-----------------------------------------------|
| 1                  | 110%                                          |
| 10                 | 140%                                          |
| 50                 | 200%                                          |

([XXX](XXX): above numbers are via internal Slack convo, confirm if we want this in docs ??? i feel like someone is going to ask anyway so we might as well say?)

### Security privileges

[Policy expressions]({% link {{ page.version.version }}/create-policy.md %}) execute with the [privileges]({% link {{ page.version.version }}/security-reference/authorization.md %}#supported-privileges) of the user invoking the query, unless functions marked [`SECURITY DEFINER`]({% link {{ page.version.version }}/create-function.md %}#create-a-security-definer-function) are used. 

{{site.data.alerts.callout_danger}}
Functions marked `SECURITY DEFINER` should only be used with **extreme caution** to ensure expressions do not have unintended side effects.
{{site.data.alerts.end}}

## Limitations

### SQL language features that bypass row-level security

The following SQL language features bypass row-level security:

- [Foreign keys]({% link {{ page.version.version }}/foreign-key.md %}) (including cascades)
- [Primary key constraints]({% link {{ page.version.version }}/primary-key.md %})
- [Unique constraints]({% link {{ page.version.version }}/unique.md %})
- [`TRUNCATE`]({% link {{ page.version.version }}/truncate.md %})

### Change data capture (CDC)

[CDC]({% link {{ page.version.version }}/change-data-capture-overview.md %}) messages that are emitted from a table will not be filtered using RLS policies. Furthermore, [CDC queries]({% link {{ page.version.version }}/cdc-queries.md %}) are not supported on tables using RLS, and will fail with the error message: `CDC queries are not supported on tables with row-level security enabled`

### Backup and restore

[Backup and restore]({% link {{ page.version.version }}/backup-and-restore-overview.md %}) functionality does not take RLS policies into account.

### Logical data replication (LDR) and Physical cluster replication (PCR)

[Logical Data Replication (LDR)]({% link {{ page.version.version }}/logical-data-replication-overview.md %}) and [Physical Cluster Replication (PCR)]({% link {{ page.version.version }}/physical-cluster-replication-overview.md %}) do not take RLS policies into account.

LDR's [limitations with respect to schema changes]({% link {{ page.version.version }}/manage-logical-data-replication.md %}#schema-changes) also apply to RLS, since RLS policies amount to a schema change.

If you use PCR, the target cluster will have all RLS policies applied to the data because PCR performs byte for byte replication.

### Views

When [views]({% link {{ page.version.version }}/views.md %}) are accessed, RLS policies on any underlying [tables]({% link {{ page.version.version }}/schema-design-table.md %}) are applied. [Policies]({% link {{ page.version.version }}/create-policy.md %}) can also be defined directly on views.

Views use the [role]({% link {{ page.version.version }}/security-reference/authorization.md %}#roles) of the view owner to determine row-level security filters, **not** the role of the user executing the view. This can cause issues because the view owner may have entirely different policies than the user executing the view.

The following security attributes for views are unimplemented:

- `security_invoker`, which would allow using the role of the user executing the view.
- `security_barrier`, which instructs the [optimizer]({% link {{ page.version.version }}/cost-based-optimizer.md %}) to process policy filtering first, ensuring that [user-defined functions]({% link {{ page.version.version }}/user-defined-functions.md %}) (UDFs) never receive rows violating RLS policies.

## Examples

### Create a policy

For an example, see [`CREATE POLICY`]({% link {{ page.version.version }}/create-policy.md %}).

### Alter a policy

For an example, see [`ALTER POLICY`]({% link {{ page.version.version }}/alter-policy.md %}).

### Drop a policy

For an example, see [`DROP POLICY`]({% link {{ page.version.version }}/drop-policy.md %}).

### Enable or disable row-level security

For examples, see:

- [`ALTER TABLE ... (ENABLE, DISABLE) ROW LEVEL SECURITY`]({% link {{ page.version.version }}/alter-table.md %}#enable-disable-row-level-security).
- [`ALTER TABLE ... (FORCE, UNFORCE) ROW LEVEL SECURITY`]({% link {{ page.version.version }}/alter-table.md %}#force-unforce-row-level-security).

### Example: RLS for Data Security (Fine-Grained Access Control)

[XXX](XXX): EDIT THIS EXAMPLE (fine grained access control)

#### Goal

Restrict access to specific rows within a table based on user roles, attributes, or relationships defined within the data itself. This goes beyond table-level `GRANT` permissions. Common examples include restricting access to salary information, personal data, or region-specific records.

#### Scenario Example: Employee Data Access

Consider an `employees` table containing sensitive salary information. We want to enforce the following rules:
- Employees can view their own record.
- Managers can view the records of their direct reports.
- Members of the `hr_department` role can view all records.

#### Implementation Example

1. Sample Schema & Roles:

{% include_cached copy-clipboard.html %}
~~~ sql
-- Create a role needed for the example policies.
-- Note: In a real scenario, manage roles appropriately.
-- This may require admin privileges not available to all users.
CREATE ROLE hr_department;

-- Assume roles 'hr_department' and potentially others exist.
-- Usernames are assumed to match the 'username' column for simplicity.
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    username TEXT UNIQUE NOT NULL,
    full_name TEXT NOT NULL,
    manager_username TEXT,
    salary NUMERIC(10, 2) NOT NULL
);

-- Sample Data
INSERT INTO employees (username, full_name, manager_username, salary) VALUES
('alice', 'Alice Smith', NULL, 120000),
('bob', 'Bob Jones', 'alice', 80000),
('carol', 'Carol White', 'alice', 85000),
('david', 'David Green', 'carol', 70000);

-- Grant basic table access (RLS will refine row access)
GRANT SELECT, UPDATE(full_name) ON employees TO PUBLIC; -- Example: allow self-update of name
GRANT SELECT, INSERT, UPDATE, DELETE ON employees TO hr_department;
~~~

2. Enable RLS:

{% include_cached copy-clipboard.html %}
~~~ sql
ALTER TABLE employees ENABLE ROW LEVEL SECURITY;
-- Optional: Ensure owner is also subject to policies if needed
-- ALTER TABLE employees FORCE ROW LEVEL SECURITY;
~~~

3. Define Policies:

{% include_cached copy-clipboard.html %}
~~~ sql
-- Policy 1: Allow HR full access
CREATE POLICY hr_access ON employees
    FOR ALL -- Applies to SELECT, INSERT, UPDATE, DELETE
    TO hr_department
    USING (true) -- No row restriction for HR
    WITH CHECK (true); -- No check restriction for HR
~~~

{% include_cached copy-clipboard.html %}
~~~ sql
-- Policy 2: Allow employees to view/update their own record
-- Note: Assumes database username matches 'employees.username'
CREATE POLICY self_access ON employees
    AS PERMISSIVE -- Combine with other permissive policies (like manager access)
    FOR SELECT -- Only applies to SELECT queries
    TO PUBLIC -- Applies to all roles not covered by more specific policies
    USING (username = current_user)
    WITH CHECK (username = current_user); -- Ensure users can only update their own record
~~~

{% include_cached copy-clipboard.html %}
~~~ sql
-- Policy 3: Allow managers to view direct reports' records
-- Requires a way to look up the manager's username. Using current_user here.
CREATE POLICY manager_access ON employees
    AS PERMISSIVE -- Combine with self_access
    FOR SELECT -- Only for viewing
    TO PUBLIC
    USING (manager_username = current_user);
    -- No WITH CHECK needed as it's SELECT-only
~~~

4. Verification (Conceptual):

- `SELECT * FROM employees;` executed by user `alice` (manager): Shows Alice, Bob, Carol.
- `SELECT * FROM employees;` executed by user `bob` (employee): Shows only Bob.
- `SELECT * FROM employees;` executed by user `carol` (manager): Shows Carol, David.
- `SELECT * FROM employees;` executed by user belonging to `hr_department`: Shows all rows.
- `UPDATE employees SET salary = 999999 WHERE username = 'bob';` executed by `alice`: Fails (violates `self_access` `WITH CHECK` if Alice tries to update Bob's salary, and `manager_access` doesn't grant `UPDATE`).
- `UPDATE employees SET full_name = 'Robert Jones' WHERE username = 'bob';` executed by `bob`: Succeeds (allowed by `self_access`).

#### Considerations

- Performance: Complex `USING` clauses, especially those involving subqueries or joins to check relationships (like manager hierarchy), can impact query performance. Index relevant columns (`username`, `manager_username`).
- `PERMISSIVE` vs `RESTRICTIVE`: `PERMISSIVE` is often suitable here, allowing access if *any* relevant policy grants it (e.g., self-access OR manager-access). `RESTRICTIVE` policies act as additional hurdles that *must* be passed.
- `SECURITY DEFINER` Functions: If policy logic requires elevated privileges (e.g., accessing a central authorization table), use `SECURITY DEFINER` functions with extreme caution, ensuring they are secure against misuse.
- Complexity: Managing many overlapping policies requires careful design, naming, and documentation. Consider simplifying rules or using role inheritance where possible.

### Example: RLS for Multi-Tenant Isolation

[XXX](XXX): EDIT THIS EXAMPLE (multi-tenant isolation)

#### Goal

Enforce strict data separation between different tenants (customers, organizations) sharing the same database infrastructure and schema. Each tenant must only be able to see and modify their own data. This is critical for SaaS applications.

#### Scenario Example: Shared Invoice Table

A SaaS application serves multiple tenants. All invoice data resides in a single `invoices` table, distinguished by a `tenant_id` column. The application ensures that user sessions are associated with a specific `tenant_id`.

#### Implementation Example

1. Sample Schema & Tenant Context:

{% include_cached copy-clipboard.html %}
~~~ sql
CREATE TABLE IF NOT EXISTS tenants (
    tenant_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL UNIQUE
);

CREATE TABLE IF NOT EXISTS invoices (
    invoice_id SERIAL PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(tenant_id),
    customer_name TEXT NOT NULL,
    amount NUMERIC(12, 2) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Index tenant_id for performance! Crucial for RLS.
CREATE INDEX idx_invoices_tenant_id ON invoices(tenant_id);

-- Assume application sets the tenant context for the session, e.g.:
-- SET app.current_tenant_id = '...'; -- (Use appropriate GUC or session variable)
-- We will use current_setting('app.current_tenant_id') in policies.
-- Ensure this setting is reliably managed by the application layer.
~~~

2. Enable RLS:

{% include_cached copy-clipboard.html %}
~~~ sql
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;
ALTER TABLE invoices FORCE ROW LEVEL SECURITY; -- Crucial: Apply to owner/admins too unless bypassed explicitly
~~~

3. Define Tenant Isolation Policy:

A single, robust policy is often sufficient and preferred for clarity and security.

{% include_cached copy-clipboard.html %}
~~~ sql
-- Policy: Enforce tenant isolation for all operations
CREATE POLICY tenant_isolation ON invoices
    AS RESTRICTIVE -- Ensures this MUST pass, prevents accidental bypass by permissive policies
    FOR ALL -- Applies to SELECT, INSERT, UPDATE, DELETE
    TO PUBLIC -- Applies to all application users (adjust if specific roles are used)
    USING (tenant_id = current_setting('app.current_tenant_id', true)::UUID) -- Filter rows on SELECT/UPDATE/DELETE
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id', true)::UUID); -- Enforce tenant_id on INSERT/UPDATE
~~~

Explanation:

- `AS RESTRICTIVE`: Makes this policy mandatory. If other policies exist, they must *also* pass. For simple tenant isolation, this is often the safest default. If only this policy exists, `PERMISSIVE` would functionally be similar but `RESTRICTIVE` signals intent more clearly.
- `FOR ALL`: Covers all data modification and retrieval.
- `TO PUBLIC`: Applies the policy broadly. Roles should primarily manage *table-level* access (`GRANT`), while this policy handles *row-level* visibility.
- `USING`: Ensures queries only *see* rows matching the session's `tenant_id`. The `current_setting('var', true)` variant returns NULL if the setting is not defined, preventing accidental exposure if the context isn't set (NULL comparison yields NULL, denying access). The cast `::UUID` is necessary if `tenant_id` is UUID type.
- `WITH CHECK`: Critically important. Prevents users from `INSERT`ing rows with a `tenant_id` different from their session's `tenant_id`, or `UPDATE`ing a row to change its `tenant_id` across boundaries. Without this, a user could potentially insert data into another tenant's space.

4. Verification (Conceptual):

- Session A: `SET app.current_tenant_id = 'tenant-A-uuid'; SELECT * FROM invoices;` -> Shows only invoices where `tenant_id = 'tenant-A-uuid'`.
- Session B: `SET app.current_tenant_id = 'tenant-B-uuid'; SELECT * FROM invoices;` -> Shows only invoices where `tenant_id = 'tenant-B-uuid'`.
- Session A: `INSERT INTO invoices (tenant_id, ...) VALUES ('tenant-B-uuid', ...);` -> Fails (violates `WITH CHECK` constraint).
- Session A: `INSERT INTO invoices (tenant_id, ...) VALUES ('tenant-A-uuid', ...);` -> Succeeds.
- Session A: `UPDATE invoices SET tenant_id = 'tenant-B-uuid' WHERE invoice_id = 123;` -> Fails (violates `WITH CHECK`).

#### Considerations

- Tenant Context Management: The reliability of setting and isolating the `app.current_tenant_id` (or equivalent mechanism) per session/request is paramount. This is typically handled in the application layer or connection pool middleware. Failure here bypasses RLS.
- Performance: The `USING (tenant_id = ...)` check is highly efficient if `tenant_id` is indexed. Avoid complex logic in tenant isolation policies; keep them simple and fast.
- `WITH CHECK` is Non-Negotiable: Omitting `WITH CHECK` creates a severe security vulnerability, allowing cross-tenant data insertion or modification.
- Superuser/Admin Access: Database superusers and roles with the `BYPASSRLS` attribute bypass RLS. Application-level administrative roles needing cross-tenant access might require specific functions or views designed with `SECURITY DEFINER` (use cautiously) or temporary privilege escalation managed outside standard RLS.
- Default Deny: If RLS is enabled (`FORCE`) and the `current_setting` is missing, the `USING` condition evaluates to NULL, denying access, which is generally the desired safe behavior.
- Testing: Rigorously test tenant isolation, including edge cases and attempts to circumvent the policies.

### Video demo: Row-level Security

For a demo showing how to combine Row-level security with [Multi-region SQL]({% link {{ page.version.version }}/multiregion-overview.md %}) to constrain access to specific rows based on a user's geographic region, play the following video:

{% include_cached youtube.html video_id="ZG8RsfwMaa8" %}

## See also

+ [`SHOW POLICIES`]({% link {{ page.version.version }}/show-policies.md %})
- [`CREATE POLICY`]({% link {{ page.version.version }}/create-policy.md %})
- [`ALTER POLICY`]({% link {{ page.version.version }}/alter-policy.md %})
- [`DROP POLICY`]({% link {{ page.version.version }}/drop-policy.md %})
- [`ALTER TABLE {ENABLE, DISABLE} ROW LEVEL SECURITY`]({% link {{ page.version.version }}/alter-table.md %}#enable-disable-row-level-security)
- [`ALTER TABLE {FORCE, UNFORCE} ROW LEVEL SECURITY`]({% link {{ page.version.version }}/alter-table.md %}#force-unforce-row-level-security)
- [`CREATE ROLE ... WITH BYPASSRLS`]({% link {{ page.version.version }}/create-role.md %}#create-a-role-that-can-bypass-row-level-security-rls)
- [`ALTER ROLE ... WITH BYPASSRLS`]({% link {{ page.version.version }}/alter-role.md %}#allow-a-role-to-bypass-row-level-security-rls)

<!-- Sqlchecker test cleanup block. NB. This must always come last. Be sure to comment this out when finished writing the doc. -->

<!--

{% include_cached copy-clipboard.html %}
~~~ sql
DROP POLICY IF EXISTS hr_access ON employees CASCADE;
REVOKE ALL ON employees FROM hr_department;
DROP ROLE hr_department;
DROP POLICY IF EXISTS user_orders_policy ON orders CASCADE;
DROP TABLE IF EXISTS employees, orders CASCADE;
DROP POLICY IF EXISTS tenant_isolation ON invoices CASCADE;
DROP TABLE IF EXISTS tenants, invoices CASCADE;
~~~

-->
