# Data Model / Database Schema

This document proposes a database schema for the key entities in FocusFlow AI.

## 1. Users

Stores information about application users.

| Field         | Type                      | Constraints                                  | Notes                                     |
|---------------|---------------------------|----------------------------------------------|-------------------------------------------|
| UserID        | UUID / BIGINT             | Primary Key, Auto-increment (if BIGINT)      | Universally Unique Identifier             |
| Email         | VARCHAR(255) / String     | Not Null, Unique                             | User's email address for login & comms    |
| PasswordHash  | VARCHAR(255) / String     | Not Null                                     | Hashed password                           |
| Name          | VARCHAR(255) / String     | Nullable                                     | User's full name or display name          |
| CreatedAt     | TIMESTAMP WITH TIME ZONE  | Not Null, Default CURRENT_TIMESTAMP          | Timestamp of user creation                |
| UpdatedAt     | TIMESTAMP WITH TIME ZONE  | Not Null, Default CURRENT_TIMESTAMP          | Timestamp of last user update             |

## 2. Tasks

Stores details about tasks created by users.

| Field             | Type                      | Constraints                                  | Notes                                            |
|-------------------|---------------------------|----------------------------------------------|--------------------------------------------------|
| TaskID            | UUID / BIGINT             | Primary Key, Auto-increment (if BIGINT)      | Unique identifier for the task                   |
| UserID            | UUID / BIGINT             | Not Null, Foreign Key (Users.UserID)         | Owner of the task                                |
| Title             | VARCHAR(255) / String     | Not Null                                     | Concise title of the task                        |
| Description       | TEXT                      | Nullable                                     | Detailed description (supports rich text)        |
| DueDate           | TIMESTAMP WITH TIME ZONE  | Not Null                                     | When the task is due                             |
| EstimatedDuration | INTEGER                   | Not Null                                     | Estimated time in minutes to complete the task   |
| Priority          | ENUM / VARCHAR(10)        | Not Null, (Critical, High, Medium, Low)      | Priority level of the task                       |
| ProjectID         | UUID / BIGINT             | Nullable, Foreign Key (Projects.ProjectID)   | Associated project/goal                          |
| IsRecurring       | BOOLEAN                   | Not Null, Default FALSE                      | True if the task is recurring                    |
| RecurrenceRule    | VARCHAR(255) / String     | Nullable                                     | e.g., RRULE string (iCalendar spec)              |
| Status            | ENUM / VARCHAR(15)        | Not Null, (Pending, Scheduled, Completed, Cancelled), Default 'Pending' | Current status of the task                      |
| CreatedAt         | TIMESTAMP WITH TIME ZONE  | Not Null, Default CURRENT_TIMESTAMP          | Timestamp of task creation                       |
| UpdatedAt         | TIMESTAMP WITH TIME ZONE  | Not Null, Default CURRENT_TIMESTAMP          | Timestamp of last task update                    |
| ParentTaskID      | UUID / BIGINT             | Nullable, Foreign Key (Tasks.TaskID)         | For sub-tasks created by task breakdown          |

## 3. Events

Stores user-defined calendar entries that are not tasks (e.g., imported personal appointments).

| Field           | Type                      | Constraints                                    | Notes                                                                   |
|-----------------|---------------------------|------------------------------------------------|-------------------------------------------------------------------------|
| EventID         | UUID / BIGINT             | Primary Key, Auto-increment (if BIGINT)        | Unique identifier for the event                                         |
| UserID          | UUID / BIGINT             | Not Null, Foreign Key (Users.UserID)           | Owner of the event                                                      |
| Title           | VARCHAR(255) / String     | Not Null                                       | Title of the event                                                      |
| Description     | TEXT                      | Nullable                                       | Detailed description of the event                                       |
| StartTime       | TIMESTAMP WITH TIME ZONE  | Not Null                                       | Start date and time of the event                                        |
| EndTime         | TIMESTAMP WITH TIME ZONE  | Not Null                                       | End date and time of the event                                          |
| IsAllDay        | BOOLEAN                   | Not Null, Default FALSE                        | True if it's an all-day event                                           |
| CalendarSourceID| UUID / BIGINT             | Nullable, Foreign Key (CalendarSyncInfo.CalendarSyncID) | Link to the external calendar source, if imported                     |
| CreatedAt       | TIMESTAMP WITH TIME ZONE  | Not Null, Default CURRENT_TIMESTAMP            | Timestamp of event creation                                             |
| UpdatedAt       | TIMESTAMP WITH TIME ZONE  | Not Null, Default CURRENT_TIMESTAMP            | Timestamp of last event update                                          |

*Note: AI-scheduled task blocks will exist on the calendar but are primarily managed via the Tasks entity and AI engine logic, not as separate 'Event' entries in this table unless it's a direct representation of an external calendar event.*

## 4. Projects (or Goals)

Stores user-defined projects or goals to group tasks.

| Field       | Type                      | Constraints                                  | Notes                                     |
|-------------|---------------------------|----------------------------------------------|-------------------------------------------|
| ProjectID   | UUID / BIGINT             | Primary Key, Auto-increment (if BIGINT)      | Unique identifier for the project/goal    |
| UserID      | UUID / BIGINT             | Not Null, Foreign Key (Users.UserID)         | Owner of the project/goal                 |
| Name        | VARCHAR(255) / String     | Not Null                                     | Name of the project/goal                  |
| Description | TEXT                      | Nullable                                     | Optional description for the project/goal |
| CreatedAt   | TIMESTAMP WITH TIME ZONE  | Not Null, Default CURRENT_TIMESTAMP          | Timestamp of project/goal creation        |
| UpdatedAt   | TIMESTAMP WITH TIME ZONE  | Not Null, Default CURRENT_TIMESTAMP          | Timestamp of last project/goal update     |

## 5. UserPreferences

Stores user-specific settings for application behavior.

| Field                 | Type                      | Constraints                                     | Notes                                                                 |
|-----------------------|---------------------------|-------------------------------------------------|-----------------------------------------------------------------------|
| PreferenceID          | UUID / BIGINT             | Primary Key, Auto-increment (if BIGINT)         | Unique identifier for the preference set                              |
| UserID                | UUID / BIGINT             | Not Null, Unique, Foreign Key (Users.UserID)    | Links to the user these preferences belong to                         |
| WorkingHours          | JSON / TEXT               | Not Null                                        | e.g., `{ "Mon": ["09:00-12:00", "13:00-17:00"], ... }`                 |
| PreferredBlockTypes   | JSON / TEXT               | Nullable                                        | e.g., `{ "Deep Work": ["09:00-11:00"], "Admin": ["14:00-15:00"] }`      |
| TaskChunkingLimit     | INTEGER                   | Nullable, Default 120                           | Max duration in minutes for a single work block (before suggesting break) |
| BufferTime            | INTEGER                   | Nullable, Default 10                            | Default buffer time in minutes around tasks                           |
| AutoScheduleNewTasks  | BOOLEAN                   | Not Null, Default TRUE                          | Whether to automatically schedule new tasks                           |
| CreatedAt             | TIMESTAMP WITH TIME ZONE  | Not Null, Default CURRENT_TIMESTAMP             | Timestamp of preference creation                                      |
| UpdatedAt             | TIMESTAMP WITH TIME ZONE  | Not Null, Default CURRENT_TIMESTAMP             | Timestamp of last preference update                                   |

## 6. CalendarSyncInfo

Stores information for linking and synchronizing with external calendars.

| Field              | Type                      | Constraints                                    | Notes                                                       |
|--------------------|---------------------------|------------------------------------------------|-------------------------------------------------------------|
| CalendarSyncID     | UUID / BIGINT             | Primary Key, Auto-increment (if BIGINT)        | Unique identifier for the sync configuration                |
| UserID             | UUID / BIGINT             | Not Null, Foreign Key (Users.UserID)           | User who owns this calendar link                            |
| CalendarProvider   | ENUM / VARCHAR(20)        | Not Null, (Google, Outlook, Apple)             | e.g., Google, Outlook, Apple                                |
| ExternalCalendarID | VARCHAR(255) / String     | Not Null                                       | ID of the calendar from the external provider               |
| AccessToken        | TEXT                      | Not Null                                       | OAuth Access Token (store securely, e.g., encrypted)         |
| RefreshToken       | TEXT                      | Nullable                                       | OAuth Refresh Token (store securely, e.g., encrypted)        |
| SyncEnabled        | BOOLEAN                   | Not Null, Default TRUE                         | Whether sync is active for this calendar                    |
| LastSyncTime       | TIMESTAMP WITH TIME ZONE  | Nullable                                       | Timestamp of the last successful synchronization            |
| ReadAccess         | BOOLEAN                   | Not Null, Default TRUE                         | Permission to read events from the external calendar        |
| WriteAccess        | BOOLEAN                   | Not Null, Default FALSE                        | Permission to write events to the external calendar         |
| CreatedAt          | TIMESTAMP WITH TIME ZONE  | Not Null, Default CURRENT_TIMESTAMP            | Timestamp of sync info creation                             |
| UpdatedAt          | TIMESTAMP WITH TIME ZONE  | Not Null, Default CURRENT_TIMESTAMP            | Timestamp of last sync info update                          |

## 7. Tags (Optional)

Stores user-defined tags for task organization.

| Field     | Type                      | Constraints                                  | Notes                                     |
|-----------|---------------------------|----------------------------------------------|-------------------------------------------|
| TagID     | UUID / BIGINT             | Primary Key, Auto-increment (if BIGINT)      | Unique identifier for the tag             |
| UserID    | UUID / BIGINT             | Not Null, Foreign Key (Users.UserID)         | User who created the tag                  |
| TagName   | VARCHAR(100) / String     | Not Null                                     | The name of the tag (e.g., "#important")   |
| CreatedAt | TIMESTAMP WITH TIME ZONE  | Not Null, Default CURRENT_TIMESTAMP          | Timestamp of tag creation                 |

*Constraint: (UserID, TagName) should be unique to prevent a user from creating duplicate tags.*

## 8. TaskTags (Join Table)

Links Tasks with Tags in a many-to-many relationship.

| Field  | Type          | Constraints                                 | Notes                       |
|--------|---------------|---------------------------------------------|-----------------------------|
| TaskID | UUID / BIGINT | Not Null, Foreign Key (Tasks.TaskID)        | Part of the composite PK    |
| TagID  | UUID / BIGINT | Not Null, Foreign Key (Tags.TagID)          | Part of the composite PK    |
|        |               | Primary Key (TaskID, TagID)                 | Composite Primary Key       |

This schema provides a foundational structure. Depending on specific RDBMS choices, `UUID` can be implemented as `UUID` type or `VARCHAR(36)`. `ENUM` types can be native ENUMs or `VARCHAR` with check constraints. Timestamps are stored with time zone information for accuracy.
```
