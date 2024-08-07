# py-package-template
Template for creating a python package repository

## Setting up the git workflow

```
git clone ssh://git@git.example.com/project/repo
cd repo
git checkout $(git commit-tree $(git hash-object -t tree /dev/null) < /dev/null)
git worktree add main main
```

Justification of working with git worktrees: [ref1](https://www.youtube.com/watch?v=2uEqYw-N8uE) [ref2](https://morgan.cugerone.com/blog/how-to-use-git-worktree-and-in-a-clean-way/)

Best way I found to work with worktrees is described in [this ref](https://stackoverflow.com/questions/54367011/git-bare-repositories-worktrees-and-tracking-branches) 
which corresponds:
* not using `--bare` due to the problems with `git fetch` and `git pull`
* removing everything in the repository in a detached HEAD (with `git checkout $(git commit-tree $(git hash-object -t tree /dev/null) < /dev/null)`) so that in the `repo` directory there are only the worktree directories (so it is cleaner)
