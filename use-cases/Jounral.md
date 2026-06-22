# Task Manager CLI - Code Analysis and Architecture Review

Task Manager CLI - Code Analysis and Architecture Review is a command line tool, and this document provides a code-related analysis and architecture review of it.

## Introduction

At first I thought the project was just going to be a simple Java task management program. After I went through the code, I noticed that it's actually a structured, layered architecture. All of the following files were analyzed:

- TaskManagerCli.java
- TaskManager.java
- Task.java
- TaskStatus.java
- TaskPriority.java
- TaskStorage.java

I wanted to know how it was implemented and what design decisions were taken in the process of building the project.

---

# Understanding the Structure of the Application

One of the first things I noticed was that the application is divided into packages, and each package has a specific responsibility.

## CLI Layer
The CLI layer is the entry point of the application. It listens for user commands, parses the arguments, and sends a request to the service layer.

## Service Layer
The application's business rules live in the service layer. All operations are mediated by `TaskManager` — the CLI is not allowed to manipulate tasks directly.

## Model Layer
The model layer models all the objects in the system. This comprises tasks, their statuses, and their priorities.

## Storage Layer
The storage layer stores task data and saves it to disk using JSON serialization.

This separation makes the code easier to understand, because each component has only one responsibility.

---

# Feature Analysis – Making a Task

There are several components involved in the creation of a task. Once the user runs a `create` command, the input is retrieved by the CLI and passed to the service layer. A series of validations and conversions are then done in the service layer:

- Maps the numeric priority to an item of `TaskPriority`.
- Parses a date string into a `LocalDateTime`.
- Checks the information provided before the task is built.

After validation, a new task object is created. It's interesting that every new task is immediately set to the `TODO` state — the constructor ensures this, so tasks are always added to the system in the same starting state. Once created, the task is added to the application's in-memory collection before being written to disk.

## Task Creation Workflow
```
User Command
      ↓
TaskManagerCli
      ↓
TaskManager
      ↓
Task Constructor
      ↓
TaskStorage
      ↓
tasks.json
```

---

# Defining How Task Status Changes

Changing the status of a task revealed some interesting implementation choices.

When a user enters:
```bash
status <task_id> <status>
```

the application takes the provided text, converts it into a `TaskStatus` enum, and updates the corresponding task.

During the analysis, what was noticeable was a lack of consistency in how status changes are handled. For most status updates, the application creates a temporary task object holding only the changed value. This temporary object is then used to update the original task.

But when the new status is `DONE`, a completely different process takes place. Instead of creating a temporary object, the application looks up the real task directly and calls:

```java
markAsDone()
```

This method updates:
- Status
- Completion date
- Last updated timestamp

The changes are then immediately saved. Both paths produce the same end result, but they are two different strategies for updating task data.

---

# The Priority System

Going into this, I thought the priority feature would affect the order in which tasks appear when viewing them. The priority levels are expressed with numbers:

| Level | Value |
|---------|---------|
| LOW | 1 |
| MEDIUM | 2 |
| HIGH | 3 |
| URGENT | 4 |

Because of these numeric values, it seemed likely that high-priority tasks would show up before low-priority ones. After following the code, I found this assumption to be false. Priority values are not used to sort tasks anywhere in the application. Instead, priority is used for three main purposes:

## Filtering
Users can retrieve tasks associated with a particular priority level.

## Statistical Reporting
Priority values are counted when generating task statistics.

## Display Formatting
The CLI represents priority levels visually — for example, using exclamation marks.

The numeric values aren't there to rank tasks; their main purpose is to make user input and category identification easier.

---

# Tracking Data Impacts on the System

I traced the application's full workflow for marking a task as done, to see how information flows through the system.

## User Action
```bash
taskmanager status 12345 done
```

## Internal Processing
The command passes down through several layers:
```
CLI
 ↓
TaskManager
 ↓
TaskStorage
 ↓
Task Object
 ↓
JSON Persistence
```

The task is retrieved from storage, altered in memory, and then re-written to storage. In the process, three fields are updated:
- status
- completedAt
- updatedAt

All task data is saved once the update is complete.

---

# Storage Strategy

The application uses a simple persistence strategy. All tasks are kept in a single:

```java
HashMap<String, Task>
```

during runtime. Task IDs are used as map keys, so lookups are fast. For permanent storage, it uses a single JSON file:

```
tasks.json
```

Each time a change is made, the entire set of tasks is re-written to the file. Examples include:
- Creating tasks
- Updating tasks
- Deleting tasks
- Modifying priorities
- Updating due dates
- Adding and removing tags

This is a straightforward approach, but it can become inefficient with a large number of tasks.

---

# Identifying and Discussing Design Decisions

A number of notable implementation choices emerged during the analysis.

## Patch Object Update Pattern
Temporary task objects are often created holding only the modified field(s). These temporary objects are then merged into the existing, already-saved task. The same `update()` method is used as a patch operation, copying over only the values that have changed.

## Direct Object Mutation
When completing a task, the application skips the patch-object model entirely and updates the original object directly. This means updates are not handled consistently across the system.

## Full File Persistence
Each change results in the entire JSON file being rewritten. This makes the implementation simpler, but it also increases disk operations and introduces the possibility of a write being interrupted partway through.

---

# Risks and Limitations Identified

While analyzing the code, I identified a few areas of concern.

## Limited Error Handling
Some invalid status values can cause exceptions that aren't handled gracefully. A user mistake might surface as a raw stack trace instead of a clean error message.

## File Corruption Risk
The application overwrites `tasks.json` directly. If the program crashes mid-write, the file could end up incomplete or corrupted.

## Concurrent Access Issues
The application implements none of the following:
- File locking
- Synchronization
- Concurrency controls

Multiple instances running at once could overwrite each other's changes.

## Shared References
The storage layer hands out direct references to the actual task objects, not copies. Changes can be made to these objects without any additional protection.

---

# Reflection

The most important thing I learned from this exercise was the need to always test my assumptions against the actual code. Initially, the priority system seemed like it would implement task ranking, because of its numeric values — but tracing the code showed it's only a categorization system.

Tracing the code also gave me a better understanding of how responsibility is divided across a layered architecture, and how a single command travels through multiple layers. A command entered on the CLI passes through the service layer, the model layer, and the storage layer — which gave me a much clearer picture of how the application works as a whole system, rather than as a set of isolated classes.

Overall, this analysis surfaced both the strong points of the application's architecture and a few implementation decisions that could be improved in future versions.
