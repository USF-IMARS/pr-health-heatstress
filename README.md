
# build & deploy
jb build . && ghp-import -n -p -f _build/html



# setup
## Python toolchain
python3 -m venv .venv
source .venv/bin/activate
pip install -U pip jupyter-book ghp-import

pip install earthengine-api geemap