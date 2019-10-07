There are three places git uses to manage your project:

**Working directory:** your whole project. (untracked, changes not staged, changes to committed). `git status` lists the state of your working directory.

**Index:** also known as 'staging area'. All the changes you want to make to your next commit. When using ```git add <file>``` or ```git add .```, you add to the index. Consider it the staged snapshot of the next commit.

**Tree:** when you commit, you add to the tree. It actually records the whole history of commits.

**HEAD:** a pointer to the current branch, which in turn a pointer to your last commit you made or checked out into your working directory. Consider it the **snapshot** of the last commit.

**Branch:** nothing but pointers to commits. You are 'on a branch' when HEAD is pointing to this branch.

Switching branches first changes the HEAD to point to the new commit, populates the index with the snapshot of that commit, then checks out the content in the index into the working directory.


[Undo changes](https://www.atlassian.com/git/tutorials/undoing-changes)
---
**Reset**   
Undo changes that haven't shared in public. It uses different options to manipulate tree, index and working area in a specific order

Reset to `commit` we want, how git handles the changes included in the discarded commit (after the commit we are resetting to) depends on options:

1. Moving HEAD `git reset --soft <commit>`
   Only reset HEAD to the commit. like `git checkout <commit>`, but does not create detached head state. Changes stay in staging area.   
   When you run `git status`, it will show changes to be commited

2. Updating the index `git reset [--mixed]  <commit>`
   This is the default option. It will un-stage everything, making you roll back to what you have before ```git add``` and ```git commit```

3. Update the working directory --hard
   Continue to undo changes in the working directory. All changes will get LOST!

```sh
# assume we just did a merge commit from remote, now want to rollback to previous one

# HEAD@{0} is 'pull and merge'
# HEAD@{1} is the commit we had before merging, that's where we want to reset to
# HEAD@ is pointer to a particular commit
git reset --hard HEAD@{1}
# same as
git reset --hard HEAD^
```


**Revert**  
It is the tool to undo changes you have pushed to remote. Basically you specify those commits that need to be undone, git helps undo all changes introduced in those commits and make new commits for each with content rollback. So it helps you avoid undoing all changes manually.
```sh
# create 3 separate revert commits
$ git revert a867b4af 25eee4ca 0766c053
# use range
$ git revert a867b4af..0766c053

# easy to revert last 3 commits
$ git revert HEAD~3

# do not make commit for each, after this we can commit manually
$ git revert --no-commit hash1 hash2
```

[Integrate change](https://www.atlassian.com/git/tutorials/merging-vs-rebasing)
---

When we incorporate remote changes that other developers have pushed, we can only fetch those commits first and make our local commits on top of them, not the other way around.
We cannot rebase the commit history of publicly shared branch (pulled by other people into their local branch and have worked on it)

◘ Rebase to rewrite the **LOCAL** commit history, that hasn't been pushed and pulled by anyone
```sh
# Show a list of commits since dd61ab32
$ git rebase -i dd61ab32^
# then we can choose to keep, remove, squash some of the commits, and save

$ git push <remote> <branch> -f # force to remote if needed, be careful
```

◘ Rebase between master (release) and topic (dev)
```
c means commit hash
c1 <- c2 <- c3' <- c4'       [master]
       \    
        [origin/master]

     c3 <- c4 <- c5 <- c6 <-c7 [dev] HEAD
```
⁃ First [master] is `c3 <- c4`, branch out [dev], start adding things c5,c6...    
⁃ Then some changes pushed to [origin/master], we `pull -r` remote changes (c1 and c2)  so c3,c4 changed to c3',c4' (as history changed)   
⁃ c3,c4 in [dev] remain unchanged     

We want to incorporate changes from dev to master (rebase dev first and merge in master )

```
                     c5' <- c6' <-c7'  dev HEAD   
                    /
c1 <- c2 <- c3' <- c4'  master    
       \                    
        origin/master
```

```sh
# if we are not on topic branch:
git rebase master dev # shorthand for git checkout dev && git rebase master

# if we are on dev branch
git rebase master [dev]  
# since upstream branch (master, the one we rebase current against) contains changes
# (same snapshot,different commit hash) in dev (c3, c4) they will be skipped

# Another way
# take commits from c5 up to c7 (HEAD pointing to) and apply on top of master
# c5,c7 can be any commit hash from different branch
git rebase onto master c5^ [HEAD]
```
**[Undo rebase](http://gunnariauvinen.com/how-to-undo-a-git-rebase-and-recover-hours-of-work/)**  
Real case: we are in `new-feature` branch, and just pulling one commit from remote master into local master, when we do `git rebase master`, we end up with reflogs below:

```sh
# git reflog:
5b8fb9d HEAD@{0}: rebase finished: returning to refs/heads/auth-fbaccountkit
5b8fb9d HEAD@{1}: rebase: Also verify Facebook app id associated with account kit login
94478c9 HEAD@{2}: rebase: Integrate auth adapter for Facebook accountkit login
# original branch commits are put on top of '33de770' instead of 'bad2179'
33de770 HEAD@{3}: rebase: checkout master
b44c378 HEAD@{4}: checkout: moving from master to auth-fbaccountkit

# 33de770: new commit from master (Add Parse Server Generic Email Adapter to README (#4101)
33de770 HEAD@{5}: merge upstream/master: Fast-forward
bad2179 HEAD@{6}: checkout: moving from auth-fbaccountkit to master

# b44c378: current tip of feature branch, have 2 commits (the last amended)
b44c378 HEAD@{7}: commit (amend): Also verify Facebook app id associated with account kit login
84aeb15 HEAD@{8}: commit: Also verify Facebook app id associated with account kit login
b488568 HEAD@{9}: commit: Integrate auth adapter for Facebook accountkit login
bad2179 HEAD@{10}: checkout: moving from master to auth-fbaccountkit
bad2179 HEAD@{11}: clone: from https://github.com/6thfdwp/parse-server
```
So we want to go back to original tip of feature branch, which is `HEAD@{7}`

we can see original commits history before rebase with `git log HEAD@{7}:`
```
commit b44c37814e984ac6b16fe2874ab35912f5c7fa90
    Also verify Facebook app id associated with account kit login

commit b4885681a7051d277a5d03c588bea9c54f818305
    Integrate auth adapter for Facebook accountkit login

commit bad217911cb870b7b4fb849c6c7e97d6ed5e6528
    Adds ability to login with email when provided as username (#4420)

```

Then we do `git reset --hard HEAD@{7}`, moving tree head pointing to `b44c378`, update the index, un-stage all commits after it (those we rebased from), as like no commit added, finally update the working directory, give exactly what we have before rebasing.

#### Inspect the history
> commit history

git log shows existing commits history, git reflog shows full ops history

> file changes history (between lines)
```sh
git log -L 50,75: ./some/path/file.ext
git blame -L 50, 57
```

---

```sh    
# 1. new features
git checkout -b exciting-feature
git commit
# push to the same remote branch (origin/exciting-feature) and set upstream to it
git push -u origin HEAD

git check master
git pull [origin master]
git merge exciting-feature

# from current branch to remote master
git push <remote> HEAD:master
```
