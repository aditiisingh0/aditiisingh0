name: Update README with Pinned Repos

on:
  schedule:
    - cron: '0 0 * * *'   # runs every day at midnight UTC
  push:
    branches: [main]
  workflow_dispatch:       # allows manual trigger from GitHub UI

jobs:
  update-readme:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - name: Checkout repo
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install requests

      - name: Fetch pinned repos and update README
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          USERNAME: aditiisingh0
        run: |
          python3 << 'EOF'
          import requests, os, re

          username = os.environ["USERNAME"]
          token    = os.environ["GITHUB_TOKEN"]

          # GraphQL query to get pinned repositories
          query = """
          {
            user(login: "%s") {
              pinnedItems(first: 6, types: REPOSITORY) {
                nodes {
                  ... on Repository {
                    name
                    description
                    url
                    stargazerCount
                    forkCount
                    primaryLanguage { name color }
                  }
                }
              }
            }
          }
          """ % username

          resp = requests.post(
            "https://api.github.com/graphql",
            json={"query": query},
            headers={"Authorization": f"Bearer {token}"}
          )

          nodes = resp.json()["data"]["user"]["pinnedItems"]["nodes"]

          # Build markdown table for pinned repos
          lines = []
          lines.append("<div align='center'>")
          lines.append("")
          lines.append("| Repository | Description | Stars | Forks | Language |")
          lines.append("|-----------|-------------|-------|-------|----------|")

          for repo in nodes:
              name  = repo["name"]
              desc  = repo.get("description") or "No description"
              url   = repo["url"]
              stars = repo["stargazerCount"]
              forks = repo["forkCount"]
              lang  = repo["primaryLanguage"]["name"] if repo["primaryLanguage"] else "—"
              lines.append(f"| [**{name}**]({url}) | {desc} | ⭐ {stars} | 🍴 {forks} | `{lang}` |")

          lines.append("")
          lines.append("</div>")

          new_section = "\n".join(lines)

          # Replace content between markers in README
          with open("README.md", "r") as f:
              content = f.read()

          pattern = r"(<!-- PINNED-REPOS-START -->).*?(<!-- PINNED-REPOS-END -->)"
          replacement = f"<!-- PINNED-REPOS-START -->\n{new_section}\n<!-- PINNED-REPOS-END -->"
          updated = re.sub(pattern, replacement, content, flags=re.DOTALL)

          with open("README.md", "w") as f:
              f.write(updated)

          print("README updated successfully!")
          EOF

      - name: Commit and push changes
        run: |
          git config --global user.name  "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.com"
          git add README.md
          git diff --staged --quiet || git commit -m "chore: auto-update pinned repos [skip ci]"
          git push
