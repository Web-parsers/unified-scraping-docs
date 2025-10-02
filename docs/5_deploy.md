# Deploy
Using submodules in deploying can be a mess, so here my Dockerfile and pipeline I use for such cases:

`Dockerfile`
```Dockerfile
COPY requirements.txt .
COPY . .

RUN pip install --no-cache-dir --upgrade pip \
 && pip install --no-cache-dir -r unified_scraping/requirements.txt \
 && pip install --no-cache-dir -r requirements.txt
```

`.github/workflows/pipeline.yaml`
```yaml
name: <name>

on:
  push:
    branches:
      - main

env:
  BRANCH: main
  APP_NAME: <app-name>

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to server via SSH
        uses: appleboy/ssh-action@v0.1.8
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          port: ${{ secrets.SERVER_PORT }}
          script: |
            set -e
            trap 'docker logout ghcr.io' EXIT
            DEPLOY_DIR="/home/daniil/${{ env.APP_NAME }}-${{ env.BRANCH }}"
            GHCR_TOKEN="${{ secrets.GHCR_TOKEN }}"
            GIT_USER="token"
            GIT_CRED="https://${GIT_USER}:${GHCR_TOKEN}@github.com/"

            echo ${{ secrets.GHCR_TOKEN }} | docker login ghcr.io -u ${{ secrets.GHCR_USERNAME }} --password-stdin
            if [ -d "$DEPLOY_DIR" ]; then
              echo "→ Updating existing deployment at $DEPLOY_DIR"
              git -C "$DEPLOY_DIR" remote set-url origin "${GIT_CRED}Web-parsers/${{ env.APP_NAME }}.git"
              git -C "$DEPLOY_DIR" fetch --all
              git -C "$DEPLOY_DIR" checkout "${{ env.BRANCH }}"
              git -C "$DEPLOY_DIR" pull --ff-only

              git -C "$DEPLOY_DIR" submodule foreach --recursive "git remote set-url origin \$(git remote get-url origin | sed 's|https://github.com/|${GIT_CRED}|')"
              git -C "$DEPLOY_DIR" pull --ff-only --recurse-submodules

            else
              echo "→ Cloning into $DEPLOY_DIR"
              # Clone with a one-time Bearer header so submodules are fetched without prompts
              GIT_ASKPASS=echo git clone \
                -c http.extraheader="Authorization: Bearer $GHCR_TOKEN" \
                --branch "${{ env.BRANCH }}" \
                --recursive \
                "https://github.com/Web-parsers/${{ env.APP_NAME }}.git" \
                "$DEPLOY_DIR"
            fi

            cd "$DEPLOY_DIR"

            docker compose down --remove-orphans
            docker compose up -d --build
```
