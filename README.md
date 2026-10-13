# 100 Days: Zero to One AI Engineer

The public site for the program: the philosophy, orientation slides, how each week works, the paths, what you build, and sign-up.

Live site: https://dpaul0501.github.io/ai-engineer-100/

The daily guide, labs, and starter code live in the private labs repo.

## Run locally

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python -m mkdocs serve
```

Open `http://127.0.0.1:8000`. The orientation deck is at `/orientation.html`. Use the arrow keys to move between slides, and Esc for the overview.

## Publish

Every push to `main` builds the site with `mkdocs build --strict` and deploys it to GitHub Pages through `.github/workflows/pages.yml`.
