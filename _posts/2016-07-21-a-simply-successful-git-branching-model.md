---
layout: post
title: A Simply Successful Git Branching Model
---

I'm designing this Git workflow based on the well known
[A successful Git branching model](http://nvie.com/posts/a-successful-git-branching-model/) of Vincent Driessen (known as the **gitflow**), but somewhat simpler, hence the title. The target user is a team (an *organization* in Github terminology) that

- does not practice rigorous continuous integration or automated testing, hence does not guarantee a certain branch is always in a *deployable* state;
- has been somewhat liberal (read: sloppy) in the use of Git.

I make two changes to the **gitflow**:

1. there are no **release** branches---release is made from **master**;
2. use `rebase` in some situations, rather than always `merge`.

## 'master' and 'develop' branches

These are the only two long-lived branches.

On Github,

- designate **develop** as the *default branch*;
- make both branches *protected*, therefore `--force push` and certain other potentially dangerous things are not allowed;
- make **master** *restricted* to a small number of designated users in terms of `push` privilege. Ideally, a repo should have exactly **one release manager** who has `push` privilege to **master**. Some manager or team head who created the repo may automatically get the power to `push`, but they should **not** use it unless they are also the designated release manager.

## Release procedure

1. `merge` **develop** into **master**.
2. Test the `HEAD` of **master**.
3. When ready, deploy the `HEAD` of **master**.
4. Make a *release* on Github: `tag` the `HEAD` of **master** (using a tag name such as 'release-20160723'), write some release notes.
5. `merge` **master** into **develop**.

Bug fixes during pre-release testing:

1. Create bug fix branch off of **master**.
2. Test the fix in the fix branch.
3. `merge` the fix branch into **master**.
4. Continue pre-release testing on **master**.

In this procedure, release (and pre-release testing) builds are always made out of the `HEAD` of **master**. This will give automation scripts happy.

## Hot fix on production

Assume the version in production is always the latest release tag in **master**. If the process is followed, there should be no commits on **master** after this tag. (With a liberal team, however, there could be a few, which would make things a little more messy. But by all means please avoid that. Stick to the *only-release-manager-pushs-to-master* policy.)

1. Branch off of **master** to work on the fix.
2. This puts **master** in the pre-release role and the new branch is a fix branch during pre-release testing. Follow the **release procedure** above from this point on.

## Feature branches

This is where daily development happens.

1. Create a new branch off of **develop**, with a descriptive name, like **feature-a**.
2. Work on the feature.
3. Integrate new changes in **develop** into **feature-a**, **often**, like daily.

   If no one is collaborating on **feature-a** with you, then 

   ```
   git checkout develop
   git pull
   git checkout feature-a
   git rebase develop -i
   ```

   Regardless of collaboration, as long as your local work on **feature-a** has not been `push`ed to Github, hence is not yet visible to others, you can use `rebase`.

   Otherwise, or in doubt,

   ```
   git checkout develop
   git pull
   git checkout feature-a
   git merge develop
   ```

4. Integrate **feature-a** into **develop**, **at milestones**.

   Strictly speaking, integrate **feature-a** into **develop** only if this feature is part of the upcoming release. And you want to integrate from time to time so that the progress is visible to collaborators.

   Practically, if this feature work is in a separate directory, and does not interfere with other code, the *upcoming release* does not matter much.

   Every time you have finished a self-contained functionality and have brought **feature-a** back to a (reasonably) tested, functional state, consider making this progress visible to others by integrating it into **develop**. Depending on the modularity of your work, this may happen a couple times a week.

   First, integrate **develop** into **feature-a** as described above.

   Then, integrate **feature-a** into **develop**:

   ```
   git checkout develop
   git pull    # this should show "Already up-to-date."
   git merge feature-a [--no-ff]
   git push
   ```

5. Once **feature-a** has been integrated into **develop**, and you're done with the feature for good for a long period of time, delete branch **feature-a** both on your local machine and on Github.

Sometimes a feature branch serves as a big 'base' branch for a large feature that is worked on by multiple developers. In this situation, the developers create branches off of this base feature branch. Their relation with this base branch is not unlike that of **feature-a** with **develop** as described above.




