# DBA Docs — MkDocs Setup

## First-time setup
```bash
pip install -r requirements.txt --break-system-packages
```

## Local preview (live reload on save)
```bash
mkdocs serve
# open http://127.0.0.1:8000
```

## Build static site (for hosting / sharing)
```bash
mkdocs build
# output in ./site/ — copy anywhere, e.g. internal web server, or serve via `python3 -m http.server` from site/
```

## Workflow
1. Add/edit `.md` files under `docs/<category>/`
2. `git add . && git commit -m "..." && git push`
3. `mkdocs build` (or wire to CI/cron) regenerates the HTML site — nav + search rebuilt automatically, zero manual index editing

## Git init (if not already a repo)
```bash
git init
git add .
git commit -m "init DBA docs"
git remote add origin <your-repo-url>
git push -u origin main
```

## Optional: auto-rebuild on push (cron, cheap)
```bash
# crontab -e, rebuild every 15 min if repo has changes
*/15 * * * * cd /path/to/dba-docs && git pull -q && mkdocs build -q
```
