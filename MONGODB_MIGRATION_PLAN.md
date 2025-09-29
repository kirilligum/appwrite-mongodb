# Appwrite MongoDB Migration Plan (Detailed)

This document outlines a detailed, step-by-step plan to migrate Appwrite's primary database from MariaDB to MongoDB. This is a major architectural change that requires a phased approach with rigorous testing at each step.

---

## Phase 1: Create a MongoDB Database Adapter for Utopia

The core of this migration is to create a new database adapter for the `Utopia\Database` library that Appwrite uses. This adapter will translate the Utopia Database queries and commands into MongoDB-specific operations.

### **Step 1.1: Set up the Development Environment for the Utopia Library**
*   **Action:**
    1.  Locate the `utopia-php/database` library in the `composer.json` file to identify its source and version.
    2.  Set up a separate development environment or a local test bench to work on the library in isolation from the main Appwrite application. This will allow for focused development and testing of the adapter.
*   **Test:**
    *   Run the existing unit tests for the `utopia-php/database` library to ensure the development environment is correctly configured and all current tests pass.

### **Step 1.2: Create the MongoDB Adapter Class Skeleton**
*   **Action:**
    1.  Create a new file: `Utopia/Database/Adapter/MongoDB.php`.
    2.  Define the `MongoDB` class within this file.
    3.  Make the class implement the `Utopia\Database\Adapter` interface.
    4.  Add placeholder methods for all the required interface methods.
*   **Test:**
    *   Write a basic unit test that attempts to instantiate the `MongoDB` adapter.
    *   Assert that the created object is an instance of `Utopia\Database\Adapter`. This test will initially fail until all abstract methods are implemented.

### **Step 1.3: Implement Collection Management**
*   **Action:**
    1.  Implement the `createCollection()` and `deleteCollection()` methods using the MongoDB PHP driver. These will correspond to `createCollection` and `dropCollection` in MongoDB.
    2.  Implement `getCollection()` to verify the existence of a collection.
*   **Test:**
    *   Write unit tests that:
        1.  Create a new test collection.
        2.  Verify the collection's existence.
        3.  Delete the collection.
        4.  Verify the collection has been successfully deleted.

### **Step 1.4: Implement Attribute and Index Management**
*   **Action:**
    1.  **Attributes:** In MongoDB, documents within a collection are schema-less. Appwrite's concept of "attributes" needs to be managed manually. A common approach is to create a metadata collection (e.g., `_schemas`) to store the attribute definitions for each Appwrite collection. The `createAttribute()` and `deleteAttribute()` methods will manage documents in this metadata collection.
    2.  **Indexes:** Implement `createIndex()` and `deleteIndex()` to map Appwrite's index types (key, unique, fulltext) to their MongoDB counterparts.
*   **Test:**
    *   **Attributes:** Write unit tests to add, retrieve, and remove attribute definitions from the metadata collection.
    *   **Indexes:** Write unit tests that create various types of indexes (single-field, compound, unique, text) on a test collection and then verify their successful deletion.

### **Step 1.5: Implement Document CRUD Operations**
*   **Action:**
    1.  Implement the core document operations: `createDocument()`, `getDocument()`, `updateDocument()`, and `deleteDocument()`.
    2.  These methods will use the MongoDB PHP driver's `insertOne()`, `findOne()`, `updateOne()`, and `deleteOne()` functions.
*   **Test:**
    *   Write comprehensive unit tests for each CRUD operation.
    *   Tests should cover creating a document, retrieving it by its ID, updating its fields, and deleting it.
    *   Include tests for edge cases like attempting to get or delete a non-existent document.

### **Step 1.6: Implement Querying (`find` method)**
*   **Action:**
    1.  This is a critical and complex step. The `find()` method receives an array of `Utopia\Database\Query` objects.
    2.  Implement a translation layer that converts each `Utopia\Database\Query` object into the corresponding MongoDB query filter syntax (e.g., `Query::TYPE_EQUAL` becomes `$eq`, `Query::TYPE_GREATER` becomes `$gt`).
    3.  Handle all query types, including logical operators (`AND`, `OR`).
*   **Test:**
    *   Write extensive unit tests covering every query type and operator supported by Utopia.
    *   For each test, create a set of test documents, execute a query, and assert that the correct subset of documents is returned.

### **Step 1.7: Implement Aggregate Operations**
*   **Action:**
    1.  Implement the `sum()` and `count()` methods.
    2.  These methods will use MongoDB's aggregation framework (`$group`, `$sum`, `$count`).
*   **Test:**
    *   Write unit tests that perform `sum` and `count` operations on a collection of test documents with various query filters and verify the results.

---

## Phase 2: Integrate the MongoDB Adapter into Appwrite

### **Step 2.1: Update Docker and Configuration**
*   **Action:**
    1.  Add a new `mongodb` service to the `docker-compose.yml` file.
    2.  Update all Appwrite services that have a `depends_on: [mariadb]` directive to also include `mongodb`.
    3.  Introduce new environment variables for MongoDB configuration (e.g., `_APP_DB_ADAPTER=mongodb`, `_APP_DB_DSN`).
    4.  Update the database initialization logic in `app/init/registers.php` and `app/init/resources.php` to check for the `_APP_DB_ADAPTER` variable and instantiate the correct adapter (`MariaDB` or `MongoDB`).
*   **Test:**
    1.  Start the Appwrite stack with `_APP_DB_ADAPTER=mongodb`.
    2.  Check the logs to ensure the Appwrite containers start without any database connection errors.
    3.  Confirm that the MongoDB container is running and accessible.

### **Step 2.2: Full-System Integration Testing**
*   **Action:**
    1.  With Appwrite running on the new MongoDB adapter, execute the entire suite of end-to-end tests located in the `tests/e2e` directory.
*   **Test:**
    *   **All end-to-end tests must pass.** This is the ultimate validation of the adapter's correctness and its integration into the Appwrite ecosystem. Any failures must be debugged and fixed before proceeding.

---

## Phase 3: Data Migration

### **Step 3.1: Create the Migration CLI Tool**
*   **Action:**
    1.  Create a new CLI command specifically for migrating data from a MariaDB instance to a MongoDB instance.
    2.  The command should accept connection parameters for both the source (MariaDB) and destination (MongoDB) databases.
*   **Test:**
    *   Run the new command with the `--help` flag to ensure it is correctly registered and its options are documented.

### **Step 3.2: Implement and Test Migration Logic**
*   **Action:**
    1.  Implement the core migration logic. The script will need to:
        *   Connect to both databases.
        *   Read the schema from the `app/config/collections/` directory.
        *   Iterate through each collection and document in the MariaDB database.
        *   Transform data types as needed (e.g., timestamps).
        *   Insert the transformed data into the corresponding MongoDB collection.
*   **Test:**
    1.  Create a small, representative test database in MariaDB.
    2.  Run the migration script.
    3.  Write a verification script or manually inspect the MongoDB database to ensure that all data has been migrated accurately and with the correct data types.

---

## Phase 4: Finalization and Submission

### **Step 4.1: Code Review and Documentation**
*   **Action:**
    1.  Ensure all new code is well-documented and follows Appwrite's coding standards.
    2.  Update the main `README.md` and any relevant documentation to reflect the new database option.
*   **Test:**
    *   Submit the code for a final review by the Appwrite core team.

### **Step 4.2: Submit the Final Pull Request**
*   **Action:**
    1.  Submit the completed work as a pull request to the Appwrite repository. The PR will include the new MongoDB adapter, the updated Appwrite configuration, and the migration tool.
*   **Test:**
    *   The PR should be approved and merged by the user.

This detailed plan provides a clear, actionable roadmap for this complex migration. I am now ready to proceed with the first step.