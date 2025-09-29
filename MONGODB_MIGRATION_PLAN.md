# Appwrite MongoDB Migration Plan

This document outlines the steps required to migrate Appwrite's primary database from MariaDB to MongoDB. This is a major architectural change that will require significant effort. This plan is divided into several phases to manage the complexity of the migration.

## Phase 1: Create a MongoDB Database Adapter for Utopia

The core of this migration is to create a new database adapter for the `Utopia\Database` library that Appwrite uses. This adapter will translate the Utopia Database queries and commands into MongoDB-specific operations.

### 1.1. Locate the Utopia Database Library

The `Utopia\Database` library is used extensively throughout the Appwrite codebase. The source code for this library needs to be located. Based on the file structure, it is likely located within the `src/` directory or included as a Composer dependency in `composer.json`.

### 1.2. Implement the MongoDB Adapter

A new class, `Utopia\Database\Adapter\MongoDB`, needs to be created. This class must implement the same interface as the existing `Utopia\Database\Adapter\MariaDB`. The key methods to implement will be:

*   `getCollection()`
*   `createCollection()`
*   `deleteCollection()`
*   `createAttribute()`
*   `deleteAttribute()`
*   `createIndex()`
*   `deleteIndex()`
*   `getDocument()`
*   `createDocument()`
*   `updateDocument()`
*   `deleteDocument()`
*   `find()`
*   `sum()`
*   `count()`

These methods will need to be implemented using the official [MongoDB PHP Driver](https://www.php.net/manual/en/book.mongodb.php).

### 1.3. Handle Data Type Conversion

The adapter will need to handle the conversion of Appwrite's data types to MongoDB's BSON data types. For example, Appwrite's `Database::VAR_STRING` will map to a BSON string, and `Database::VAR_INTEGER` will map to a BSON integer.

### 1.4. Implement Indexing

The adapter must support the creation and deletion of indexes. This will involve mapping Appwrite's index types (key, unique, fulltext) to MongoDB's index types.

## Phase 2: Integrate the MongoDB Adapter into Appwrite

Once the MongoDB adapter is created, it needs to be integrated into the Appwrite application.

### 2.1. Update `docker-compose.yml`

A new MongoDB service needs to be added to the `docker-compose.yml` file. The existing `mariadb` service will need to be replaced or made optional. All services that depend on `mariadb` will need to be updated to depend on the new `mongodb` service.

### 2.2. Update Environment Variables

New environment variables will be needed to configure the MongoDB connection (e.g., `_APP_DB_HOST`, `_APP_DB_PORT`, `_APP_DB_USER`, `_APP_DB_PASS`, `_APP_DB_SCHEMA`). The existing MariaDB environment variables will need to be updated or replaced.

### 2.3. Update Database Initialization

The database initialization logic in `app/init/registers.php` and `app/init/resources.php` will need to be updated to allow for the selection of the MongoDB adapter. This will likely involve a new environment variable to specify the database adapter to use (e.g., `_APP_DB_ADAPTER=mongodb`).

## Phase 3: Data Migration

A data migration strategy is needed to move data from an existing MariaDB database to a new MongoDB database.

### 3.1. Create a Migration Script

A new CLI command should be created to handle the data migration. This script will:
1.  Connect to both the MariaDB and MongoDB databases.
2.  Read the schema from the `app/config/collections/` directory.
3.  Iterate through each collection and document in the MariaDB database.
4.  Transform the data to a format suitable for MongoDB.
5.  Insert the data into the MongoDB database.

### 3.2. Handle Data Transformation

The migration script will need to handle any necessary data transformations. For example, it will need to convert MariaDB's integer-based timestamps to MongoDB's `BSON\UTCDateTime` objects.

## Phase 4: Testing

Thorough testing is critical to ensure the success of the migration.

### 4.1. Unit Tests

The new MongoDB adapter should have its own set of unit tests to ensure that it functions correctly.

### 4.2. End-to-End Tests

The existing end-to-end tests in the `tests/e2e` directory need to be run against an Appwrite instance configured to use the MongoDB adapter. All tests must pass before the migration can be considered complete.

### 4.3. Performance Testing

Performance testing should be conducted to compare the performance of the MongoDB-based Appwrite instance to the MariaDB-based instance. Any performance regressions should be identified and addressed.

## Conclusion

Migrating Appwrite to MongoDB is a complex but achievable task. By following this plan, we can systematically replace the database backend while minimizing disruption to the application. The key to success will be the careful implementation of the MongoDB adapter and thorough testing of the entire application.