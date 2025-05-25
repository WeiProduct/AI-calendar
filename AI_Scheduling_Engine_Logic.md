# AI Scheduling Engine Logic Document

This document details the algorithms, rules, and heuristics for the FocusFlow AI smart scheduling, re-scheduling, and optimization engine.

## 1. Core Scheduling Algorithm (Automatic Time Blocking)

The primary goal of the Core Scheduling Algorithm is to automatically find and reserve optimal time blocks for tasks within the user's calendar, considering various constraints and preferences.

*   **Inputs:**
    *   Task List: Each task with properties like:
        *   `task_id`: Unique identifier.
        *   `title`: Task description.
        *   `due_date`: Specific date and time.
        *   `priority_level`: Critical, High, Medium, Low.
        *   `estimated_duration`: Time needed (e.g., in minutes).
        *   `dependencies`: List of `task_id`s that must be completed before this task can start.
        *   `project_goal_id`: Associated project or goal.
        *   `type`: User-defined type (e.g., "Deep Work", "Meeting", "Admin").
    *   User Preferences:
        *   `working_hours`: Defined start and end times for each day of the week, including break periods (e.g., lunch).
        *   `preferred_block_types_times`: Mapping of task types to preferred times (e.g., "Deep Work" in Mornings 9 AM-12 PM).
        *   `max_chunk_duration`: Maximum duration for a single work block before a break is suggested.
        *   `buffer_time_before`: Default buffer time before tasks.
        *   `buffer_time_after`: Default buffer time after tasks.
    *   External Calendar Events: List of non-movable events from integrated calendars (Google, Outlook, Apple) with start and end times.
    *   Manually Set Unavailable Blocks: User-defined time blocks within the application where no tasks should be scheduled.

*   **Output:**
    *   A set of scheduled time blocks in the user's calendar, each linked to a specific task. Each block will have a start time and end time.
    *   For tasks that cannot be scheduled, a list of unscheduled tasks with reasons (e.g., "No available slots before deadline," "Conflict with high-priority task").

*   **Logic Details:**

    *   **Task Prioritization (Initial Sorting & Scoring):**
        1.  **Hard Constraints First:** Tasks are initially filtered. Any task whose earliest possible start time (considering dependencies) is already past its due date is flagged as potentially unschedulable or requiring immediate attention.
        2.  **Dependency Resolution:** A directed acyclic graph (DAG) is constructed based on task dependencies. Topological sort is used to determine a valid execution order. Tasks with no dependencies are prioritized earlier in processing.
        3.  **Scoring System:** Each task is assigned a score. Higher scores get scheduling priority. The scoring considers:
            *   **Deadline Proximity (Urgency):** Calculated as `1 / (time_until_deadline_in_hours + 1)`. This gives exponentially higher scores to tasks due very soon.
            *   **Explicit Priority Level (Importance):**
                *   Critical: High fixed score (e.g., 1000 points).
                *   High: Medium-high fixed score (e.g., 500 points).
                *   Medium: Medium fixed score (e.g., 100 points).
                *   Low: Low fixed score (e.g., 10 points).
            *   **Dependency Chain:** Tasks that unblock a larger number of subsequent tasks (or high-priority tasks) may receive a bonus to their score.
        4.  **Primary Sort Key:** Tasks are primarily sorted by their `priority_level` (Critical > High > Medium > Low).
        5.  **Secondary Sort Key (within priority levels):** Tasks are then sorted by their `due_date` (earliest first). If due dates are the same, the calculated urgency score can be used as a tie-breaker.
        6.  **Iterative Scheduling:** The algorithm iterates through the prioritized list of tasks.

    *   **Free Time Slot Identification:**
        1.  **User's Master Availability:** The system starts with the user's defined `working_hours` for each day.
        2.  **Block Out Existing Events:** All non-movable events from integrated calendars and manually set unavailable blocks are overlaid onto the master availability, marking these times as "busy."
        3.  **Consider Buffer Times:** When identifying slots for a specific task, the task's `estimated_duration` is augmented by the user's configured `buffer_time_before` and `buffer_time_after`. This effective duration is used for slot finding.
        4.  **Scan for Contiguous Slots:** The algorithm scans the remaining available time day by day, looking for contiguous free slots that are large enough to accommodate the (effective) duration of the current task being scheduled.

    *   **Constraint Handling:**
        1.  **Working Hours:** Slots are only considered if they fall entirely within the user's defined working hours (respecting daily start/end times and breaks).
        2.  **Existing Events/Unavailable Blocks:** Any slot that overlaps with an existing calendar event or a user-defined unavailable block is invalid.
        3.  **Dependencies:**
            *   When considering Task B, if it depends on Task A, the algorithm ensures that the earliest possible start time for Task B is after the *scheduled end time* of Task A (plus any post-task buffer for A and pre-task buffer for B).
            *   If Task A is not yet scheduled, Task B is temporarily deferred, and Task A is prioritized (its score might be boosted). If Task A cannot be scheduled, Task B also becomes unschedulable due to the dependency.

    *   **Heuristics for "Optimal" Slot Selection:**
        1.  **Earliest Possible Slot (Default):** Generally, the algorithm tries to schedule tasks as early as possible within the available valid slots, especially for higher priority/urgency tasks.
        2.  **Preferred Work Block Type Alignment:**
            *   If a task has a `type` (e.g., "Deep Work") and the user has defined `preferred_block_types_times` (e.g., "Mornings for Deep Work"), slots within these preferred windows are given a higher preference score.
            *   This can be implemented by adding a bonus to the "desirability" of such slots or by filtering slots first by preference and then by other criteria.
        3.  **Grouping Similar Tasks (Future Enhancement):** If multiple tasks of the same `type` or `project_goal_id` are pending, the engine could try to schedule them back-to-back (respecting chunking limits) to promote focused work. This would involve looking ahead in the task queue.
        4.  **Minimizing Gaps:** The engine may slightly prefer slots that fit snugly between already scheduled events to avoid excessive fragmentation of the day, unless this conflicts with other higher-weighted heuristics like preferred times.
        5.  **Task Chunking Adherence:** When selecting a slot, the algorithm considers the `max_chunk_duration`. If a task is longer, it will be handled by the "Deadline Proximity Scheduling & Task Breakdown" logic.

## 2. Dynamic Rescheduling & Re-optimization Algorithm

This algorithm adjusts the existing schedule in response to new inputs or user actions.

*   **Triggers:**
    *   **Automatic:**
        *   **New High-Priority Task Added:** A new task with "Critical" or "High" priority is created.
        *   **Task Duration Change:** User modifies the `estimated_duration` of an existing scheduled task.
        *   **User Manually Moves/Drags a Scheduled Task:** User changes the time/date of a task block directly in the native calendar view.
        *   **External Calendar Event Change:** A new conflicting event appears in a synced external calendar, or an existing event is moved, creating a conflict or opening up new space.
    *   **User-Initiated:**
        *   **"Re-optimize Today's Schedule":** User clicks a button to re-optimize tasks for the remainder of the current day.
        *   **"Re-optimize Week's Schedule":** User clicks a button to re-optimize tasks for the remainder of the current week.

*   **Logic Details:**

    *   **Impact Evaluation:**
        1.  **Identify Affected Tasks:** Determine which tasks are directly and indirectly affected by the trigger.
            *   New High-Priority Task: All lower-priority tasks scheduled after the new task's potential earliest start time are candidates for rescheduling.
            *   Duration Change / Manual Move: The task itself and any tasks scheduled immediately after it, or tasks that now conflict, are affected. Dependencies are also traced.
            *   External Event Change: Any tasks conflicting with the new/modified external event.
        2.  **Assess Severity:** Determine if the change creates a critical conflict (e.g., a high-priority task is now unschedulable) or if it's a minor adjustment.

    *   **Strategies for Re-shuffling:**
        1.  **Minimal Disruption (Default for minor automatic changes):**
            *   Attempt to shift only the directly affected tasks and those immediately following them to the next available valid slots.
            *   Prioritize keeping tasks on the same day if possible.
            *   This is faster but may not be globally optimal.
        2.  **Localized Re-optimization (For task duration changes, manual moves):**
            *   Unschedule the modified task and a small window of surrounding tasks (e.g., tasks within the next X hours or on the same day).
            *   Re-run the Core Scheduling Algorithm for this subset of tasks and the identified time window.
        3.  **Full Re-optimization (For new high-priority tasks, user-initiated full re-optimization):**
            *   Unschedule all pending/scheduled tasks (or tasks within the specified scope: today/week).
            *   Re-run the Core Scheduling Algorithm on the entire set of affected tasks from scratch. This provides the most optimal schedule but can be more computationally intensive.

    *   **User-Initiated Re-optimization Scopes:**
        *   **"Re-optimize Today's Schedule":**
            *   Identify all tasks scheduled for the remainder of the current day (from the current time onwards).
            *   Unschedule these tasks.
            *   Re-run the Core Scheduling Algorithm for this subset of tasks, considering only time slots available for the rest of today. Tasks that cannot fit may be pushed to subsequent days or flagged.
        *   **"Re-optimize Week's Schedule":**
            *   Identify all tasks scheduled for the remainder of the current week (from the current time on the current day to the end of the user-defined work week).
            *   Unschedule these tasks.
            *   Re-run the Core Scheduling Algorithm for this subset, considering slots within the current week. Tasks that cannot fit may be pushed to the following week or flagged.

    *   **Notification:** After rescheduling, users are notified of significant changes (see System Notifications).

## 3. Conflict Avoidance

Conflict avoidance is a strict rule enforced during all scheduling and rescheduling operations.

*   **Logic:**
    1.  **Consolidated Busy Times:** Before any scheduling attempt, the system creates a consolidated view of all "busy" times. This includes:
        *   Events from all connected and synced external calendars (Google, Outlook, Apple).
        *   User-defined "unavailable" blocks created directly within the FocusFlow app.
        *   Already scheduled task blocks (with their buffer times).
    2.  **Strict Prohibition:** The scheduling engine (both Core and Dynamic) will *never* place a new task block if any part of it (including its pre- and post-buffer times) overlaps with these consolidated busy times.
    3.  **Resolution Attempts:** If a preferred slot for a task is busy, the engine will search for alternative available slots according to its heuristics. If no valid slot is found without violating a hard constraint (like a non-movable event or working hours), the task may be deferred, broken down, or flagged as unschedulable.

## 4. Buffer Time Implementation

Buffer times ensure users have transition periods between tasks/events.

*   **Logic:**
    1.  **User Configuration:** Users can set default `buffer_time_before` and `buffer_time_after` values (e.g., 5, 10, 15 minutes) in their preferences. These can also be set per task, overriding the defaults.
    2.  **Effective Task Duration:** When the Core Scheduling Algorithm or Dynamic Rescheduling Algorithm calculates the time required for a task, it uses: `effective_duration = buffer_time_before + estimated_duration + buffer_time_after`.
    3.  **Slot Identification:** The algorithm searches for free time slots that can accommodate this `effective_duration`.
    4.  **Calendar Event Creation:** When a task is scheduled and an event is created (either in the native calendar or synced to external calendars), the actual calendar event will typically represent the `estimated_duration`, starting after `buffer_time_before` from the beginning of the allocated slot and ending before `buffer_time_after` from the end of the allocated slot. The buffer times themselves are treated as "busy" for scheduling subsequent tasks.
        *   Alternatively, the calendar event itself could span the `effective_duration`, with the task title indicating the core work period (e.g., "Task X (Work: 10:00-11:00, Buffer: 15m before/after)"). This needs UI/UX consideration. The former approach is more common.

## 5. Deadline Proximity Scheduling & Task Breakdown

This feature ensures that large or deadline-critical tasks are handled effectively.

*   **Deadline Proximity Prioritization:**
    *   As described in the Core Scheduling Algorithm, tasks closer to their deadline receive a higher urgency score, naturally prioritizing them for earlier slots.
    *   The engine can also employ a "fill-backwards" heuristic for very critical tasks: attempt to secure the latest possible slot that still meets the deadline, then fill earlier slots if the task is large. This is often combined with task breakdown.

*   **Task Breakdown Logic:**

    *   **When is a task "too large"?**
        1.  **Exceeds Max Chunk Limit:** If `task.estimated_duration` > `user_preferences.max_chunk_duration`.
        2.  **No Single Contiguous Slot:** If the Core Scheduling Algorithm cannot find a single available slot large enough for the task (plus buffers) before its deadline, even if the total available time spread across multiple slots would suffice.
        3.  **User-Initiated (Future):** User might manually flag a task for breakdown.

    *   **How are tasks broken down?**
        1.  **Determine Number of Sub-Blocks:**
            `num_sub_blocks = ceil(task.estimated_duration / user_preferences.max_chunk_duration)` (if applicable, or based on available slot sizes).
        2.  **Sub-Block Duration:** Distribute the `estimated_duration` as evenly as possible among `num_sub_blocks`. For example, a 5-hour task with a 2-hour max chunk limit would become two 2-hour blocks and one 1-hour block.
        3.  **Create Linked Sub-Tasks/Blocks:**
            *   **Option A (Preferred for Clarity):** Create actual sub-tasks in the system (e.g., "Project X - Part 1/3", "Project X - Part 2/3"). These sub-tasks are then scheduled individually.
            *   **Option B (Simpler Data Model):** Schedule multiple, distinct calendar blocks for the same parent task. The UI would need to clearly indicate these are parts of a whole.
        4.  **Scheduling Sub-Blocks:** Each sub-block is then scheduled using the Core Scheduling Algorithm. Dependencies are automatically created between these sub-blocks (Part 1 must complete before Part 2, etc.).

    *   **How do sub-blocks inherit properties?**
        1.  **Deadline:** All sub-blocks inherit the parent task's deadline. The scheduling of the *last* sub-block must meet this deadline.
        2.  **Priority:** All sub-blocks inherit the parent task's priority level.
        3.  **Project/Goal/Type:** Inherited from the parent task.
        4.  **Dependencies:** The first sub-block inherits the parent task's dependencies. Subsequent sub-blocks depend on the completion of the previous sub-block. Any tasks that depended on the original parent task now depend on the *last* sub-block.

## 6. User Scheduling Preferences Processing

User preferences are key to personalizing the schedule.

*   **Working Hours:**
    *   This is a hard constraint. The `working_hours` (including days of the week, start/end times, and defined breaks like lunch) define the absolute boundaries within which tasks can be scheduled. No task block will be scheduled outside these times.

*   **Preferred Work Block Types & Times:**
    *   **Soft Constraint/Heuristic:** When the Core Scheduling Algorithm evaluates potential slots for a task of a specific `type` (e.g., "Deep Work"), slots falling within the user-defined `preferred_block_types_times` (e.g., "Mornings for Deep Work") receive a "desirability" bonus.
    *   **Tie-Breaking:** If multiple slots are equally good based on hard constraints (deadline, priority), the slot matching the type preference will be chosen.
    *   **Flexibility:** If no slot is available within the preferred time, the system will schedule the task in other available (less preferred) slots, possibly notifying the user if a strong preference couldn't be met for a high-priority task.

*   **Task "Chunking" Limits:**
    *   **Primary Handling:** This is primarily addressed by the "Task Breakdown Logic" (Section 5). If a task's `estimated_duration` exceeds `max_chunk_duration`, it's a trigger for breaking it down.
    *   **Break Suggestion/Incorporation:**
        *   If tasks are broken down, natural breaks occur between the scheduled sub-blocks.
        *   If a single task block is scheduled that is shorter than `max_chunk_duration` but still long (e.g., 1.5 hours when limit is 2 hours), the system doesn't automatically schedule a break *within* that block. However, buffer times after the block can serve as short recovery periods.
        *   (Future Enhancement): For very long, uninterruptible but single-instance tasks, the UI could suggest adding a short manual break event immediately after.

## 7. Weighting and Heuristics Overview

The engine uses a hybrid approach combining hard constraints, a scoring system, and heuristics.

1.  **Hard Constraints (Must be met):**
    *   Task Dependencies.
    *   User-defined Working Hours (and unavailable blocks).
    *   External Calendar non-movable events.
    *   Task due dates (tasks cannot be scheduled to finish after their deadline).

2.  **Scoring System (for prioritization among valid tasks):**
    *   **Primary Factors:**
        *   **Explicit Priority Level:** Critical tasks are paramount, followed by High, Medium, Low. This acts as the first level of sorting.
        *   **Deadline Proximity (Urgency):** Within the same explicit priority, tasks due sooner are generally scheduled first.
    *   **Secondary Factors (Heuristics/Bonuses):**
        *   **Preferred Time for Task Type:** Slots matching user's type preferences get a bonus.
        *   **Dependency Unblocking:** Tasks that unblock more subsequent tasks (or high-priority ones) might get a slight bonus.
        *   **Slot Fitness:** (Future) Slots that perfectly fit a task without leaving very small, unusable gaps might be slightly preferred over slots that create such fragmentation, assuming other factors are equal.

3.  **Process Flow Idea:**
    a.  Filter out tasks that violate hard constraints from the outset (e.g., already past due with no possible start time).
    b.  Sort remaining tasks primarily by Priority Level, then by Due Date.
    c.  For each task, identify all potential valid time slots that satisfy hard constraints (working hours, existing events, dependencies, deadline).
    d.  Score each valid slot based on heuristics (e.g., alignment with preferred work times, minimizing fragmentation).
    e.  Select the highest-scoring slot. If multiple slots have similar top scores, factors like "earliest possible" or "fewest gaps" might be used as tie-breakers.
    f.  If a task cannot be scheduled without violating a hard constraint (especially its deadline), it's flagged. For critical/high-priority tasks, this might trigger an immediate alert or re-optimization attempt. For lower-priority tasks, they may remain unscheduled with a reason.
    g.  Task breakdown logic (Section 5) is invoked if a task exceeds chunk limits or cannot fit into a single slot before its deadline. The resulting sub-tasks re-enter the scheduling process.

This system aims to be robust by strictly adhering to hard constraints while using a flexible scoring and heuristic layer to make intelligent choices that align with user preferences and general productivity principles. The exact weights and interplay of heuristics will require iterative tuning based on user feedback and performance testing.
```
