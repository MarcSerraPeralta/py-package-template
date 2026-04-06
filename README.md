# py-package-template

Template for creating a python package repository. 

##### Table of Contents  
- [Setting up the git worktrees](#git-worktrees)  
  - [Cloning the repo](#cloning-repo)  
  - [Creating new worktrees](#new-worktrees)
- [Setting up PR configuration](#pr-configuration)
- [Managing the requirements for the python package](#managing-requirements)
- [Publishing the package to PyPI](#publish-pypi)
- [GitHub actions for CI/CD pipeline](#ci-pipeline)
- [Badges in `README.md`](#readme-badges)
- [Documentation hosted using ReadTheDocs](#readthedocs)
- [Jupyter lab and notebook](#jupyter)
- [SSH tips](#ssh)
- [Git tips](#git-tips)


## Setting up the git worktrees <a name="git-worktrees"/>

### Cloning the repo <a name="cloning-repo"/>

```
git clone ssh://git@git.example.com/project/repo
cd repo
# create detached HEAD with no files on the directory (cleaner structure)
git checkout $(git commit-tree $(git hash-object -t tree /dev/null) < /dev/null)
# create new worktree corresponding to the `main` branch
git worktree add main main
```

Justification of working with git worktrees: [ref1](https://www.youtube.com/watch?v=2uEqYw-N8uE) [ref2](https://morgan.cugerone.com/blog/how-to-use-git-worktree-and-in-a-clean-way/)

Another advantage of using the described method is that it is possible to install different branches of the project in edible mode in different virtual environments (e.g. `pip install -e main/`). 
If git worktrees are not used, then one needs to remember to checkout to the correct branch when using each virtual environment, which is very error prone. 

Best way I found to work with worktrees is described in [this ref](https://stackoverflow.com/questions/54367011/git-bare-repositories-worktrees-and-tracking-branches) 
which corresponds:
* not using `--bare` due to the problems with `git fetch` and `git pull`
* removing everything in the repository in a detached HEAD (with `git checkout $(git commit-tree $(git hash-object -t tree /dev/null) < /dev/null)`) so that in the `repo` directory there are only the worktree directories (so it is cleaner)


### Creating new worktrees <a name="new-worktrees"/>

```
git worktree add new-branch-name
cd new-branch-name
git merge main --allow-unrelated-histories
```

The last command (`git merge main --allow-unrelated-histories`) is required to populate the new worktree directory (remember that by default the `repo` directory is an empty detached HEAD). 
The option `--allow-unrelated-histories` is required because the detached head and `main` have no common previous commit so git would throw an error. 


## Setting up PR configuration <a name="pr-configuration"/>

Note that all rules for the same branch described below can be merged into a single rule. 

Below is a quick list of standard settings for the PR configuration. For more information, see the subsections below.
```
Settings > Code and automation > Rules > Rulesets > New branch ruleset
- Name: pr_main
- Bypass list: Repository Admin > Always allow
- Target branches: Default
- Restrict creations (to avoid creating release/*)
- Restrict updates (to avoid pushing directly to main)
- Restrict deletions (to avoid deleting main or release/*)
- Require a pull request before merging
    - Required approvals: 1
    - Dismiss stale pull request approvals when new commits are pushed
    - Require review from Code Owners
    - Require conversation resolution before merging
    - Allowed merge methods: Squash
- Require status checks to pass
    - Status checks that are required: e.g. tests (see the action's YAML in this repo)
- Block force pushes

Settings > General
- Pull requests
    - only Allow squash merging
    - Default commit message: Pull request title
    - Automatically delete head branches
- Issues
    - Auto-close issues with merged linked pull requests 
```

### Squash merging <a name="squash-merging"/>

Allow only `squash merging` for the PR. This can be done in `Settings > General > Pull Requests` by unselecting all the options except the `Allow squash merging`. 
The reason is that PRs should be small and thus there is no need to do a merging as it will lead to more commits to the main branch (with probably worse commit messages). 
In this sense, it is useful to set up the `Default commit message` to be the `Pull request title`.

### Require PR to merge to main branch

Go to `Settings > Code and automation > Rules > Rulesets` and create a new rule. In the configuration menu for this new rule, under `Rules > Branch rules`, select `Require a pull request before merging`. It is recommended to also select (in the advanced options): 
1. `Dismiss stale pull request approvals when new commits are pushed`
2. `Require review from Code Owners`
3. `Require conversation resolution before merging`
4. `Allowed merge methods > Squash` (see [Squash merging](#squash-merging))
Then, under `Targets > Target branches` in the configuration meny for this new rule, select `Add target > Include default branch`. 

### PR checks

If a GitHub action is set (see [GitHub actions for CI/CD pipeline](#ci-pipeline)), one can make it a check for a PR, meaning that the action must succeed before being able to merge the PR. 
This is useful for CI pipelines when merging to the main branch. 
To set this up, one must have specified a name for the build job in the GitHub action, which is done in the corresponding YAML file (e.g. `.github/workflows/action.yaml`):
```
name: ... # name of the GitHub action
jobs:
  build:
    name: ... # name of the build job of this GitHub action
```
Then, go to `Settings > Code and automation > Rules > Rulesets` and create a new rule. In the configuration menu for this new rule, under `Rules > Branch rules`, select `Require status checks to pass` and open `Show additional settings`. Select `+ Add checks` and write the (full) name of the build job and select the first item in the dropdown menu. This should lead to the addition of the check with the logo of GitHub and a text saying `GitHub Actions` next to it. Finally, fill the other options of the rule and save it. 

See section [GitHub actions for CI/CD pipeline](#ci-pipeline) to know how to set up the CI pipeline to run when opening and synchronizing a PR. 

## Managing the requirements for the python package <a name="managing-requirements"/>

The `pyproject.toml` file shouldn't have strict requirements (i.e. specific versions pinned) unless there is a known bug in the current version of the libraries used. In this repo there is an example of `pyproject.toml`. 

The specific versions should be pinned in the `requirements.txt` and the `requirements-dev.txt`. This can be done using `pip-tools` and the following commands:

```
# pip install pip-tools
pip-compile -o requirements.txt pyproject.toml
pip-compile --extra dev -o requirements_dev.txt pyproject.toml
```


## Publishing the package to PyPI <a name="publish-pypi"/>

The steps to perform are ([ref](https://packaging.python.org/en/latest/tutorials/packaging-projects/)):
1. Update the version number
1. Build the package
1. Use TestPyPI to test the release
1. Relase package in PyPI
1. Add tag and release in GitHub

### 1. Update the version number

Usually the version numbers can be found in the base `__ini__.py` and `pyproject.toml` files. If there is Sphinx documentation, there can also be a "release" number (which correspond to the version) in `docs/conf.py`. 

For example, if the current version is `0.9.1`, run 
```
grep -R "0.9.1"
```
to get all the instances of `version`.

Remember to run `git pull` on `main` to get all the latest changes before building the package!

### 2. Build the package

```
pip install --upgrade pip build setuptools wheel pkginfo packaging
python -m build
```
This should output a lot of text and once completed should generate two files in the `dist/` directory. The `.tar.gz` file is a source distribution whereas the `.whl` file is a built distribution. 

### 3. Test the release in TestPyPI

To securely upload your project, you’ll need a PyPI API token. Create one [here](https://test.pypi.org/manage/account/#api-tokens), setting the “Scope” to “Entire account”.

```
pip install --upgrade twine
python -m twine upload --skip-existing --repository testpypi dist/*
```

You will be prompted for a username and password. For the username, use `__token__`. For the password, use the token value, including the `pypi-` prefix.
Once uploaded, your package should be viewable on TestPyPI; for example: `https://test.pypi.org/project/example_package_YOUR_USERNAME_HERE`.

To check that the installation works, run the following commands:

```
pip install --upgrade virtualenv
virtualenv venv/
source ./venv/bin/activate
pip install --index-url https://test.pypi.org/simple/ --no-deps example-package-YOUR-USERNAME-HERE
pip install -r requirements.txt
python -c "import example-package-YOUR-USERNAME-HERE"
deactivate
rm -r venv/
```

### 4. Release in PyPI

If the package has been released correctly to TestPyPI, it can be released to PyPI using:

```
# (activate the initial virtual environment where we already have twine installed)
python -m twine upload --skip-existing dist/*
```

### 5. Add tag and release in GitHub

To add a tag (e.g. `v0.1.0`), run:
```
git tag <tagname>
git push origin --tags
```

In the "tags" tab in GitHub, click on the defined tag and then click on `Create release from tag`. To generate automatic notes, click on `Generate release notes`. 


## GitHub actions for CI/CD pipeline <a name="ci-pipeline"/>

GitHub has (limited) free runners to run the CI pipeline for the repository. 
This can be achieved by setting up a YAML file in the folder `.github/workflows`, see `.github/workflows/ci_pipeline.yaml`.
The current file sets up an SSH key so that the runner can clone repos that are not published in PyPI.
However, for it to work, one needs to set up two repository secrets. 
These can be added in `Settings > Secrets and variables > Actions > New repository secret` and create the following keys:
* `SSH_PRIVATE_KEY`
* `KNOWN_HOSTS`

The information stored in these keys can be obtained from the following two steps. 
Firstly, run `ssh-keyscan github.com` and search for the line that resembles `github.com ssh-rsa [KEY]`, `github.com ssh-ed25519 [KEY]`, or any other ssh key. Copy this line and store it in the secret `KNOWN_HOSTS`. 
Secondly, copy the data in `~/.shh/id_rsa`, `~/.ssh/id_ed25519` or the corresponding file as the one from the previous step and paste it in the secret `SSH_PRIVATE_KEY`. 

Note that one needs to update `.github/workflows/ci_pipeline.yaml` so that it knows which ssh encription to use, currently set up to `id_ed25519`. 

To make the CI pipeline run inside the PRs to `main` (i.e. when opening and when synchronizing them), add the following in the action's YAML file:
```
on:
  push:
    branches:
      - main
  pull_request:
    types:
      - opened
      - synchronize
    branches:
      - main
```

## Badges in `README.md` <a name="readme-badges"/>

Copy the lines below and change `PACKAGE_NAME` to the correct package name. 
```
[![Documentation Status](https://readthedocs.org/projects/PACKAGE_NAME/badge/?version=latest)](https://PACKAGE_NAME.readthedocs.io/en/latest/?badge=latest)
![example workflow](https://github.com/MarcSerraPeralta/PACKAGE_NAME/actions/workflows/ci_pipeline.yaml/badge.svg)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)
![PyPI](https://img.shields.io/pypi/v/PACKAGE_NAME?label=pypi%20package)
```

## Documentation hosted using ReadTheDocs <a name="readthedocs"/>

The documentation can be hosted in [ReadTheDocs](https://www.readthedocs.org/) for free if the projects are public. It is very simple to create a new documentation website: just click "Add project" and then follow the steps. In this repo, there is a setup example for documentation with Sphinx in `docs/`. 

Before publishing the documentation in ReadTheDocs, it is recommended to build it locally to check for bugs or warnings. To compile the documentation locally, run 
```
make html
```
inside `docs/` (note that here we are assuming that you have copied the `make.bat` and `Makefile` files from this repo. The first compilation can take some time. 
Then, just open one of the generated HTML files and check that the documentation looks correct locally. Then, push the changes to ReadTheDocs. 

By default, ReadTheDocs generates two documentation websites, "latest" and "stable", where "latest" corresponds to the last commit in the main branch, and "stable" corresponds to the latest tag in the repo. 


## Jupyter lab and notebook <a name="jupyter"/>

Even though Jupyter notebooks are not part of Python packages, they can be a tool to try out features and debug problems. When working with virtual environments, it is inefficient to have JupyterLab installed on all venvs because it takes quite a lot of disk space. A better option is to use the system/user's JupyterLab installation (so only one is installed) and then create a kernel for each venv. 
A problem with this solution is that if the kernels are not installed in the correct directory, they are not automatically loaded when creating Jupyter notebooks. The solution is to install the kernel in the same directory as the venv:
```
which jupyter-lab           # should point to the user installation
source <venv>/bin/activate  # activate venv
which jupyter-lab           # should still point to the user installation (if not, "pip uninstall jupyterlab")
pip install ipykernel       # install ipykernel in venv for creating a kernel
python -m ipykernel install --prefix "$VIRTUAL_ENV" --name <venv> # --display-name "(this is optional, by default <venv>)"
```
When installing the kernel in the venv, it can be found at `$VIRTUAL_ENV/share/jupyter/kernels/<venv>` and can be listed with
```
jupyter kernelspec list
```


## SSH tips <a name="ssh"/>

To **git clone a GitHub repo through SSH**, one needs to have (1) generated an SSH key on the machine on which to clone the repo, and (2) uploaded the SSH public key to your GitHub account. 
```
# check if you already have a key
ls ~/.ssh/id_ed25519.pub
# generate the key (if needed)
ssh-keygen -t ed25519 -C "your_email@example.com"
# display the public key
cat ~/.ssh/id_ed25519.pub
# copy it to GitHub > Profile > Settings > SSH and GPG keys > New SSH key > Add SSH key
# test it with:
ssh -T git@github.com
```

**Avoid having to type the login password in SSH** with:
```
# check if you already have a key
ls ~/.ssh/id_ed25519.pub
# generate the key (if needed)
ssh-keygen -t ed25519 -C "your_email@example.com"
# add the authorized key to the cluster
ssh-copy-id user@cluster-address
# test it with
ssh user@cluster-address
```


## Git tips <a name="git-tips"/>

**Avoid the `git push --set-upstream ...`** with:
```
git config push.autoSetupRemote true         # add --global to apply to everything, remove "true" to print the current value
```
if it still fails, then check the following:
```
git config push.default                      # if it does not print anything, then run the command below
git config push.default current              # add --global to apply to everything
```

**Remove all non-active branches in the local git repository** with:
```
git branch | grep -v '^[+*]' | grep -vE 'main|master|dev' | xargs -n 1 git branch -D
```
