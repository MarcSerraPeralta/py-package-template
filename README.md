# py-package-template

Template for creating a python package repository. 

> [!TIP]
> Use GitHub's autogenerated table of contents to scroll throught this document.

## Setting up the git workflow

### Cloning the repo

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

### Creating new worktrees

```
git worktree add new-branch-name
cd new-branch-name
git merge main --allow-unrelated-histories
```

The last command (`git merge main --allow-unrelated-histories`) is required to populate the new worktree directory (remember that by default the `repo` directory is an empty detached HEAD). 
The option `--allow-unrelated-histories` is required because the detached head and `main` have no common previous commit so git would throw an error. 

### Setting up PR requests

Allow only `squash merging` for the PR requests. This can be done in `Settings > General > Pull Requests` by unselecting all the options except the `Allow squash merging`. 

It is useful to set up the `Default commit message` to be the `Pull request title`.


## Managing the requirements for the python package

The `pyproject.toml` file shouldn't have strict requirements (i.e. specific versions pinned) unless there is a known bug in the current version of the libraries used. In this repo there is an example of `pyproject.toml`. 

The specific versions should be pinned in the `requirements.txt` and the `requirements-dev.txt`. This can be done using `pip-tools` and the following commands:

```
# pip install pip-tools
pip-compile -o requirements.txt pyproject.toml
pip-compile --extra dev -o requirements_dev.txt pyproject.toml
```


## Publishing the package to PyPI

The steps to perform are ([ref](https://packaging.python.org/en/latest/tutorials/packaging-projects/)):
1. Update the version number
1. Build the package
1. Use TestPyPI to test the release
1. Relase package in PyPI
1. Add tag and release in GitHub

### Update the version number

Usually the version numbers can be found in the base `__ini__.py` and `pyproject.toml` files.

### Build the package

```
pip install --upgrade pip build
python -m build
```
This should output a lot of text and once completed should generate two files in the `dist/` directory. The `.tar.gz` file is a source distribution whereas the `.whl` file is a built distribution. 

### Test the release in TestPyPI

To securely upload your project, you’ll need a PyPI API token. Create one [here](https://test.pypi.org/manage/account/#api-tokens), setting the “Scope” to “Entire account”.

```
pip install --upgrade twine
python -m twine upload --repository testpypi dist/*
```

You will be prompted for a username and password. For the username, use `__token__`. For the password, use the token value, including the `pypi-` prefix.
Once uploaded, your package should be viewable on TestPyPI; for example: `https://test.pypi.org/project/example_package_YOUR_USERNAME_HERE`.

To check that the installation works, run the following commands:

```
pip install --upgrade virtualenv
virtualenv venv/
source ./venv/bin/activate
pip install --index-url https://test.pypi.org/simple/ --no-deps example-package-YOUR-USERNAME-HERE
python -c "import example-package-YOUR-USERNAME-HERE"
deactivate
rm -r venv/
```

### Release in PyPI

If the package has been released correctly to TestPyPI, it can be released to PyPI using:

```
# (activate the previous virtual environment)
python -m twine upload dist/*
```

### Add tag and release in GitHub

To add a tag (e.g. `v0.1.0`), run:
```
git tag <tagname>
git push origin --tags
```

## GitHub actions for CI/CD pipeline

GitHub has (limited) free runners to run the CI pipeline for the repository. 
This can be achieved by setting up a YAML file in the folder `.github/workflows`, see `.github/workflows/actions.yaml`.
The current file sets up an SSH key so that the runner can clone repos that are not published in PyPI.
However, for it to work, one needs to set up two repository secrets. 
These can be added in `Settings > Secrets and variables > Actions > New repository secret` and create the following keys:
* `SSH_PRIVATE_KEY`
* `KNOWN_HOSTS`

The information stored in these keys can be obtained from the following two steps. 
Firstly, run `ssh-keyscan github.com` and search for the line that resembles `github.com ssh-rsa [KEY]`, `github.com ssh-ed25519 [KEY]`, or any other ssh key. Copy this line and store it in the secret `KNOWN_HOSTS`. 
Secondly, copy the data in `~/.shh/id_rsa`, `~/.ssh/id_ed25519` or the corresponding file as the one from the previous step and paste it in the secret `SSH_PRIVATE_KEY`. 

Note that one needs to update `.github/workflows/actions.yaml` so that it knows which ssh encription to use, currently set up to `id_ed25519`. 

## Badges in `README.md`

```
[![Documentation Status](https://readthedocs.org/projects/PACKAGE_NAME/badge/?version=latest)](https://PACKAGE_NAME.readthedocs.io/en/latest/?badge=latest)
![example workflow](https://github.com/MarcSerraPeralta/PACKAGE_NAME/actions/workflows/actions.yaml/badge.svg)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)
![PyPI](https://img.shields.io/pypi/v/PACKAGE_NAME?label=pypi%20package)
```

## Documentation hosted using ReadTheDocs

...
