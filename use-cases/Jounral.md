
# Task Manager CLI - Code Analysis and Architecture Review

Task Manager CLI - Code Analysis and Architecture Review is a command line tool that provides a code-related analysis and architecture review.

## Introduction

At first I thought the project was just going to be a simple Java work management program. After I went through the code, I noticed that it's a structured layered junit test.

All the following files were analysed:

- `TaskManagerCli.java`
- `TaskManager.java`
- `Task.java`
- `TaskStatus.java`
- `TaskPriority.java`
- `TaskStorage.java`

I wanted to know how it was implemented and what design decisions were taken in the process of the project.

---

# Understanding the Structure of the Application

One of the first things that I noticed was that there is a division of the application into packages and each package has a specific responsibility.

## CLI Layer

The CLI layer is the beginning point of the application. It listens to the user commands, parses the arguments and sends a request to the service layer.

## Service Layer

Application's business rules are in the service layer. All operations are mediated by `TaskManager`, which is not allowed to manipulate tasks directly by the CLI.

## Model Layer

Model layer is the layer that models all the objects in the system. This comprises tasks, their statuses and priorities.

## Storage Layer

The storage layer stores the data of the tasks and saves it to disks, using the JSON serialization.

This separation makes it easier for the code to be understood, because each component has only one responsibility.

---

# Feature Analysis – Making a Task

There are several components to the creation of a task.

Once the user inputs a create command, the create information is retrieved by the CLI and passed to the service layer.

Then a series of validations and conversions are done in the service layer:

- Maps the numeric priority to an item of TaskPriority.
- Parses and returns a `LocalDateTime` from a date string.
- Verifies information provided in advance of task development.

After validation it creates a new task object.

It's interesting that each new task is created and immediately set to the ‘TODO' state. The constructor ensures this as tasks are always added to the system in the same state.

Once created, the task is saved in the in-memory collection of the application prior to being stored on disk.

## Task Creation Workflow

```text
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

# Defining how the task status changes.Defining what should happen to the task status.

Changing the status for a task showed some interesting choices that were made in implementation.

When a user enters:

```bash
status <task_id> <status>
```

The application takes the provided text and updates the corresponding task, based on the supplied text being converted into a TaskStatus enum.

During the analysis what was noticeable was the lack of consistency in status changes.

The application generates a temporary task object that has only the changed values for most status updates. This is a temporary object that is then used to update the original task.

But when the task is `DONE`, a completely different process takes place.

The application uses: to find the real task and to directly change it, rather than creating a temporary object:

```java
markAsDone()
```

This method updates:

- Status
- Completion date
- Last updated timestamp

The changes are then immediately saved.

They both give the same results but are two different strategies of updating task data.

---

# The Priority System is discussed.The Priority System is examined.

Going into this I thought that the priority feature would have an impact on what order the tasks would appear in when viewing the implementation.

The priority levels are expressed with numbers:

| Level | Value |
|---------|---------|
| LOW | 1 |
| MEDIUM | 2 |
| HIGH | 3 |
| URGENT | 4 |

Due to these numeric values, it was likely that high priority tasks would show up before low priority tasks.

I found out this was false after following the code.

Priority values are not used to sort tasks in the application.

Rather, priority is used for three primary purposes:

## Filtering

Users can get tasks associated with a particular priority category.

## Statistical Reporting

Priority values are counted when generating task statistics.

## Display Formatting

The CLI interacts with priority levels in a graphical manner, for example, exclamation marks.

The numeric values are not there to rank tasks - the primary reason for using them is to make the user input and identification of the categories easier.

---

# Impacts of tracking data on the system are monitored.

I traced the whole application workflow of marking a task as done, to see how information flows throughout the application.

## User Action

```bash
taskmanager status 12345 done
```

## Internal Processing

The command is sent down several layers:

```text
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

The task is retrieved from storage, altered in memory, and then re-written to storage.

In the process, three fields are updated:

- `status`
- `completedAt`
- `updatedAt`

All task data is stored once the update has been completed.

---

# Storage Strategy

The application has a simple persistence strategy.

All of the tasks are kept in one:

```java
HashMap<String, Task>
```

During run-time of the application.

Task IDs are used as map keys, so it is fast to look them up.

It uses only one JSON file for permanent storage:

```text
tasks.json
```

Each time a change is made, all the set of tasks is re-written to the file.

Examples include:

- Creating tasks
- Updating tasks
- Deleting tasks
- Modifying priorities
- Updating due dates
- To add and delete tags.

This is a straightforward method, but can get inefficient with a large number of tasks.

---

# Identify and discuss design decisions.

A number of options emerged during analysis for implementation.

## Puppy Training: Patch Object Update Pattern

Task objects are often created as temporary objects with only the modified fields.

These are temporary objects that are then incorporated into previously completed tasks.

The same method is used as a patch operation with only the values that have been changed being copied.

## Direct Object Mutation

The application doesn't go through the patch model when completing tasks, it just updates the original object.

This means that the updates are not consistent across the system.

## Full File Persistence

Each change results in an overall rewrite of the JSON file.

This makes for a much easier implementation, but also raises disk operations and the possibility of having disk operations interrupted by a write operation.

---

# Risks and limitations identified:Risks and limitations identified:

While analyzing I discovered that there could be problems in a couple of areas.

## Limited Error Handling

Some values for the status variable may cause exceptions to be thrown that do not always occur gracefully.

You might get a stack trace instead of a message to the user if the user made a mistake.

## File Corruption Risk

The application directly modifies `tasks.json`.

The program may crash while writing and cause the file to be incomplete or corrupt.

## Concurrent Access Issues

The application does not implement:

- File locking
- Synchronization
- Concurrency controls

Several instances may change each other.

## Shared References

The storage layer provides direct object references for the tasks.

These references are to original objects, and changes can be made without further protections.

---

# Reflection

The most important thing we learned from this exercise was the need to always test assumptions with code tracing.

Initially the priority system seemed to implement the task ranking, due to their numerical values. But it was found that it is only a categorisation system.

Another benefit of the tracing was to get a better understanding of the division of responsibility within a layered architecture and to trace commands through multiple layers. The data from the command line was passed through the service layer, the model layer and the storage layer, giving me a much better understanding of the operation of the application as a whole system, rather than just as individual classes.

In general, this analysis identified the great features in the application architecture as well as some of the decisions made in the implementation that could be enhanced in future versions.
