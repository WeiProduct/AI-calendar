# API Endpoint Definitions (High-Level) - FocusFlow AI

This document outlines the key RESTful API endpoints for frontend-backend communication in the FocusFlow AI application.

## 1. User Authentication

| Method | Path                | Description                                 | Key Request Body/Query Parameters (Optional)      | Key Response Body (Optional)                                  |
|--------|---------------------|---------------------------------------------|---------------------------------------------------|---------------------------------------------------------------|
| `POST` | `/api/auth/register`| User registration.                          | `email`, `password`, `name` (optional)            | `userID`, `email`, `name`, `accessToken`                        |
| `POST` | `/api/auth/login`   | User login.                                 | `email`, `password`                               | `userID`, `email`, `name`, `accessToken`                        |
| `POST` | `/api/auth/logout`  | User logout.                                | (Requires authentication token)                   | Success/failure message                                       |
| `GET`  | `/api/auth/me`      | Get current authenticated user details.     | (Requires authentication token)                   | `userID`, `email`, `name`, `preferences` (summary or link)      |

## 2. Tasks

| Method | Path                          | Description                                      | Key Request Body/Query Parameters (Optional)                                                                 | Key Response Body (Optional)                                    |
|--------|-------------------------------|--------------------------------------------------|--------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| `POST` | `/api/tasks`                  | Create a new task.                               | `title`, `description` (optional), `dueDate`, `estimatedDuration`, `priority`, `projectID` (optional), `tags` (optional list of IDs) | Full task object                                                |
| `GET`  | `/api/tasks`                  | List tasks.                                      | Filters (query params): `projectID`, `priority`, `dueDateStart`, `dueDateEnd`, `status`, `tags`                | Array of task objects                                           |
| `GET`  | `/api/tasks/{id}`             | Get a specific task by its ID.                   | -                                                                                                            | Full task object                                                |
| `PUT`  | `/api/tasks/{id}`             | Update an existing task.                         | Fields to update (e.g., `title`, `description`, `dueDate`, `priority`, etc.)                                 | Updated full task object                                        |
| `DELETE`| `/api/tasks/{id}`             | Delete a task.                                   | -                                                                                                            | Success/failure message                                         |
| `POST` | `/api/tasks/{id}/complete`    | Mark a task as complete.                         | -                                                                                                            | Updated full task object (with status 'Completed')               |
| `POST` | `/api/tasks/{id}/uncomplete`  | Mark a task as not complete (back to 'Pending' or 'Scheduled'). | -                                                                                                            | Updated full task object (with status reverted)                 |

## 3. AI Scheduler & Calendar View

| Method | Path                                              | Description                                                                          | Key Request Body/Query Parameters (Optional)                                     | Key Response Body (Optional)                                                                 |
|--------|---------------------------------------------------|--------------------------------------------------------------------------------------|----------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|
| `GET`  | `/api/schedule?start_date=YYYY-MM-DD&end_date=YYYY-MM-DD` | Get scheduled tasks and external calendar events for a given date range.           | `start_date`, `end_date`                                                         | Array of scheduled items (tasks with schedule info, external events)                             |
| `POST` | `/api/schedule/reoptimize/today`                  | Trigger AI to re-optimize the schedule for the current day.                          | -                                                                                | Success/failure message, potentially a summary of changes or link to updated schedule view   |
| `POST` | `/api/schedule/reoptimize/week`                   | Trigger AI to re-optimize the schedule for the current week.                         | -                                                                                | Success/failure message, potentially a summary of changes or link to updated schedule view   |
| `PUT`  | `/api/schedule/tasks/{taskId}/move`               | User manually moves/resizes a task block in their calendar view; triggers re-optimization. | `newStartTime`, `newEndTime` (optional, if duration changes)                     | Updated task object and potentially a summary of other re-optimized tasks.                 |

## 4. Projects/Goals

| Method | Path                   | Description                                   | Key Request Body/Query Parameters (Optional) | Key Response Body (Optional)                                  |
|--------|------------------------|-----------------------------------------------|----------------------------------------------|---------------------------------------------------------------|
| `POST` | `/api/projects`        | Create a new project or goal.                 | `name`, `description` (optional)             | Full project object                                           |
| `GET`  | `/api/projects`        | List all projects/goals for the user.         | -                                            | Array of project objects                                      |
| `GET`  | `/api/projects/{id}`   | Get a specific project/goal and its tasks.    | -                                            | Project object with an array of associated task objects       |
| `PUT`  | `/api/projects/{id}`   | Update a project/goal's details.              | `name` (optional), `description` (optional)  | Updated full project object                                   |
| `DELETE`| `/api/projects/{id}`   | Delete a project/goal.                        | -                                            | Success/failure message (consider handling of associated tasks) |

## 5. User Preferences

| Method | Path                         | Description                               | Key Request Body/Query Parameters (Optional)                                      | Key Response Body (Optional)                     |
|--------|------------------------------|-------------------------------------------|-----------------------------------------------------------------------------------|--------------------------------------------------|
| `GET`  | `/api/users/me/preferences`  | Get current user's preferences.           | -                                                                                 | User preferences object (working hours, chunking, buffers, etc.) |
| `PUT`  | `/api/users/me/preferences`  | Update user's preferences.                | Preference fields to update (e.g., `workingHours`, `taskChunkingLimit`, `bufferTime`) | Updated user preferences object                  |

## 6. External Calendar Integration

| Method | Path                                           | Description                                                                     | Key Request Body/Query Parameters (Optional)                               | Key Response Body (Optional)                                                              |
|--------|------------------------------------------------|---------------------------------------------------------------------------------|----------------------------------------------------------------------------|-------------------------------------------------------------------------------------------|
| `POST` | `/api/calendars/sync/authorize`                | Initiate OAuth flow for a calendar provider. Redirects user to provider.        | `provider` (e.g., 'google', 'outlook', 'apple'), `redirect_uri` (for callback) | OAuth authorization URL / redirect instructions                                           |
| `GET`  | `/api/calendars/sync/callback`                 | Callback URL for OAuth provider (not directly called by frontend).              | `code` (from OAuth provider), `state`                                      | - (Handles token exchange, creates connection, redirects user back to frontend)           |
| `GET`  | `/api/calendars/sync/connections`              | List active calendar connections for the user.                                  | -                                                                          | Array of connection objects (`connectionId`, `provider`, `calendarName`, `syncEnabled`, `writeAccess`) |
| `PUT`  | `/api/calendars/sync/connections/{connectionId}` | Update sync settings for a specific connection.                                 | `syncEnabled` (boolean, optional), `writeAccess` (boolean, optional)       | Updated connection object                                                                 |
| `DELETE`| `/api/calendars/sync/connections/{connectionId}` | Remove/disconnect an external calendar connection.                              | -                                                                          | Success/failure message                                                                   |

## 7. Tags (Optional)

| Method | Path                         | Description                                      | Key Request Body/Query Parameters (Optional) | Key Response Body (Optional)        |
|--------|------------------------------|--------------------------------------------------|----------------------------------------------|-------------------------------------|
| `POST` | `/api/tags`                  | Create a new tag.                                | `tagName`                                    | Full tag object (`tagId`, `tagName`)  |
| `GET`  | `/api/tags`                  | List all tags for the current user.              | -                                            | Array of tag objects                |
| `PUT`  | `/api/tasks/{taskId}/tags`   | Associate/update tags for a specific task.       | `tagIds` (array of tag IDs)                  | Updated list of tags for the task   |

*Note: Authentication (e.g., JWT Bearer token in Authorization header) is assumed for all endpoints except `/api/auth/register` and `/api/auth/login`.*
```
