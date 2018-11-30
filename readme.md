Copyright 2017 SEAS Education, Inc.

# DataMover v3

This repository contains the SEAS DataMover v3. This utility can read import files and process a series of transformations on them, including the ability to write the records to a SQL Server database.

Configuration of the utility is achieved via a JSON configuration file. Each configuration file, also known as a "directive", contains the information required to locate the file(s) to process, define the transformation pipeline, direct the information to a specific database, and notify interested parties of the completion of the directive.

### Execution
To execute a directive, pass the path to the configuration file to the DataMover v3 executable:
```
dm.exe "..\configurations\directive1.json" "..\configurations\directive2.json"
```
As demonstrated, multiple directives can be passed (separated by spaces), and they will be processed in order.

### Configuration
Directive configuration is done exclusively via JSON.
Notes about JSON:

- Comments are not natively supported. The library used by DMv3 allows them, but beware.
- JSON does not support multi-line values. Use the `\n` escape sequence to add a line break.
- Backslash (`\`) starts an escape sequence. Use the `\\` escape sequence to add a backslash.

The following is the simplest directive that can possibly work.
```json
{
    "database": {
        "server": "test.sql.seasweb.net",
        "name": "ARStandard"
    },
    "file": {
        "folder": "\\\\Mac\\Home\\Downloads",
        "filespec": "schools.csv",
        "archive": "\\Mac\\Home\\Archive"
    },
    "reader": {
        "delimiter": ",",
        "detectColumnCountChanges": true,
        "hasHeaderRecord": true,
        "ignoreBlankLines": true,
        "quote": "\"",
        "ignoreQuotes": false,
        "skipEmptyRecords": true,
        "throwOnBadData": true,
        "trimFields": true,
        "trimHeaders": true
    },
    "pipeline": [],
    "notify": {
        "subject": "SEAS School Import",
        "from": "imports@seasweb.net",
        "smtp": "smtp.gmail.com",
        "to": [
            "john.doe@seaseducation.com",
            "some.teacher@school.edu"
        ]
    }
}
```
This is effectively a directive that will read a file and do nothing with it. The elements that can be included in the pipeline array will be discussed in the pipeline section below.

#### Database
The `database` element directs DMv3 to use a specific SQL Server database for any transformations that require connectivity. 
It does _not_ support connections to multiple databases.

The arguments for the `database` element are:

- `server`: *Required.* The value of this element is the name of a Windows Environment Variable (`Control Panel -> System -> Advanced System Settings -> Environment Variables`) that contains a SQL Server connection string for the database server. When in doubt, use a `System variable` instead of a user-specific one.
- `name`: *Required.* Name of the database on the defined server.

The `database` element is required.

#### File
The `file` element tells DMv3 how to locate the file(s) to process.

The arguments for the `file` element are:

- `folder`: *Required.* Path to the folder in which DMv3 will search for files. Note that backslashes have to be escaped.
- `filespec`: *Required.* File specification for the file(s) to read. This can be as simple as a file name, but can also contain wildcards (`*` and `?`).
- `archive`: *Optional.* When specified, DMv3 will zip the file(s) it read with a unique name and place them in this path.

The `file` element is required.

#### Reader
The `reader` element expresses the configuration of the file being read. The example above is for a standard CSV file.

The arguments for the `reader` element are:

- `delimiter`: *Optional.* The field delimiter for the file, typically either `,` (comma) or `\t` (tab). The default is `,` (comma).
- `detectColumnCountChanges`: *Optional.* When true, will cause file reading to fail if a record contains an unexpected number of columns. The default is `false`, but `true` is recommended.
- `hasHeaderRecord`: *Optional.* Whether the file has a header record. The default is `true`. When true, fields will be named by the header values. When false, they will be named `Field1`, `Field2`, `Field3`, etc.
- `ignoreBlankLines`: *Optional.* Whether if blank lines should be ignored when reading. The default is `true`.
- `quote`: *Optional.* The character used to quote fields. The default is `"` (double-quote).
- `ignoreQuotes`: *Optional.* Whether quotes should be ingored when parsing and treated like any other character. The default is `false`.
- `skipEmptyRecords`: *Optional.* Whether the reader should skip records containing all empty values. The default is `false`, but `true` is recommended.
- `throwOnBadData`: *Optional.* Whether an exception should be thrown when bad field data is detected. The default is `false`, but `true` is recommended.
- `trimFields`: *Optional.* Whether to trim trailing whitespace from fields. The default is `false`. There is a transformation for trimming both leading and trailing values, so this is may be inconsequential.
- `trimHeaders`: *Optional.* Whether to trim trailing whitespace from header names. The default is `false`.

The `reader` element is required, but all its arguments are optional, as they have default values.

#### Notify
The `notify` element configures notifications about the import.

The arguments for the `notify` element are:

- `subject`: *Required.* Subject of the email notification. This is also the name of the import in the import history.
- `from`: *Required.* Email address from which to send the notification. There must also be a corresponding Windows Environment Varible of the same name containing the password for the email account.
- `smtp`: *Required.* The SMTP server to use for sending the notification email.
- `to`: *Required.* The collection of email addresses that should receive the notification. If this value is an empty array (`[]`), no message will be sent.

The `notify` element is required.

#### Pipeline
The `pipeline` element contains the collection of transformations to execute on the records.

Notes:

- Transformations will always be processed in the order they appear in the collection.
- Each record is processed through the pipeline in its own database transaction, allowing the entire record to be rolled-back in the event of a fatal error.
- Specific transformations (notably the `seasToken` transformation) are executed outside of the transaction to avoid deadlocks.
- All transformations require the `type` argument, which defines which transformation to perform.

Transformations are capable of producing errors. Most allow you to specify the level of the error that will be generated should the transformation fail.

Errors:

- `fatal`: Processing of the record stops, the record transaction is rolled back, and a fatal error message is generated for the notification.
- `warning`: Processing of the record continues, but any fields that failed to transform are removed. An error message will be generated and included in the notification.
- `silent`: Identical to the `warning` level, but error messages in the notification are suppressed (but kept in the import history).
- `ignore`: A special case that functions identically to a `fatal` error, but generates no error message at all. This was created for use with certain CLASS imports where the data set is not filtered.

##### Default
```json
{
    "type": "default",
    "fields": {
        "HEADER_FIELD_1": "field value1",
        "HEADER_FIELD_2": "field value2"
    }
}
```
The `default` transformation adds a default value for a field or fields, optionally creating new field(s) in the record.

Elements:

- `fields`: *Required.* This is a dictionary that specifies the target fields on the left, and the values to use for the default on the right. If the field indicated by the header is not present in the data, it will be added with the default value. If the field is present, the default value will be set only if the incoming value is null.

##### Copy
```json
{
    "type": "copy",
    "error": "warning",
    "fields": {
        "TARGET_FIELD_1": "SOURCE_FIELD_1",
        "TARGET_FIELD_2": "SOURCE_FIELD_2"
    }
}
```
The `copy` transformation copies the value of one field to another, optionally creating new field(s) in the record.

Elements:

- `error`: *Optional.* The default is `warning`. Occurs when the source field is not present in the record.
- `fields`: *Required.* This is a dictionary that specifies the target fields on the left, and the source fields on the right. If the target field is not present in the data, it will be added. If the target field is present, it will be overwritten. If the source field is not present, an error will be generated.

##### Map
```json
{
    "type": "map",
    "error": "warning",
    "required": true,
    "caseSensitive": false,
    "fields": [
        "MAPPED_FIELD_1",
        "MAPPED_FIELD_2"
    ],
    "values": {
        "": null,
        "INCOMING_VALUE": "MAPPED_VALUE"
    }
}
```
The `map` transformation exchanges defined incoming values with another value, so that incoming values can be "mapped" to values understood by the target system.

Elements:

- `error`: *Optional.* The default is `warning`. Occurs when `required` is `true` and the incoming value does not have a defined mapping.
- `required`: *Optional.* The default is `true`. When true, incoming values that have no defined mapping will generate an error. When false, the incoming value is passed along without modification nor error.
- `caseSensitive`: *Optional.* The default is `false`. When true, the incoming value must be matched by a case-sensitive ordinal comparison. When false, the ordinal comparison is case-insensitive.
- `fields`: *Required.* Array of the field names that participate in the mapping. Fields that aren't present in the data will not be processed.
- `values`: *Required.* This is a dictionary that specifies the incoming values on the left, and the values to convert them to on the right.

##### Lookup
```json
{
    "type": "lookup",
    "error": "warning",
    "required": true,
    "cache": true,
    "sql": "select id from someTable where code = @value;"
    "fields": [
        "LOOKUP_FIELD_1",
        "LOOKUP_FIELD_2"
    ]
}
```
The `lookup` transformation exchanges incoming values with the result of a scalar query. The incoming value is passed to the query as the `@value` parameter.

Elements:

- `error`: *Optional.* The default is `warning`. Occurs when `required` is `true` and the query for the lookup yields no results. Also occurs if the query yields multiple results (whether required or not).
- `required`: *Optional.* The default is `true`. When true, incoming values that do not result in exactly one match will generate an error. When false, incoming values that match zero records are passed along without modification nor error.
- `cache`: *Optional.* The default is `true`. When true, caches the results of the query per-parameter to avoid repetetive queries yielding the same result. Set this value to false if the total number of distinct lookup matches is expected to be large (e.g., greater than 1000 distinct matches).
- `sql`: *Required.* The SQL query to execute. The `@value` parameter can be used to pass the incoming value to the query. The query should be crafted to return a scalar result.
- `fields`: *Required.* Array of field names that participate in the lookup. Fields that aren't present in the data will not be processed.

##### PasswordV2
```json
{
    "type": "passwordV2",
    "error": "fatal",
    "fields": [
        "PASSWORD_FIELD"
    ]
}
```
The `passwordV2` transformation replaces the incoming field with a password hash compatible with ASP.NET Identity v2.x using the following algorithm: 
`PBKDF2 with HMAC-SHA1, 128-bit salt, 256-bit subkey, 1000 iterations.`
This allows the transformation to create passwords for CLASS users.

Elements:

- `error`: *Optional.* The default is `fatal`. This is the error that will be generated when the incoming value is either null or is not a string value.
- `fields`: *Required.* Array of field names whose values will be hashed and replaced. Fields that aren't present in the data will not be processed.

##### Query
```json
{
    "type": "query",
    "error": "warning",
    "cache": false,
    "sql": "select queryField1, queryField2 from someTable where code = @paramField1 and discriminator = @paramField2;",
    "message": "The query did not yeild a result.",
    "params": {
        "paramField1": "CODE_FIELD",
        "paramField2": "DISCRIMINATOR_FIELD"
    },
    "fields": {
        "CODE_ID_FIELD": "queryField1",
        "OTHER_FIELD": "queryField2"
    }
}
```
The `query` transformation executes a query using multiple named parameters and can output multiple query result fields to the output.

Elements:

- `error`: *Optional.* The default is `warning`. This is the error that will be generated under the following conditions:
    - Any of the parameter fields are not present in the data.
    - Any of the query output fields are not present in the query result.
    - No results were returned by the query and `required` is true.
    - Multiple results were returned by the query.
- `cache`: *Optional.* The default is `false`. When true, caches the results of the query per-parameter to avoid repetetive queries yielding the same result. Set this value to true if the total number of distinct query matches is expected to be small (e.g., less than 1000 distinct matches).
- `sql`: *Required.* The SQL query to execute. Parameters can be named (and mapped via the `params` element). Result columns can also be output to data fields via the `fields` element. The query should be crafted to return a single row, but can return multiple columns.
- `params`: *Required.* This is a dictionary that specifies the query parameter names on the left, and the incoming field names to submit as parameter values on the right. If any of the incoming fields is not present in the data, and error will be generated.
- `fields`: *Required.* This is a dictionary that specifies output field names on the left, and the names of the query columns containing the output values on the right. If the output field is not present in the data, it will be added. If the output field is present, its value will be replaced. In the event of an error, the output fields will be removed from the data. If there is no match and `required` is false, the incoming values will be passed along without modification nor error.
- `message`: *Optional.* Custom error message in the event the query does not return a single result.

##### RandomGUID
```json
{
    "type": "randomGuid",
    "fields": [
        "TARGET_FIELD_1",
        "TARGET_FIELD_2"
    ]
}
```
The `randomGuid` transformation adds a random (version 4) GUID to the data. Each field of each record processed by this transformation will receive a new value.

Elements:

`fields`: *Required.* Array of field names to add a random GUID. If the field is present in the data, its value will be overwritten. If the field is not present, it will be added.

##### Required
```json
{
    "type": "required",
    "error": "fatal",
    "fields": [
        "REQUIRED_FIELD_1",
        "REQUIRED_FIELD_2"
    ]
}
```
The `required` transformation will check all of the specified fields and generate an error if any of the required fields are missing or contain null values.

Elements:

- `error`: *Optional.* The default is `fatal`. This specifies the error to add when a required field is missing or null.
- `fields`: *Required.* Array of field names whose values are required.

##### SEAS Staff-School
```json
{
    "type": "seasStaffSchool",
    "employee": "SEAS_STAFF_ID",
    "purge": false,
    "allSchools": "ALL_SCHOOLS",
    "primary": "PRIMARY_SCHOOL",
    "secondary": [
        "ADDITIONAL_SCHOOL1",
        "ADDITIONAL_SCHOOL2"
    ]
}
```
The `seasStaffSchool` transformation will handle school assignments for staff members in SEAS.

Elements:

- `employee`: *Required.* Field name containing the SEAS `e_id` from the `t_employee` entry for the staff member.
- `purge`: *Optional* The default is `false`. When true, removes any existing school assignments not located in the file. When false, existing assignments will be retained.
- `allSchools`: *Optional.* Field name containing a boolean value that indicates whether to assign the staff member to all schools.
- `primary`: *Required.* Field name containing the SEAS `s_id` from the `t_school` entry for the staff member's primary school.
- `secondary`: *Required.* Array of field names containing the SEAS `s_id` from the `t_school` entry for the staff member's secondary schools.

Here is an example (from FLDuvalCounty) of how to combine this with other transforms to extract a secondary school of "AllSchools" into a boolean field:
```json
{
    "type": "when",
    "expression": "Record[\"ADDITIONAL_SCHOOL\"] is string value && string.Equals( value, \"AllSchools\", System.StringComparison.OrdinalIgnoreCase )",
    "false": [],
    "true": [
        {
            "type": "default",
            "fields": {
                "ALL_SCHOOLS": true
            }
        },
        {
            "type": "boolean",
            "fields": [
                "ALL_SCHOOLS"
            ]
        },
        {
            "type": "map",
            "fields": [
                "ADDITIONAL_SCHOOL"
            ],
            "values": {
                "AllSchools": null
            }
        }
    ]
},
{
    "type": "lookup",
    "sql": "select s_id from t_school where s_schoolNumber = @value;",
    "fields": [
        "PRIMARY_SCHOOL",
        "ADDITIONAL_SCHOOL"
    ]
},
{
    "type": "seasStaffSchool",
    "employee": "SEAS_STAFF_ID",
    "allSchools": "ALL_SCHOOLS",
    "primary": "PRIMARY_SCHOOL",
    "secondary": [
        "ADDITIONAL_SCHOOL"
    ]
}
```

##### SEASToken
Example for getting the token for the student table based on a multi-value key (name/dob):
```json
{
    "type": "seasToken",
    "field": "SEAS_STUDENT_ID",
    "id": "s_id",
    "table": "t_student",
    "key": {
        "s_lastName": "STUDENT_LAST_NAME",
        "s_firstName": "STUDENT_FIRST_NAME",
        "s_birthdate": "STUDENT_DOB"
    }
}
```
Example for getting the token for a detail table based on the student's s_id:
```json
{
    "type": "seasToken",
    "field": "SEAS_STUDENTDETAIL_ID",
    "id": "sd_id",
    "table": "t_studentDetail",
    "key": {
        "sd_studentId": "SEAS_STUDENT_ID"
    }
}
```
The `seasToken` transformation acquires a token for the record from the token table. Using the key, it will first attempt to locate the token for an existing record, and should one not exist, it uses the token table to generate the next token in sequence.

**Notes:**

- This transformation is *not* executed in the same transaction as rest of the record. Even if processing of the record fails, the generated token will not be made available again.
- This transformation will always result in a `fatal` error being generated under the following conditions:
    - No token being returned. This will likely mean that the token table is out-of-sync with the target table.
    - Multiple tokens being returned. This likely means that multiple records in the target table match the key.
    - Any of the key fields not being present in the data.

Elements:

- `field`: *Required.* This is the target field in the data that will receive the token. The field will be created/overwritten.
- `id`: *Required.* Specifies the field in the target table that contains token values (for obtaining an existing token).
- `table`: *Required.* Name of the table for which to return a token.
- `key`: *Required.* This is a dictionary of the database columns on the left, and the incoming value fields to use as parameters for locating an existing record.

##### Timestamp
```json
{
    "type": "timestamp",
    "fields": [
        "TARGET_FIELD_1",
        "TARGET_FIELD_2"
    ]
}
```
The `timestamp` column set the values of the specified fields to the current pipeline execution date/time in UTC. This time will be the moment when the pipeline is constructed, so all records of all fields in the pipeline for the file will receive the same time value.

Elements:

- `fields`: *Required.* Array of the field names to receive the timestamp value. The fields will be created/overwritten.

##### Trim
```json
{
    "type": "trim",
    "fields": [
        "FIELD_1",
        "FIELD_2"
    ]
}
```
The `trim` transformation will remove leading and trailing whitespace from text values.

Elements:

- `fields`: *Required.* Array of field names to trim. An asterisk (`*`) can be used as a field name to signify trimming all fields.

##### Unique
```json
{
    "type": "unique",
    "fields": [
        "KEY_FIELD_1",
        "KEY_FIELD_2"
    ]
}
```
The `unique` transformation attempts to detect when multiple records in the file contain the same key values, and generates a `fatal` error when a duplicate is detected.
**Note:** Duplicate detection is *case-sensitive*.

Elements:

- `fields`: *Required.* Array of the field names to participate in the uniqueness check.

##### Upsert
Example for SEAS:
```json
{
    "type": "upsert",
    "table": "someTable",
    "timestamp": "s_lastUpdated",
    "new": true,
    "key": {
        "s_id": "SEAS_STUDENT_ID"
    },
    "insert": {
        "s_id": "SEAS_STUDENT_ID",
        "s_createdBy": "CREATED_BY",
        "s_createdDate": "CREATED_DATE"
    },
    "upsert": {
        "s_active": "ACTIVE",
        "s_birthDate": "DOB",
        "s_disabilityId1": "PRIMEXCEPTIONALITY",
        "s_disabilityId2": "OTHEREXCEPTIONALITY1",
        "s_disabilityId3": "OTHEREXCEPTIONALITY2",
        "s_disabilityId4": "OTHEREXCEPTIONALITY3",
        "s_disabilityId5": "OTHEREXCEPTIONALITY4",
        "s_disabilityId6": "OTHEREXCEPTIONALITY5",
        "s_disabilityId7": "OTHEREXCEPTIONALITY6",
        "s_disabilityId8": "OTHEREXCEPTIONALITY7",
        "s_firstName": "STUDENT_FIRST_NAME",
        "s_IDNumber": "STUDENTNUMBER",
        "s_ethnicity": "ETHNICITY",
        "s_gender": "GENDER",
        "s_grade": "GRADELEVEL",
        "s_lastName": "STUDENT_LAST_NAME",
        "s_middleName": "STUDENT_MIDDLE_NAME",
        "s_raceId": "RACE1",
        "s_raceId2": "RACE2",
        "s_raceId3": "RACE3",
        "s_raceId4": "RACE4",
        "s_raceId5": "RACE5",
        "s_systemActive": "SYSTEM_ACTIVE"
    }
}
```
Example for CLASS:
```json
{
    "type": "upsert",
    "table": "student",
    "new": true,
    "key": {
        "importedStudentID": "STUDENT_ID",
        "districtId": "CLASS_DISTRICT_ID"
    },
    "insert": {
        "importedStudentID": "STUDENT_ID",
        "districtId": "CLASS_DISTRICT_ID",
        "inactive": "CLASS_DEFAULT_ZERO"
    },
    "upsert": {
        "FirstName": "FIRST_NAME",
        "MiddleName": "MIDDLE_NAME",
        "LastName": "LAST_NAME",
        "GradeLevel": "GRADE",
        "BirthDate": "DOB",
        "StudentDisplayID": "STUDENT_ID"
    },
    "output": {
        "CLASS_STUDENT_ID": "StudentId"
    }
}
```
The `upsert` transformation handles inserting/updating information in the target database table. It will query the table to determine if the record is to be inserted or updated. It will also compare the existing data with the incoming values to generate the minimal update statement required to effect the record change (including skipping the update entirely if there are no differences). It is also capable of outputting results (e.g., obtaining identity value for subsequent transformations).

To generate minimal update statements, the incoming data must be converted to the appropriate data types. Otherwise comparisons will always result in differences that trigger an update.

Elements:

- `table`: *Required.* This is the database table to insert/update with the incoming values.
- `new`: *Optional.* The default is `false`. When true, successful inserts to the table will report the record as an insert in log notifications. This will generally be set to `true` for the most important upsert transformation in the pipeline. When `false`, the upsert is considered an ancillary record and inserts will still be considered an update. E.g., in SEAS, set to true only for the `t_student` table. Inserts into that table would report as an insert, but inserts into secodary tables would simply report as an update to an existing student record.
- `key`: *Required.* This is a dictionary mapping of the database table key, expressed as table column names on the left and the incoming field names that match those columns on the right. This is used to locate the existing record, if one exists.
- `insert`: *Optional.* This is a dictionary mapping of the database fields to be processed only on insert, expressed as table column names on the left and the incoming field names that match those columns on the right. These mappings will be applied only when a record is inserted, which is useful for setting default values or creation information.
- `upsert`: *Optional.* This is a dictionary mapping of the database fields that will be processed in both inserts and updates, expressed as table column names on the left and the incoming field names that match those columns on the right.
- `timestamp`: *Optional.* This is a field in the table to update with the current time (UTC) when the record is either inserted or updated. Note that this will not update if no changes were detected for the record.
- `output`: *Optional.* This is a dictionary mapping of the database fields that will be output from the inserted/updated record, expressed as record field names on the left, and table column names on the right. Note that if no update is performed, the existing values from the database row will be used.

Error levels for the `upsert` transformation cannot be set.

The upsert will generate a validation exception if there is not at least one mapping in either the `insert` or `upsert` elements.

The upsert will generate a `fatal` error in the following conditions:

- Any of the output databse columns are not present in the resulting data. This indicates a bug in the transformation itself, and should not be encountered.
- The `output` element contains mappings, but the table has a trigger defined for the insert/update action.
- Any of the `key` fields are not present in the incoming data.
- The resulting statement generates a database error (e.g., constraint violation).

The upsert will generate a `warning` error in the following conditions:

- The target database column is text and the incoming value exceeds the maximum size of the field. The field will be removed from the data and the upsert will continute processing.

##### Convert Value
```json
{
    "type": "datetime",
    "error": "warning",
    "empty": true,
    "nullable": true,
    "fields": [
        "CONVERT_FIELD_1",
        "CONVERT_FIELD_2"
    ]
}
```
All remaining transformations are for converting string data into another data type suitable for use with the database.

Elements:

- `type`: *Required.* This can be any of the following data types:
    - `boolean`: Converts text values of `true` and `false` to the SQL Server `bit` type.
    - `date`: Converts text values to the SQL Server `date` type.
    - `datetime`: Converts text values to the SQL Server `datetime` type.
    - `integer`: Converts text values to the SQL Server `int` type.
    - `ssn`: Converts text values to formatted Social-Security Numbers. 
- `error`: *Optional.* The default is `warning`. Occurs when the conversion fails, which will also remove the target field(s) from the record.
- `empty`: *Optional.* The default is `true`. When `true`, treats empty strings as a null value.
- `nullable`: *Optional.* The default is `true`. When `true`, allows null values to pass conversion. When `false`, null values will fail conversion.
- `fields`: *Required.* Array of field names that participate in the data conversion.
