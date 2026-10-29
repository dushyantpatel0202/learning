# Learn MySQL with the Doctor Appointment App

This guide shows how the current doctor appointment app could be modeled with MySQL. The running project currently uses **MongoDB through Mongoose**, so every MySQL schema and query below is a learning example and is **not implemented** in the backend. The frontend would still call the Express API; only the backend's database layer would change.

## 1. What MySQL is

MySQL is a relational database. Data is organized into tables with defined columns and data types. Rows represent records, and relationships between tables are usually represented by foreign keys. SQL (Structured Query Language) is used to create tables and read or change data.

MySQL is not the only SQL database. PostgreSQL is another relational database. SQL syntax and features vary somewhat between database products; examples here use common MySQL 8 syntax.

## 2. How the current app maps to relational data

The existing Mongoose models are `User`, `Doctor`, `Appointment`, and `Token`:

| Current MongoDB model | Possible MySQL table | Relationship / notes |
| --- | --- | --- |
| User | `users` | One app account; Firebase UID and email can be unique |
| Doctor | `doctors` | One doctor profile belongs to a user |
| Doctor `charges` array | `doctor_services` | One doctor can offer many services, each with a fee |
| Appointment | `appointments` | Each booking belongs to one patient and one doctor |
| Token | `tokens` | Each queue token belongs to one patient and one doctor for a date |
| Doctor `tokenSystem` | `doctor_daily_queues` (possible design) | Daily queue state can be stored per doctor and date |

The doctor profile's nested values become ordinary columns or related rows. In a relational database, a list like `charges` is usually represented by another table, rather than a JSON array in the doctor row. A complex or rarely queried flexible field could be stored as JSON in MySQL, but relational columns are easier to constrain and query for core data.

## 3. Tables, rows, columns, and data types

A table has a defined set of columns; each row is one record. Choose types that match the meaning of the data: `DATE` for a calendar date, `TIME` for a time of day, `DECIMAL` for money, and integer types for identifiers and counts.

Here is an illustrative starting schema. It is designed for learning, not a drop-in migration script:

```sql
CREATE TABLE users (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  firebase_uid VARCHAR(128) NOT NULL UNIQUE,
  name VARCHAR(160) NOT NULL,
  email VARCHAR(254) NULL UNIQUE,
  phone VARCHAR(32) NULL,
  role ENUM('patient', 'doctor') NOT NULL DEFAULT 'patient',
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE doctors (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT UNSIGNED NOT NULL UNIQUE,
  doctor_code INT UNSIGNED NOT NULL UNIQUE,
  name VARCHAR(160) NOT NULL,
  gender ENUM('m', 'f', 'other') NULL,
  photo_url VARCHAR(2048) NOT NULL DEFAULT '',
  specialization VARCHAR(120) NOT NULL,
  qualifications VARCHAR(255) NOT NULL,
  experience_years SMALLINT UNSIGNED NOT NULL DEFAULT 0,
  clinic_name VARCHAR(200) NOT NULL,
  city VARCHAR(120) NOT NULL,
  district VARCHAR(120) NOT NULL,
  latitude DECIMAL(10, 7) NOT NULL,
  longitude DECIMAL(10, 7) NOT NULL,
  address VARCHAR(500) NOT NULL,
  verified BOOLEAN NOT NULL DEFAULT FALSE,
  about TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  CONSTRAINT fk_doctors_user FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE TABLE doctor_services (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  doctor_id BIGINT UNSIGNED NOT NULL,
  service_name VARCHAR(160) NOT NULL,
  fee DECIMAL(10, 2) NOT NULL,
  CONSTRAINT chk_service_fee CHECK (fee >= 0),
  CONSTRAINT fk_services_doctor FOREIGN KEY (doctor_id) REFERENCES doctors(id),
  UNIQUE KEY uq_doctor_service (doctor_id, service_name)
);

CREATE TABLE appointments (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  doctor_id BIGINT UNSIGNED NOT NULL,
  patient_id BIGINT UNSIGNED NOT NULL,
  service_name VARCHAR(160) NOT NULL,
  fee_at_booking DECIMAL(10, 2) NOT NULL,
  appointment_date DATE NOT NULL,
  time_slot TIME NOT NULL,
  status ENUM('pending', 'confirmed', 'completed', 'cancelled') NOT NULL DEFAULT 'pending',
  notes TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  CONSTRAINT fk_appointments_doctor FOREIGN KEY (doctor_id) REFERENCES doctors(id),
  CONSTRAINT fk_appointments_patient FOREIGN KEY (patient_id) REFERENCES users(id),
  INDEX ix_appointments_patient_date (patient_id, appointment_date),
  INDEX ix_appointments_doctor_date (doctor_id, appointment_date)
);

CREATE TABLE tokens (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  doctor_id BIGINT UNSIGNED NOT NULL,
  patient_id BIGINT UNSIGNED NOT NULL,
  token_date DATE NOT NULL,
  token_number INT UNSIGNED NOT NULL,
  status ENUM('waiting', 'serving', 'done', 'cancelled') NOT NULL DEFAULT 'waiting',
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT fk_tokens_doctor FOREIGN KEY (doctor_id) REFERENCES doctors(id),
  CONSTRAINT fk_tokens_patient FOREIGN KEY (patient_id) REFERENCES users(id),
  UNIQUE KEY uq_patient_daily_token (doctor_id, patient_id, token_date),
  UNIQUE KEY uq_daily_token_number (doctor_id, token_date, token_number),
  INDEX ix_tokens_daily_queue (doctor_id, token_date, status, token_number)
);
```

For production, review the exact MySQL version, collation, nullability, time zone rules, and constraints. The current app allows some optional fields and stores a date as a string; a migration must define how existing values map into these column types. MySQL unique indexes permit multiple `NULL` values, which is relevant for optional email.

## 4. Primary keys and foreign keys

A primary key uniquely identifies a row. A foreign key requires a referenced row to exist, helping prevent orphan records. This is similar in purpose to MongoDB ObjectId references, but a relational foreign key is checked by the database.

**In the proposed design:** `doctors.user_id` references `users.id`; `appointments.doctor_id` and `patient_id` reference their owners. `UNIQUE` on `doctors.user_id` models the app rule that one user has at most one doctor profile.

Foreign keys require decisions for deletion. For example, deleting a user with appointment history might be disallowed, or the account could be deactivated while preserving historical records. Do not use cascading deletion without considering medical and audit history requirements.

## 5. Relationships and normalization

Normalization stores each fact in an appropriate place and connects related rows by keys. A doctor and their services are one-to-many: one doctor can have multiple service rows. Each appointment links one doctor and one patient.

**Comparison with the current MongoDB design:** MongoDB embeds a doctor's `charges` array inside the doctor document. In the proposed MySQL model, `doctor_services` holds one service per row. This makes it natural to search or report on services and fees across doctors, while requiring a join to show the full doctor profile and its services.

Avoid putting comma-separated service names into one column. That makes it hard to validate, search, update, and index individual services.

## 6. SQL CRUD examples

Create a doctor service:

```sql
INSERT INTO doctor_services (doctor_id, service_name, fee)
VALUES (?, ?, ?);
```

The question marks are parameter placeholders. In Node.js, use a MySQL driver such as `mysql2` with prepared/parameterized queries. Never build SQL by concatenating user input into the query string.

Read appointments for a patient with doctor details:

```sql
SELECT a.id, a.appointment_date, a.time_slot, a.service_name,
       a.fee_at_booking, a.status,
       d.name AS doctor_name, d.specialization, d.clinic_name, d.city
FROM appointments AS a
JOIN doctors AS d ON d.id = a.doctor_id
WHERE a.patient_id = ?
ORDER BY a.appointment_date, a.time_slot;
```

This join plays a similar role to Mongoose `.populate()`: the database returns related information in one SQL query. SQL joins are a core relational feature; indexes on join and filter columns help as data grows.

Update a booking status, scoped to its owner:

```sql
UPDATE appointments
SET status = ?
WHERE id = ? AND doctor_id = ?;
```

The API should verify the doctor's identity and allowed status transition before running the update. A safe query predicate is an additional guard, not a replacement for authorization.

## 7. Transactions and booking correctness

A transaction groups SQL operations so they commit together or roll back together. This is useful when one product action changes multiple related records.

For booking a doctor's appointment slot, first define whether a doctor can have only one active booking per slot. A simple unique index on `(doctor_id, appointment_date, time_slot)` would also prevent rebooking a slot after cancellation, so that may not match the rule. One design is a separate `appointment_slots` table with a unique slot key and an appointment/status relationship; another is to keep the unique key and free the slot by deleting or updating a dedicated slot claim. Pick a design that enforces the real business rule under concurrent requests.

An illustrative transaction flow in Node.js (pseudocode):

```js
const connection = await pool.getConnection();
try {
  await connection.beginTransaction();
  // Read the doctor's service fee inside the transaction.
  // Validate the slot is available and claim it with a uniqueness constraint.
  // Insert the appointment using the server-read fee, not a client-supplied fee.
  await connection.commit();
} catch (error) {
  await connection.rollback();
  throw error;
} finally {
  connection.release();
}
```

Transactions need error handling and short, well-scoped work. The database constraints remain important because two requests can reach the API at the same time.

## 8. Reliable daily queue numbers

The current MongoDB token controller checks a patient's existing token, counts today's tokens, and inserts the next number in separate operations. Concurrent requests can race. The proposed MySQL `tokens` table has unique constraints that prevent duplicate patient/day tokens and duplicate token numbers for a doctor/day, but a safe number allocator is still needed.

One approach is a counter table keyed by doctor and date. Inside a transaction, create the counter row if absent, lock it with `SELECT ... FOR UPDATE`, increment it, then insert the token using that number. If insertion fails, roll back. The lock serializes allocation for that doctor's daily queue. Handle duplicate patient/day attempts using the unique key and return the existing token or a useful error.

This is a conceptual design, not code currently in the repository. Test the behavior with simultaneous requests before deploying a queue system.

## 9. Indexes and query planning

An index helps MySQL find rows for common filters, joins, and ordering. A composite index's column order matters: an index beginning with `patient_id` can help queries that filter by patient, while it may not help a query filtering only by date.

The example schema includes indexes for appointments by patient/date and doctor/date, and tokens by doctor/date/status/token number. These align with common screens in this app: patient appointment history, doctor's schedule, and today's ordered queue.

Indexes use storage and add work to inserts and updates. Use `EXPLAIN` with representative queries and real data volumes to inspect the query plan; don't add indexes without a query need.

## 10. Schema migrations

A relational schema is explicitly defined and changes over time through migrations. A migration might add a nullable column, backfill it, and later make it required. Keep schema changes versioned and deploy them in a planned order so old and new application versions can operate safely during rollout.

**For this project:** migrating from MongoDB would require exporting users, doctors, charges, appointments, and tokens; mapping ObjectIds to new numeric IDs; preserving references; converting date strings; validating duplicates against new unique constraints; and updating the backend's data-access code. Do not switch only the connection string: Mongoose models and MongoDB query syntax would need to be replaced with SQL queries and a MySQL driver or ORM.

## 11. Searching the doctor directory

The current `doctorController.js` filters doctors by district, specialization, maximum fee, and text search. With MySQL, a relational query could join services to filter by maximum fee:

```sql
SELECT DISTINCT d.id, d.name, d.specialization, d.clinic_name, d.city
FROM doctors AS d
LEFT JOIN doctor_services AS s ON s.doctor_id = d.id
WHERE (? IS NULL OR d.district = ?)
  AND (? IS NULL OR d.specialization = ?)
  AND (? IS NULL OR s.fee <= ?)
ORDER BY d.created_at DESC
LIMIT ? OFFSET ?;
```

This demonstrates the shape, not a ready-made query. Build optional filters safely in the backend, validate pagination values, and parameterize every user-provided value. A `%search%` `LIKE` filter may need a full-text index or a dedicated search service at scale; a normal B-tree index usually cannot efficiently serve a leading-wildcard search.

## 12. MySQL and MongoDB comparison for this app

| Topic | Current MongoDB + Mongoose | Possible MySQL design |
| --- | --- | --- |
| Main structure | BSON documents in collections | Rows in tables with columns |
| Doctor services | Embedded `charges` array | `doctor_services` child table, joined when needed |
| Booking relation | ObjectId references; Mongoose populate | Foreign keys and SQL joins |
| Data shape | Flexible documents, with Mongoose validation | Explicit columns, types, and database constraints |
| Integrity | App logic and indexes; no relational FK enforcement | Foreign keys, unique constraints, checks, and transactions |
| Query style | Mongoose filters and MongoDB operators | SQL filters, joins, grouping, and transactions |
| Best fit in this app | Nested profile fields and evolving document shapes | Strong relational constraints, reporting, and multi-table consistency |

Neither database is automatically better. A good choice depends on access patterns, consistency rules, reporting needs, scale, team experience, and operations.

## 13. Authentication and other services

Firebase Authentication remains separate from the database choice. The API can verify Firebase ID tokens and use the Firebase UID as a unique key in either MongoDB or MySQL. Cloudinary can continue storing profile-photo files; the SQL `doctors.photo_url` column would store the URL. Changing to MySQL does not require replacing either service.

## 14. Follow the current code, then imagine the SQL version

1. Read `backend/config/db.js` and `backend/models/*.js` to see the current MongoDB setup.
2. Read `backend/controllers/appointmentController.js` and compare `.populate()` with the SQL `JOIN` example above.
3. Read `backend/controllers/doctorController.js` and map its filters to `WHERE` conditions and indexes.
4. Read `backend/controllers/tokenController.js` and identify the operations that need concurrency protection.
5. Trace `frontend/src/components/BookingModal.jsx`: the UI sends a request to the API and would not need to connect directly to MySQL.

## Interview practice: questions with sample answers

### 1. How would you model this app in MySQL?

**Sample answer:** “I would create users, doctors, doctor_services, appointments, and tokens tables. Doctors belong to users; each appointment and token has foreign keys to a doctor and patient. I would use primary keys, foreign keys, unique constraints for identifiers, and indexes based on the app's lookup patterns.”

### 2. How does a MySQL join compare with Mongoose `.populate()`?

**Sample answer:** “Both let the API return related data. Populate follows a MongoDB reference through Mongoose and usually performs additional database work. A SQL join combines matching rows in a query, with the database using its query planner and indexes. I would select only the columns the screen needs.”

### 3. Why put doctor services in their own table?

**Sample answer:** “A doctor can have many services, and each service has its own fee. A child table represents that one-to-many relationship cleanly and makes it straightforward to search by service or fee. MongoDB currently embeds charges because they are a small part of the profile and often read with it.”

### 4. What do foreign keys provide?

**Sample answer:** “A foreign key makes the database reject a row that points to a missing parent row. For example, an appointment cannot reference a nonexistent doctor. It helps keep relationships consistent, but I still need API authorization and a deliberate delete policy.”

### 5. How would you prevent two people booking the same appointment slot?

**Sample answer:** “I would define what counts as an occupied slot and enforce it with a database uniqueness rule or a separate slot-claim table. I would perform the claim and appointment insert in a transaction, handle the unique-conflict error, and derive the fee from the server's current service record. A check-then-insert in application code alone can race.”

### 6. How would you allocate queue numbers safely?

**Sample answer:** “I would keep a counter for each doctor and date. In a transaction, I would lock that counter row with `SELECT ... FOR UPDATE`, increment it, and insert the token. Unique constraints would enforce one token per patient/day and one token number per doctor/day. Concurrent requests would then receive distinct numbers or a clear duplicate response.”

### 7. Why use `DECIMAL` for a fee instead of `FLOAT`?

**Sample answer:** “Fees are currency values, so exact decimal arithmetic is preferable to binary floating-point approximations. I would use a suitable `DECIMAL` precision and handle formatting and currency rules in the application.”

### 8. When would MySQL be a good choice over the current MongoDB setup?

**Sample answer:** “If the application needs strong relational constraints, reporting across appointments and providers, or multi-table transactions, MySQL may be a good fit. MongoDB fits the current nested profile shape naturally. The choice should follow the data relationships, query patterns, integrity needs, and team's ability to operate the database.”

### 9. What would changing this project to MySQL involve?

**Sample answer:** “It would involve designing and migrating the schema and data, mapping MongoDB ObjectIds to SQL keys, preserving references, converting dates and optional fields, adding constraints and indexes, and rewriting the backend data layer from Mongoose queries to parameterized SQL or an ORM. The frontend could continue calling the same API if its request and response contracts remain stable.”

### 10. How do you avoid SQL injection?

**Sample answer:** “Use parameterized queries or prepared statements so user input is treated as values, not executable SQL. Validate types and allowed values as well. Never concatenate raw search text or IDs into SQL strings.”

## Readiness check

You are in good shape for entry-level MySQL interview questions about this project when you can explain tables and relationships, primary and foreign keys, joins, indexes, transactions, parameterized queries, and why the current MongoDB embedding would map to a child table. Practice distinguishing the current implementation from the proposed MySQL design.

## Glossary

- **Database:** a container for related tables and other objects.
- **Table:** a named set of rows with defined columns.
- **Row:** one record in a table.
- **Primary key:** a column or set of columns that uniquely identifies a row.
- **Foreign key:** a constraint linking a row to a valid row in another table.
- **Join:** a query operation that combines related rows from tables.
- **Normalization:** organizing facts into related tables to reduce duplication and update anomalies.
- **Index:** a structure that can speed up supported queries and enforce uniqueness.
- **Transaction:** a group of operations committed together or rolled back together.
- **Migration:** a versioned change to a database schema or data.
- **Prepared statement:** a query with separately supplied parameters, helping keep input from changing SQL structure.
