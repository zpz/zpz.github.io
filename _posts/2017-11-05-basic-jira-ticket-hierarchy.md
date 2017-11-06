---
layout: post
title: "Basic Issue Type Hierarchies in JIRA"
---

I describe a reasonable, basic hierarchy of JIRA issue types
for software development,
based on a little bit of reading, observation, and thinking.

The very basic object (or document) in JIRA is an **issue**.
I'm not a JIRA admin, and have never configured JIRA.
It is my understanding that you can define any number of issue types
of whatever name and pick one of the types for the issue you are creating;
there is no set meaning to or relationship between the issue types.
It's all up to your design.

So far I've found it useful to have only three issue types:
**epic**, **task**, and **bug**.

An important reference is a **sprint**.
This is basically the **team planning** cycle.
I'll think of two weeks as the default length.


Project
=======

**Project** is not an issue type.
Admin defines a number of "projects".
An issue must belong to exactly one project;
this is selected when creating a new issue,
and should be changeable for an existing issue.

Project is on a very high level.
It's some undertaking that definitely lasts for months, if not years.
It is not unreasonable to just use a team's identity as a "project" designation.

**Project** is an aspect of *organization*, not *planning*.
The number of projects should be small.
You don't want to choose from a long list of "projects",
and only minutes later choose from another long list of "epics",
while creating an issue.

Epic
====

**Epic** is an issue type.

An epic is an initiative that **can not be finished within a single sprint**.
Think of a few months.

An epic does not need a long and detailed description.
It should be mainly in product language rather than technical language.

The actual work of an epic will be decomposed into
and carried out by a number of **tasks**.

Do not do much actual work against an Epic directly.
Use an **Epic** as a catalog of **Tasks**.


Task
====

**Task** is an issue type.

A task is technically specific, ready to be implemented, and
**finished within a single sprint, like days**.

A Task should describe exactly what needs to be done
in reasonable detail, in technical language.

A Task may be linked to an Epic, if applicable.
But it does not have to---there are standing alone tasks.

Do not evolve much of a Task's content during the execution.
Finish it, close it, and move to the next Task.

Subtask
=======

**Subtask** is not a issue type chosen when creating an issue.
Rather, it is created under an existing Task.

A Subtask is a component of a Task that
**can be finished within a day**.

Do break down a multi-day Task into Subtasks.
This is a useful way for prioritizing various components of a large Task
and providing a very positive feeling of progress.


Bug
===

**Bug** is an issue type.
It is the same as a Task except that the nature is fixing a bug.
Like a Task, a Bug can be broken down into Subtasks.


Story (optional)
================

Agile posts talk about "stories".
This issue type may be needed if a product manager has close involvement
in the planning of the development work.

A Story should be **contained within a single sprint**.
It is described in product language.
Its actual work is delegated to one or more Tasks,
which contain technical specifics.


Conclusion
==========

In summary,

- **Epic**: weeks or months, multiple sprints; product language; evolving scope; decomposed into Tasks.
- **Task**: days, one sprint; technical specifics; fixed scope; decomposed into Subtasks.
- **Subtask**: hours, the smallest piece of logically self-contained technical work.

