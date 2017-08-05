---
layout: post
categories: ansible
title: "Ansible-Semaphore UI Walkthrough"
description: "Bunch of screenshots from the Ansible-Semaphore UI"
keywords: "ansible, guide, ansible-semaphore"
---
# Table Of Contents
* TOC
{:toc}

# Introduction

Ansible-Semaphore is Open Source alternative for Ansible Tower (Web UI and API for launching Ansible Tasks). Source code [hosted](https://github.com/ansible-semaphore/semaphore) on Github under MIT license. Semaphore supports LDAP authentification, bearer tokens for REST API, Telegram and Email alerts for the failed tasks and simple user roles system (under development, see [#413](https://github.com/ansible-semaphore/semaphore/pull/413) and [#405](https://github.com/ansible-semaphore/semaphore/pull/405) PRs).
It's written on Go (backend) and Angular (frontend).

# UI Walkthrough

Current screenshots taken from Semaphore 2.4.1 + [#413](https://github.com/ansible-semaphore/semaphore/pull/413) and [#405](https://github.com/ansible-semaphore/semaphore/pull/405) PRs.

## Login Page
![Semaphore Login Page](/static/img/posts/semaphore-login.png)

## Main Page
![Semaphore Main Page](/static/img/posts/semaphore-main-page.png)

1. List of all appeared events: Task Template/User/Project/Repository/etc Creation/Updating/Deletion, Task status updates, and much more;
1. Project List and Project Add button.

## Global User List
![Semaphore Global User List](/static/img/posts/semaphore-global-user-list.png)
This is the list of all users in Semaphore.

Explanation of some properties:

* *Alert*: user will receive alerts about failed tasks to his email;
* *Admin*: user can create new users and edit their information;
* *External*: user authenticates through external backend (currently only LDAP is supported). Admin can't edit his username and password.

## Global User Edit Dialog
Normal and External user edit dialog:
![Semaphore Global User Edit Dialog](/static/img/posts/semaphore-global-user-edit.png)

## Main Project Page
![Semaphore Main Project Page](/static/img/posts/semaphore-main-project-page.png)

1. Project Settings button and Project Menu:
    * Dashboard: this page
    * Task Templates: all task templates, available for launch
    * Inventory: Semaphore currently supports Static Inventory (same as ordinary Ansible inventory file). Dynamic Inventories not yet supported (see [#47](https://github.com/ansible-semaphore/semaphore/issues/47)). Also, there is possible to use inventory from your playbooks repo (and pass it via Ansible `-i` flag);
    * Environment: variables, related to this project;
    * Key Store: SSH keys for playbook repository fetching and ansible SSH connections. Various Task Templates may use various keys;
    * Playbook Repositories: list of git repos with playbooks for this project. Currently supported only SSH fetching. Local repos and HTTPS repos isn't supported;
    * Team: list of users belong to this project. Currently, users have 2 properties: Admin user can add new users to the project, Launch-Only user cannot edit or create new items (Task Templates, Inventories, Environments etc) in the project. Admin User can be also a Launch-Only user (e.g. in the case to prevent accidental editing/deletion of items) and can edit this property for himself.
1. Project activity: list of all events related to this project (similar to Events on the main Semaphore page)
1. Task history: list of all launched tasks. It contains:
    * Task name;
    * Playbook name (green - task successful, red - task failed, blue - task running, gray - task waiting in the queue);
    * Task start time (in local user time);
    * Task duration (rounded to minutes);
    * Task initiator.

## Project Settings Page
Settings are pretty simple: project name, Telegram chat id for alerting and on/off switch for alerts.

![Semaphore Project Settings Page](/static/img/posts/semaphore-project-settings-page.png)

## Task Templates Page

Launch-Only User (bottom half of screen) cannot copy or create new task templates.

![Semaphore Task Templates Page](/static/img/posts/semaphore-task-templates-page.png)

## Task Template Edit Page

![Semaphore Task Template Edit Page](/static/img/posts/semaphore-task-template-edit-page.png)

You should firstly create Key Store, Playbook Repository, and Inventory items and then create a Task Template item. If you use inventory from your playbook repo, then you should create an empty inventory without any content (will be fixed in [#328](https://github.com/ansible-semaphore/semaphore/issues/328)).

## Inventory Page

Red item is marked as removed (it should be deleted, but used by some task templates).

![Semaphore Inventory Page](/static/img/posts/semaphore-inventory-page.png)

## Key List and Key Edit Pages

AWS, Digital Ocean, and Google Cloud can be created, by cannot use because Dynamic Inventory support not yet realized.

![Semaphore Key List Page](/static/img/posts/semaphore-key-list-page.png)

![Semaphore Key Edit Page](/static/img/posts/semaphore-key-edit-page.png)

## Repository List Page

![Semaphore Repository List Page](/static/img/posts/semaphore-repo-list-page.png)

## Project Team Page and Add Project User Dialog

![Semaphore Project Team Page](/static/img/posts/semaphore-project-team-page.png)

![Semaphore Add Project User Page](/static/img/posts/semaphore-add-project-user-page.png)

## Task Launch Process

After clicking on Run button, Semaphore shows Run/Dry Run Dialog:

![Semaphore Run/Dry Run Dialog](/static/img/posts/semaphore-dry-run-page.png)

Then, Semaphore put the task in the queue. Currently, Semaphore can execute only one task from all project at the same time, will be fixed in [#366](https://github.com/ansible-semaphore/semaphore/pull/366).

If the task on top of the queue, it starts. Semaphore and Ansible logs will be shown to the user and stored in the database.

![Semaphore Running Task Page](/static/img/posts/semaphore-running-task-page.png)

The user can view any finished task log.

![Semaphore Finished Task Page](/static/img/posts/semaphore-finished-task-page.png)