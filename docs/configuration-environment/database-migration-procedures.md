# Database Migration Procedures

Database migrations are essential for evolving the data schema of the Gemforce platform, particularly for the MongoDB database used by Parse Server, and any auxiliary PostgreSQL databases. This document outlines the procedures, best practices, and tools used to manage schema changes, ensuring data integrity and minimal downtime during updates.

## 1. Principles of Database Migrations

*   **Idempotence**: Migration scripts should be executable multiple times without causing unintended side effects or errors.
*   **Version Control**: Migration scripts must be versioned and managed alongside the application codebase to ensure consistency and traceability.
*   **Backward Compatibility**: Whenever possible, schema changes should be backward compatible to support existing application logic during transition periods.
*   **Automated and Repeatable**: Migrations should be automated processes that can be reliably executed in all environments (development, staging, production).
*   **Rollback Capability**: Plans should include clear rollback procedures in case a migration fails or introduces critical issues.
*   **Zero Downtime (where possible)**: For critical production systems, strategies should minimize or eliminate downtime during migration.

## 2. MongoDB Migrations (Parse Server)

Parse Server's schema-less nature makes certain schema evolutions simpler, but structured migrations are still necessary for complex changes, data transformations, or ensuring consistency.

### 2.1. Types of MongoDB Migrations

*   **Non-breaking Changes**: Adding new fields, adding optional properties to existing documents. These often don't require explicit migration scripts if application logic handles missing fields gracefully.
*   **Breaking Changes**: Renaming fields, changing data types, removing fields, restructuring embedded documents. These require careful planning and dedicated migration scripts.

### 2.2. Tools for MongoDB Migrations

*   **Parse Server `_SCHEMA` Class**: Directly manipulating entries in the `_SCHEMA` collection to define field types and required status. While not a "migration tool," it's fundamental to Parse's soft schema.
*   **Custom Migration Scripts (Node.js)**: For complex data transformations, aggregation, or backfilling data, Node.js scripts using the Parse JS SDK or direct MongoDB driver are typically used.
*   **Migration Frameworks (e.g., `migrate-mongo`)**: For managing migration versions and applying them systematically.

### 2.3. Migration Workflow for MongoDB

1.  **Develop Migration Script**: Write a Node.js script that performs the necessary schema changes or data transformations.
    *   *Example: Renaming a field*
        ```javascript
        // Example MongoDB migration script (using native driver or Mongoose)
        module.exports = {
          async up(db, client) {
            // Logic to apply the migration
            // Example: Rename 'oldFieldName' to 'newFieldName' in 'MyClass'
            await db.collection('MyClass').updateMany(
              { oldFieldName: { $exists: true } },
              { $rename: { 'oldFieldName': 'newFieldName' } }
            );
          },

          async down(db, client) {
            // Logic to reverse the migration (for rollback)
            await db.collection('MyClass').updateMany(
              { newFieldName: { $exists: true } },
              { $rename: { 'newFieldName': 'oldFieldName' } }
            );
          }
        };
        ```
2.  **Test Migration**: Thoroughly test the migration script on a development or staging environment with a copy of production-like data.
3.  **Backup Database**: **Crucial step** before any production migration.
4.  **Execute Migration**: Run the migration script in the target environment. For critical production environments, consider a multi-phase approach:
    *   **Phase 1 (Backward Compatible Change)**: Introduce new fields while keeping old fields, and update application code to write to both (or prioritize new).
    *   **Phase 2 (Data Migration)**: Run a script to backfill data from old fields to new fields.
    *   **Phase 3 (Deprecation)**: Update application code to read only from new fields.
    *   **Phase 4 (Cleanup)**: Remove old fields from the database.

## 3. PostgreSQL Migrations (if applicable)

If PostgreSQL is used for specific data sets, standard relational database migration practices apply.

### 3.1. Tools for PostgreSQL Migrations

*   **Liquibase / Flyway**: Popular open-source database migration tools that manage schema changes using versioned SQL or XML files.
*   **ORM Migrations**: Many ORMs (e.g., Sequelize for Node.js, SQLAlchemy for Python, TypeORM for TypeScript) include built-in migration functionalities.

### 3.2. Migration Workflow for PostgreSQL

1.  **Create Migration File**: Generate a new migration file using the chosen tool. This file typically contains `UP` (for applying changes) and `DOWN` (for rolling back changes) SQL statements.
    *   *Example: SQL migration for adding a column*
        ```sql
        -- UP migration
        ALTER TABLE my_table ADD COLUMN new_column_name VARCHAR(255);

        -- DOWN migration
        ALTER TABLE my_table DROP COLUMN new_column_name;
        ```
2.  **Develop and Test**: Write SQL and test the migration against a staging database.
3.  **Backup Database**: Always perform a full database backup before production migrations.
4.  **Execute Migration**: Apply the migration using the tool. For production, consider:
    *   **Blue-Green Deployment**: Deploy new application version with new schema, then switch traffic.
    *   **Feature Flagging**: Use feature flags to enable/disable new application logic based on migration status.
    *   **Transaction Wrapping**: Ensure migrations are wrapped in transactions where possible for atomicity.

## 4. Rollback Procedures

*   **Pre-migration Snapshot/Backup**: The primary rollback mechanism. Always ensure a recent, verified backup is available.
*   **`DOWN` Scripts**: Use the `down` function in MongoDB scripts or `DOWN` SQL in PostgreSQL migrations to reverse changes. These should be tested.
*   **Application Version Rollback**: Be prepared to revert the application code to a previous version compatible with the rolled-back database schema.

## 5. Deployment and CI/CD Integration

*   **Automated Execution**: Integrate database migrations into CI/CD pipelines. Migrations are typically run as part of the deployment process.
*   **Permissions**: Ensure the deployment user has appropriate database permissions to execute schema changes.
*   **Logging**: All migration executions should be logged, including success/failure status and duration, for auditability.

By adhering to these rigorous database migration procedures, Gemforce can ensure seamless and safe evolution of its data models, supporting continuous development and deployment without compromising data integrity or system availability.