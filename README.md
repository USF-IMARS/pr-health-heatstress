
# Getting Started
## Initial Setup

This setup requires a unix-like CLI.
Recommended to run on ubuntu.
If starting from a windows machine it is recommended to use Windows Subsystem for Linux (WSL).
To do so open a PowerShell with admin priveledges and run `wsl --install -d Ubuntu`.
Then you can open ubuntu from the start menu, open a terminal, and continue.

```{bash}
# download the code
git clone git@github.com:USF-IMARS/pr-health-heatstress.git

# create & activate a virtual environment
python3 -m venv .venv
source .venv/bin/activate

# install dependencies
pip install -r requirements.txt

# GEE setup
earthengine authenticate
```

The software is now installed.
Continue to the Development Workflow section below.

## Development Workflow
```{bash}
# get the latest code
git pull

# activate the environment
source .venv/bin/activate

# Edit files as you like...

# build the book
jupyter-book build .

# view the book's html files to check your edits.
# edit & rebuild as needed.

# use git VCS to save changes to github
git status
# `git add` any new files
git commit -a -m "PUT YOUR DESCRIPTION OF EDITS HERE"
git push
```

## Updating the Website
If you are an admin and want to update the website:
```{bash}
# publish to GitHub Pages
ghp-import -n -p -f _build/html
```