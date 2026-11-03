# 100 Days: Zero to One AI Engineer

The public site for the program: the philosophy, orientation slides, how each week works, the paths, what you build, and sign-up.

Live site: https://dpaul0501.github.io/ai-engineer-100/

The weekly guide, labs, and starter code are in [ai-engineer-100-labs](https://github.com/dpaul0501/ai-engineer-100-labs).

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

## License

The content of this site is licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/); the full text is in [`LICENSE`](LICENSE). You may share and adapt it for any purpose, including commercially, as long as you credit "AI Engineer 100, Debjyoti Paul", link to the license, and note any changes. Linked papers, videos, posts and problems remain under their owners' terms.
