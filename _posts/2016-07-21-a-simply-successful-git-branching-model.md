---
layout: post
title: A Simply Successful Git Branching Model
---

I'm designing this Git workflow based on the well known
[A successful Git branching model](http://nvie.com/posts/a-successful-git-branching-model/) of Vincent Driessen
(known as the **gitflow**), but somewhat simpler, hence the title.
The target user is a smallish team (an *organization* in Github terminology) that

- does not practice rigorous continuous integration or automated testing,
  hence does not guarantee a certain branch is *always* in a *deployable* state;
- develops "service software"---meaning that the mode of work is largely that the deployment "head" moves forward,
  and there is rarely a need to maintain multiple production versions simultaneously.

I make two changes to **gitflow**:

1. there are no **release** branches---release is made from **master**;
2. use `rebase` in some situations, rather than always `merge`.

The subject of most of the descriptions below is one repository.

## Main points

- **master** is for end user.
- **develop** is for team.
- **feature** is for a single developer.
- **feature** makes Github 'pull requests' to **develop**.
- pull requests are code reviewed and `git merge`d into **develop** (or rejected).
- `git rebase` **feature** on top of **develop**
  (to bring other people's changes to **develop** into a **feature** branch).
- `git merge` **develop** into **master** by designated people (from time to time).
- deploy from **master** by designated people.


## 'master' and 'develop' branches

These are the only long-lived branches.

On Github,

- designate **develop** as the *default branch*;
- make both branches *protected*, therefore `--force push` and certain other potentially dangerous things are not allowed;
- make **master** *restricted* to one or two *release managers* in terms of `push` privilege.
  These release managers should be lead developers.
  Some project manager or team head who created the repo (simply because they are admins of the Github account)
  may automatically get the power to `push`, but they should **not** use the privilege unless they are also designated as release managers.
- `push` to **develop** should always be done via *pull requests*, thereby help to enforce code review.
- team members get familiar with each other's code via reviewing pull requests or browsing the **develop** branch.

## Release procedure

1. `git merge` **develop** into **master** (by a release manager)
2. Test the `HEAD` of **master**.
3. When ready, deploy the `HEAD` of **master**.
4. Make a *release* on Github: `tag` the `HEAD` of **master** (using a tag name such as 'release-20160723'), write some release notes.
5. `git merge` **master** into **develop** (in case there were any code changes during the pre-release testing on **master**).
6. Delete all fix branches created during pre-release testing.

Bug fixes during pre-release testing:

1. Create a bug fix branch off of **master**.
2. Basic testing in the fix branch.
3. When ready, `merge` the fix branch into **master** (probably via pull requests, to be merged by a release manager).
4. Continue pre-release testing on the `HEAD` of **master**. Repeat fix-branching / fixing / merging back as needed.
   This bug-fix excursion happens between steps 2 and 3 in the procedure above.

In this procedure, release (and pre-release testing) builds are always made off of the `HEAD` of **master**.
This will give automation scripts an easy time (e.g. the script may simply *always* use `HEAD` of **master**,
and does not worry about commit tags, etc.).

Let me emphasize: **do not touch the *master* branch except when making a release**.

## Hot fix on production

Assume the version in production is always the latest release tag in **master**.
If the process is followed strictly, there should be no commits on **master** since this tag.
(Please try to make this the case. Otherwise be prepared for some headaches.)

In the case that an urgent hox fix needs to be done on the version that is currently in production, do the following:

1. Branch off of **master** to work on the fix.
2. This puts **master** in the pre-release role and the new branch is a fix branch during pre-release testing.
   Follow the **release procedure** above from this point on (i.e. steps 2--6).

## Feature branches

This is where daily development happens.

1. Create a new branch off of **develop**, with a descriptive name, like **feature-a**.
2. Work on the feature. `git push` to the cloud at least daily for safe storage.
3. Integrate new changes (made by other developers) in **develop** into **feature-a**, **often**, like daily.

   If no one is collaborating on branch **feature-a** with you, `rebase` has some advantages: 

   ```bash
   git checkout develop
   git pull
   git checkout feature-a
   git rebase develop -i
   ```

   In some situations, as long as your local work on **feature-a** has not been `push`ed to Github, hence is not yet visible to others, you could use `rebase` regardless of the existence of collaboration . I think this could get tricky; I have no experience with this.

   If you are not the only one working on branch **feature-a**, or if you're nervous about `rebase`, then use `merge`:

   ```bash
   git checkout develop
   git pull
   git checkout feature-a
   git merge develop
   ```

   Resolve conflicts, if any, of course.

   Run tests. Fix bugs until tests pass.

   `git push` branch **feature-a** to the cloud.

5. Integrate **feature-a** into **develop**, **at milestones**.

   Strictly speaking, integrate **feature-a** into **develop** only if
   (1) this feature is going to be part of the next (i.e. upcoming) release, and
   (2) you want to integrate from time to time so that the progress is visible to collaborators
   and gets reviewed.

   Practically, if this feature's code is in a separate directory,
   and does not interfere with other code (that is, the code won't be called, hence won't do any harm
   even if it's half-baked and broken), then whether this feature will be part of
   the *upcoming release* does not matter that much.

   Every time you have finished a functionality and have brought **feature-a**
   to a reasonably tested, functional state, consider making this progress visible to others
   by integrating it into **develop**:

   1. First, integrate **develop** into **feature-a** as described above.

   2. Then, go to Github online and create a **pull request** to merge **feature-a** into **develop**.

   3. Wait for code review and acceptance. React to comments and make changes as needed.

   Depending on the modularity of your work, such **milestones** may happen a couple of times a week.
   *Do not make this milestone cycle too long. Do break up your big project into tasks that
   can be finished in a few days.*

6. Once **feature-a** has been integrated into **develop**,
   and you're done with the feature for good or for a indefinite period of time,
   delete branch **feature-a** both on your local machine and on Github.
   *Feature branches are temporary. Don't keep many feature branches floating around. Kill them.*

Sometimes a feature branch serves as a big 'base' branch for a large feature
that is worked on by multiple developers.
In this situation, the developers create branches off of this base feature branch.
Their relation with this base branch is not unlike that of **feature-a** with **develop** as described above.
However, don't make it too complicated.


## Code re-organization within and between repositories

Do not resist re-organization if that improves structure, modularity, and maintainability---in other words, quality---of the code.

Within a repo, if some code is deprecated but you want to keep them for reference,
**do not** use a dedicated branch to keep (indeed, hide) the code.
Instead, move it to a non-interfering directory (I name the directory 'archive' or 'obsolete')
on the top level in the repo, so that the code remains visible going forward.

I like to think that directory is for horizontal separation,
whereas branch is for vertical separation.
The former is permanent; the latter, temporary.
The dead code example above is horizontal separation.

After some time, it may become clear that all the code does not belong in the same repo.
You want to move part of the code out of the repo.
Simply move the code into 'archive' in the current repo (remember to add some notes),
then copy the code into its new home repo, where it will start with no history.
(Its history stays in the 'archive' folder in the old repo.)

