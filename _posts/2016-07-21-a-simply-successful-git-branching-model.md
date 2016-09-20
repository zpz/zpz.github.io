---
layout: post
title: A Simply Successful Git Branching Model
---

I'm designing this Git workflow based on the well known
[A successful Git branching model](http://nvie.com/posts/a-successful-git-branching-model/) of Vincent Driessen (known as the **gitflow**), but somewhat simpler, hence the title. The target user is a team (an *organization* in Github terminology) that

- does not practice rigorous continuous integration or automated testing, hence does not guarantee a certain branch is always in a *deployable* state;
- develops "service software"---meaning the mode of work is largely the deployment "head" moves forward; there is rarely a need to maintain multiple production versions simultaneously.

I make two changes to the **gitflow**:

1. there are no **release** branches---release is made from **master**;
2. use `rebase` in some situations, rather than always `merge`.

The subject of most of the descriptions below is one repository.

## 'master' and 'develop' branches

These are the only long-lived branches.

On Github,

- designate **develop** as the *default branch*;
- make both branches *protected*, therefore `--force push` and certain other potentially dangerous things are not allowed;
- make **master** *restricted* to one or two *release managers* in terms of `push` privilege. These release managers should be lead developers. Some project manager or team head who created the repo may automatically get the power to `push`, but they should **not** use the privilege unless they are also designated as release managers.
- `push` to **develop** should always be done via *pull requests*, thereby help to enforce code review.

## Release procedure

1. `merge` **develop** into **master** (by a release manager)
2. Test the `HEAD` of **master**.
3. When ready, deploy the `HEAD` of **master**.
4. Make a *release* on Github: `tag` the `HEAD` of **master** (using a tag name such as 'release-20160723'), write some release notes.
5. `merge` **master** into **develop**.
6. Delete all fix branches created during pre-release testing.

Bug fixes during pre-release testing:

1. Create a bug fix branch off of **master**.
2. Basic testing in the fix branch.
3. When ready, `merge` the fix branch into **master** (via pull requests, to be merged by a release manager).
4. Continue pre-release testing on the `HEAD` of **master**. Therefore, this bug-fix excursion happens between steps 2 and 3 in the procedure above.

In this procedure, release (and pre-release testing) builds are always made out of the `HEAD` of **master**. This will give automation scripts an easy time.

Let me emphasize: **do not touch the *master* branch except when making a release**.

## Hot fix on production

Assume the version in production is always the latest release tag in **master**. If the process is followed strictly, there should be no commits on **master** after this tag. (Please try to make this the case. Otherwise there will be some inconveniences.)

Now suppose an urgent hox fix needs to be done on the version that is currently in production. Do the following:

1. Branch off of **master** to work on the fix.
2. This puts **master** in the pre-release role and the new branch is a fix branch during pre-release testing. Follow the **release procedure** above from this point on.

## Feature branches

This is where daily development happens.

1. Create a new branch off of **develop**, with a descriptive name, like **feature-a**.
2. Work on the feature. `push` to cloud at least daily for safe storage.
3. Integrate new changes in **develop** into **feature-a**, **often**, like daily.

   If no one is collaborating on branch **feature-a** with you, `rebase` has some advantages: 

   ```
   git checkout develop
   git pull
   git checkout feature-a
   git rebase develop -i
   ```

   In some situations, as long as your local work on **feature-a** has not been `push`ed to Github, hence is not yet visible to others, you could use `rebase` regardless of the existence of collaboration . I think this could get tricky; I have no experience with this.

   If you are not the only one working on branch **feature-a**, or if you're nervous about `rebase`, then use `merge`:

   ```
   git checkout develop
   git pull
   git checkout feature-a
   git merge develop
   ```
   
   Resolve conflicts, if any, of course.
   
   Run tests. Fix bugs until tests pass.

   Push branch **feature-a** into the cloud.
   
5. Integrate **feature-a** into **develop**, **at milestones**.

   Strictly speaking, integrate **feature-a** into **develop** only if this feature is part of the next (i.e. upcoming) release. And you want to integrate from time to time so that the progress is visible to collaborators.

   Practically, if this feature's code is in a separate directory, and does not interfere with other code, the *upcoming release* does not matter that much.

   Every time you have finished a functionality and have brought **feature-a** back to a (reasonably) tested, functional state, consider making this progress visible to others by integrating it into **develop**. Depending on the modularity of your work, this may happen a couple times a week.

   First, integrate **develop** into **feature-a** as described above.

   Then, go to Github online and create a **pull request** to merge **feature-a** into **develop**.

6. Once **feature-a** has been integrated into **develop**, and you're done with the branch for good or for a indefinite period of time, delete branch **feature-a** both on your local machine and on Github.

Sometimes a feature branch serves as a big 'base' branch for a large feature that is worked on by multiple developers. In this situation, the developers create branches off of this base feature branch. Their relation with this base branch is not unlike that of **feature-a** with **develop** as described above.


## Code re-organization within and between repositories

Do not resist re-organization if that improves structure, modularity, and maintainability---in other words, quality---of the code.

Within a repo, if some code is deprecated but you want to keep them for reference, **do not** use a dedicated branch to keep (indeed, hide) the code. Instead, move it to a non-interfering directory (such as `archive`) on the top level in the repo for such "horizontal separation".

After some time, it may become clear that all the code does not belong in the same repo, and you want to move part of the code out of the repo. No problem. Simply move the code into `archive` in the current repo (remember to add some notes), then copy it into its new home repo, where it will start with no history.
