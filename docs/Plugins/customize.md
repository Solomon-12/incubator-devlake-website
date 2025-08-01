---
title: "Customize"
description: >
  Customize Plugin
---



## Summary

This plugin provides users the ability to:
- Add/delete columns in domain layer tables
- Insert values to certain columns with data extracted from some raw layer tables
- Import data from CSV files(only `issues`, `issue_commits`, `issue_repo_commits`, `sprints`, `issue_worklogs`, `issue_changelogs`, `qa_apis`, `qa_test_cases` and `qa_test_case_executions` tables are supported)

**NOTE:** The names of columns added via this plugin must start with the prefix `x_`

For now, only the following six types are supported:
- varchar(255)
- text
- bigint
- float
- timestamp
- array

## Sample Request

### Trigger Data Extraction
To extract data, switch to `Advanced Mode` on the first step of creating a Blueprint and paste a JSON config as the following:

The example below demonstrates how to extract status name from the table `_raw_jira_api_issues`:
  1. For non-array types: Extract the status name from the `_raw_jira_api_issues` table and assign it to the `x_test` column in the [issues](/DataModels/DevLakeDomainLayerSchema.md#issues) table.
  2. For array types: Extract the status name from the `_raw_jira_api_issues` table, and create a new [issue_custom_array_fields](/DataModels/DevLakeDomainLayerSchema.md#issue_custom_array_fields) table containing `issue_id`, `field_id`, and `value` columns. This table has a one-to-many relationship with the `issues` table. `issue_id` is the id corresponding to the issue, `x_test` corresponds to the `field_id` column, and the value of `x_test` corresponds to the `value` column.

We leverage the package `https://github.com/tidwall/gjson` to extract value from the JSON. For the extraction syntax, please refer to this [docs](https://github.com/tidwall/gjson/blob/master/SYNTAX.md)

- `table`: domain layer table name
- `rawDataTable`: raw layer table, from which we extract values by json path
- `rawDataParams`: the filter to select records from the raw layer table (**The value should be a string not an object**)
- `mapping`: the extraction rule; the key is the extension field name; the value is json path

```json
[
  [
    {
      "plugin":"customize",
      "options":{
        "transformationRules":[
          {
            "table":"issues", 
            "rawDataTable":"_raw_jira_api_issues", 
            "rawDataParams":"{\"ConnectionId\":1,\"BoardId\":8}", 
            "mapping":{
              "x_test":"fields.status.name" 
            }
          }
        ]
      }
    }
  ]
]
```

You can also trigger data extraction by making a POST request to `/pipelines`.
```shell
curl 'http://localhost:8080/pipelines' \
--header 'Content-Type: application/json' \
--data-raw '
{
    "name": "extract fields",
    "plan": [
        [
            {
                "plugin": "customize",
                "options": {
                    "transformationRules": [
                        {
                            "table": "issues",
                            "rawDataTable": "_raw_jira_api_issues",
                            "rawDataParams": "{\"ConnectionId\":1,\"BoardId\":8}",
                            "mapping": {
                                "x_test": "fields.status.name"
                            }
                        }
                    ]
                }
            }
        ]
    ]
}
'
```

### List Columns
Get all columns of the table `issues`
> GET /plugins/customize/issues/fields

**NOTE** some fields are omitted in the following example
response
```json
[
  {
    "columnName": "id",
    "displayName": "",
    "dataType": "varchar(255)",
    "description": ""
  },
  {
    "columnName": "created_at",
    "displayName": "",
    "dataType": "datetime(3)",
    "description": ""
  },
  {
    "columnName": "x_time",
    "displayName": "time",
    "dataType": "timestamp",
    "description": "test for time"
  },
  {
    "columnName": "x_int",
    "displayName": "bigint",
    "dataType": "bigint",
    "description": "test for int"
  },
  {
    "columnName": "x_float",
    "displayName": "float",
    "dataType": "float",
    "description": "test for float"
  },
  {
    "columnName": "x_text",
    "displayName": "text",
    "dataType": "text",
    "description": "test for text"
  },
  {
    "columnName": "x_varchar",
    "displayName": "varchar",
    "dataType": "varchar(255)",
    "description": "test for varchar"
  }
]

```

### Create a Customized Column
Create a new column `x_abc` with datatype `varchar(255)` for the table `issues`.

The value of `columnName` must start with `x_` and consist of no more than 50 alphanumerics and underscores.
The value of field `dataType` must be one of the following 5 types:
- varchar(255)
- text
- bigint
- float
- timestamp 

> POST /plugins/customize/issues/fields
```json
{
  "columnName": "x_abc",
  "displayName": "ABC",
  "dataType": "varchar(255)",
  "description": "test field"
}
```

### Drop A Column
Drop the column `x_text` of the table `issues`

> DELETE /plugins/customize/issues/fields/x_test

### Upload `issues.csv` file

> POST /plugins/customize/csvfiles/issues.csv

The HTTP `Content-Type` must be `multipart/form-data`, and the form should have four fields:

- `file`: The CSV file to upload
- `boardId`: It will be written to the `id` field of the `boards` table, the `board_id` field of `board_issues`, and the `_raw_data_params` field of `issues`
- `boardName`: It will be written to the `name` field of the `boards` table
- `incremental`: Whether to import incrementally (default: false)

Upload a CSV file and import it to the `issues` table via this API. There should be no extra fields in the file except the `labels` and `sprint_ids` fields, and if the field value is `NULL`, it should be `NULL` in the CSV instead of the empty string.

**Note:**
- The `sprint_ids` field should contain comma-separated sprint IDs (e.g. "sprint1,sprint2")
- These values will be automatically written to the `sprint_issues` table during import
DevLake will parse the CSV file and store it in the `issues` table, where the `labels` are stored in the `issue_labels` table. 
If the `boardId` does not appear, a new record will be created in the boards table. The `board_issues` table will be updated at the same time as the import.
The following is an issues.CSV file sample:

|id                           |_raw_data_params     |url                                                                 |icon_url|issue_key|title        |description                      |epic_key|type |status|original_status|story_point|resolution_date|created_date                 |updated_date                 |parent_issue_id|priority|original_estimate_minutes|time_spent_minutes|time_remaining_minutes|creator_id                                           |creator_name|assignee_id                                          |assignee_name|severity|component|lead_time_minutes|original_project|original_type|x_int         |x_time             |x_varchar|x_float|labels              |sprint_ids       |
|-----------------------------|---------------------|--------------------------------------------------------------------|--------|---------|-------------|---------------------------------|--------|-----|------|---------------|-----------|---------------|-----------------------------|-----------------------------|---------------|--------|-------------------------|------------------|----------------------|-----------------------------------------------------|------------|-----------------------------------------------------|-------------|--------|---------|-----------------|----------------|-------------|--------------|-------------------|---------|-------|--------------------|-----------------|
|bitbucket:BitbucketIssue:1:1 |board789             |https://api.bitbucket.org/2.0/repositories/thenicetgp/lake/issues/1 |        |1        |issue test   |bitbucket issues test for devlake|        |issue|TODO  |new            |0          |NULL           |2022-07-17 07:15:55.959+00:00|2022-07-17 09:11:42.656+00:00|               |major   |0                        |0                 |0                     |bitbucket:BitbucketAccount:1:62abf394192edb006fa0e8cf|tgp         |bitbucket:BitbucketAccount:1:62abf394192edb006fa0e8cf|tgp          |        |         |NULL             |NULL            |NULL         |10            |2022-09-15 15:27:56|world    |8      |NULL                |sprint1,sprint2  |
|bitbucket:BitbucketIssue:1:10|board789             |https://api.bitbucket.org/2.0/repositories/thenicetgp/lake/issues/10|        |10       |issue test007|issue test007                    |        |issue|TODO  |new            |0          |NULL           |2022-08-12 13:43:00.783+00:00|2022-08-12 13:43:00.783+00:00|               |trivial |0                        |0                 |0                     |bitbucket:BitbucketAccount:1:62abf394192edb006fa0e8cf|tgp         |bitbucket:BitbucketAccount:1:62abf394192edb006fa0e8cf|tgp          |        |         |NULL             |NULL            |NULL         |30            |2022-09-15 15:27:56|abc      |2456790|hello worlds        |
|bitbucket:BitbucketIssue:1:13|board789             |https://api.bitbucket.org/2.0/repositories/thenicetgp/lake/issues/13|        |13       |issue test010|issue test010                    |        |issue|TODO  |new            |0          |NULL           |2022-08-12 13:44:46.508+00:00|2022-08-12 13:44:46.508+00:00|               |critical|0                        |0                 |0                     |bitbucket:BitbucketAccount:1:62abf394192edb006fa0e8cf|tgp         |                                                     |             |        |         |NULL             |NULL            |NULL         |1             |2022-09-15 15:27:56|NULL     |0.00014|NULL                |
|bitbucket:BitbucketIssue:1:14|board789             |https://api.bitbucket.org/2.0/repositories/thenicetgp/lake/issues/14|        |14       |issue test011|issue test011                    |        |issue|TODO  |new            |0          |NULL           |2022-08-12 13:45:12.810+00:00|2022-08-12 13:45:12.810+00:00|               |blocker |0                        |0                 |0                     |bitbucket:BitbucketAccount:1:62abf394192edb006fa0e8cf|tgp         |bitbucket:BitbucketAccount:1:62abf394192edb006fa0e8cf|tgp          |        |         |NULL             |NULL            |NULL         |41534568464351|2022-09-15 15:27:56|NULL     |NULL   |label1,label2,label3|


### Upload `issue_commits.csv` file

> POST /plugins/customize/csvfiles/issue_commits.csv

The `Content-Type` should be  `multipart/form-data`, and the form should have two fields:

- `file`: The CSV file
- `boardId`: It will be written to the `_raw_data_params` field of `issue_commits`

The following is an issue_commits.CSV file sample:

|issue_id              |commit_sha                              |
|----------------------|----------------------------------------|
|jira:JiraIssue:1:10063|8748a066cbaf67b15e86f2c636f9931347e987cf|
|jira:JiraIssue:1:10064|e6bde456807818c5c78d7b265964d6d48b653af6|
|jira:JiraIssue:1:10065|8f91020bcf684c6ad07adfafa3d8a2f826686c42|
|jira:JiraIssue:1:10066|0dfe2e9ed88ad4e27f825d9b67d4d56ac983c5ef|
|jira:JiraIssue:1:10145|07aa2ebed68e286dc51a7e0082031196a6135f74|
|jira:JiraIssue:1:10145|d70d6687e06304d9b6e0cb32b3f8c0f0928400f7|
|jira:JiraIssue:1:10159|d28785ff09229ac9e3c6734f0c97466ab00eb4da|
|jira:JiraIssue:1:10202|0ab12c4d4064003602edceed900d1456b6209894|
|jira:JiraIssue:1:10203|980e9fe7bc3e22a0409f7241a024eaf9c53680dd|

### Upload `issue_repo_commits.csv` file

> POST /plugins/customize/csvfiles/issue_repo_commits.csv

#### API Description
Upload issue_repo_commits.csv file to import issue-repo commit relationships into DevLake.

#### Request
- **Content-Type**: multipart/form-data
- **Parameters**:
 - `boardId` (required): The ID of the board
 - `incremental` (optional): Whether to import incrementally (default: false)
 - `file` (required): The CSV file to upload

#### Responses
- **200**: Success
- **400**: Bad Request
- **500**: Internal Server Error

#### CSV Format
The CSV file should contain the following columns:

|issue_id              |repo_url                                  |commit_sha                              |host        |namespace |repo_name|
|----------------------|------------------------------------------|----------------------------------------|------------|----------|---------|
|jira:JiraIssue:1:10063|https://github.com/apache/devlake.git     |8748a066cbaf67b15e86f2c636f9931347e987cf|github.com  |apache    |devlake  |
|jira:JiraIssue:1:10064|https://github.com/apache/devlake.git     |e6bde456807818c5c78d7b265964d6d48b653af6|github.com  |apache    |devlake  |

### Upload `sprints.csv` file

> POST /plugins/customize/csvfiles/sprints.csv

The `Content-Type` should be `multipart/form-data`, and the form should have three fields:

- `file`: The CSV file to upload
- `boardId`: The ID of the board
- `incremental`: Whether to import incrementally (default: false)

The following is a sprints.CSV file sample:

|id    |url                                  |status    |name       |start_date          |ended_date            |completed_date        |
|------|-------------------------------------|----------|-----------|--------------------|----------------------|----------------------|
|sprint1|https://jira.example.com/sprint/1   |ACTIVE    |Sprint 1   |2022-01-01 00:00:00 |2022-01-14 00:00:00   |NULL                  |
|sprint2|https://jira.example.com/sprint/2   |CLOSED    |Sprint 2   |2022-01-15 00:00:00 |2022-01-28 00:00:00   |2022-01-28 12:00:00   |

### Upload `issue_worklogs.csv` file

> POST /plugins/customize/csvfiles/issue_worklogs.csv

The `Content-Type` should be `multipart/form-data`, and the form should have three fields:

- `file`: The CSV file to upload
- `boardId`: The ID of the board
- `incremental`: Whether to import incrementally (default: false)

The following is an issue_worklogs.CSV file sample:

|id    |issue_id              |author_name (will create account record)|time_spent_minutes|started_date         |logged_date          |comment               |
|------|----------------------|------------|------------------|---------------------|---------------------|----------------------|
|1     |jira:JiraIssue:1:10063|John Doe    |120               |2022-01-01 09:30:00  |2022-01-01 10:00:00 |Initial investigation |
|2     |jira:JiraIssue:1:10064|Jane Smith  |60                |2022-01-02 14:00:00  |2022-01-02 14:30:00 |Bug fixing            |

### Upload `issue_changelogs.csv` file

> POST /plugins/customize/csvfiles/issue_changelogs.csv

The `Content-Type` should be `multipart/form-data`, and the form should have three fields:

- `file`: The CSV file to upload
- `boardId`: The ID of the board
- `incremental`: Whether to import incrementally (default: false)

The following is an issue_changelogs.CSV file sample:

|id    |issue_id              |author_name (will create account record)|field_name |original_from_value|original_to_value|created_date          |
|------|----------------------|------------|-----------|-------------------|-----------------|----------------------|
|1     |jira:JiraIssue:1:10063|John Doe    |status     |Open               |In Progress      |2022-01-01 09:00:00  |
|2     |jira:JiraIssue:1:10063|John Doe    |status     |In Progress        |Done             |2022-01-03 17:00:00  |

### Upload `qa_apis.csv` file

> POST /plugins/customize/csvfiles/qa_apis.csv

The HTTP `Content-Type` must be `multipart/form-data`, and the form should have three fields:

- `file`: The CSV file
- `qaProjectId`: It will be used as qa_project_id for the imported data
- `incremental`: Boolean value indicating whether this is an incremental update (true/false)

Upload a CSV file and import it to the `qa_apis` table via this API. The following fields are required:

| Field Name       | Data Type     | Description                     |
|------------------|---------------|---------------------------------|
| id               | varchar(500)  | Unique identifier for the API   |
| name             | varchar(255)  | API name                        |
| path             | varchar(255)  | API path                        |
| method           | varchar(255)   | HTTP method (GET, POST, etc.)   |
| create_time      | timestamp     | Creation timestamp              |
| creator_name     | varchar(255)  | Creator name (will create a record in accounts table and write creator_id to qa_apis table) |

#### qa_apis.csv sample:
|id                  |name          |path              |method|create_time              |creator_name|qa_project_id|
|--------------------|--------------|------------------|------|-------------------------|------------|-------------|
|qa:api:1:101        |Login API     |/api/v1/login     |POST  |2025-07-01 10:00:00      |tester1     |project101   |
|qa:api:1:102        |User Info API |/api/v1/user/{id} |GET   |2025-07-01 11:30:00      |tester2     |project101   |
|qa:api:1:103        |Logout API    |/api/v1/logout    |POST  |NULL                     |tester1     |project102   |

### Upload `qa_test_cases.csv` file

> POST /plugins/customize/csvfiles/qa_test_cases.csv

The HTTP `Content-Type` must be `multipart/form-data`, and the form should have four fields:

- `file`: The CSV file
- `qaProjectId`: (max length 500) Will be used as qa_project_id and will create/update a record in qa_projects table
- `qaProjectName`: (max length 255) Will be written to the `name` field of qa_projects table. Together with qaProjectId, this will create a new record in qa_projects table if not exists.
- `incremental`: Boolean value indicating whether this is an incremental update (true/false)

Upload a CSV file and import it to the `qa_test_cases` table via this API. The following fields are required:

| Field Name       | Data Type     | Description                     |
|------------------|---------------|---------------------------------|
| id               | varchar(500)  | Unique test case ID             |
| name             | varchar(255)  | Test case name                  |
| create_time      | timestamp     | Creation timestamp              |
| creator_name     | varchar(255)  | Creator name (will create a record in accounts table and write creator_id to qa_test_cases table) |
| type             | varchar(255)  | Test case type, api or functional |
| qa_api_id        | varchar(255)  | Related API ID, if type is api  |

#### qa_test_cases.csv sample:
|id                  |name               |create_time              |creator_name|type   |qa_api_id      |qa_project_id|
|--------------------|-------------------|-------------------------|------------|-------|---------------|-------------|
|qa:case:1:201       |Login Test         |2025-07-02 09:00:00      |tester1     |api    |qa:api:1:101   |project101   |
|qa:case:1:202       |User Profile Test  |2025-07-02 10:30:00      |tester2     |api    |qa:api:1:102   |project101   |
|qa:case:1:203       |UI Navigation Test |2025-07-02 11:45:00      |tester3     |functional|NULL       |project102   |

### Upload `qa_test_case_executions.csv` file

> POST /plugins/customize/csvfiles/qa_test_case_executions.csv

The HTTP `Content-Type` must be `multipart/form-data`, and the form should have three fields:

- `file`: The CSV file
- `qaProjectId`: It will be used as qa_project_id for the imported data
- `incremental`: Boolean value indicating whether this is an incremental update (true/false)

Upload a CSV file and import it to the `qa_test_case_executions` table via this API. The following fields are required:

| Field Name            | Data Type     | Description                     |
|-----------------------|---------------|---------------------------------|
| id                    | varchar(500)  | Unique execution ID             |
| qa_test_case_id       | varchar(255)  | Related test case ID            |
| create_time           | timestamp     | Creation timestamp              |
| start_time            | timestamp     | Test execution start time       |
| finish_time           | timestamp     | Test execution finish time      |
| creator_name          | varchar(255)  | Creator name (will create a record in accounts table and write creator_id to qa_test_case_executions table) |
| status                | varchar(255)   | Execution status (PENDING, IN_PROGRESS, SUCCESS, FAILED) |

#### qa_test_case_executions.csv sample:
|id                  |qa_test_case_id    |create_time              |start_time              |finish_time             |creator_name|status    |qa_project_id|
|--------------------|-------------------|-------------------------|------------------------|------------------------|------------|----------|-------------|
|qa:exec:1:301       |qa:case:1:201      |2025-07-03 14:00:00      |2025-07-03 14:05:00     |2025-07-03 14:15:00     |tester1     |SUCCESS   |project101   |
|qa:exec:1:302       |qa:case:1:202      |2025-07-03 15:30:00      |2025-07-03 15:35:00     |NULL                    |tester2     |IN_PROGRESS|project101   |
|qa:exec:1:303       |qa:case:1:203      |2025-07-04 09:00:00      |NULL                    |NULL                    |tester3     |PENDING   |project102   |
