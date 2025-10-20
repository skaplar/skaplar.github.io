---
layout: post
title: "Deploying personal and hobby apps to DigitalOcean with GitHub Actions"
date: 2025-10-20
categories: [rails, devops, deployment]
description: "A simple and inexpensive CI/CD setup for deploying a Rails app to DigitalOcean using GitHub Actions and Docker."
download: true
---

As the lead backend developer, I usually focus on architecture, data models, and APIs, not on deployments.  
But with how easy and cheap it’s become to host on platforms like **Hetzner** or **DigitalOcean**, setting up a fully automated deployment pipeline has never been simpler. And of course I had a need to do it.

In this post, I’ll share the **first version** and hobby version of my Rails CI/CD workflow. 
Simple, functional, and fully automated.  


Later, I’ll share an improved version with multiple steps choosing the environment and similiar.


## Why I Tried This Setup

I wanted a **cheap, self-hosted, reproducible** way to deploy my Rails app without relying on heavy platforms like Heroku or Render.  


Developers usually come to a point where they make something, and they run it localhost, where their journey ends. However, now it's easier then ever to push something out in the open and test it. Give it to the users, and try something for yourself.

All I needed was:
- A **DigitalOcean droplet** with Docker and Docker Compose installed
- My Rails app already containerized  
- A **GitHub Actions** workflow to build, test, push, and deploy  

That’s it. Each push to the `dev` branch triggers a full CI/CD cycle, from RSpec tests to deployment on the droplet.


## The GitHub Actions Workflow

Here’s the complete workflow I started with.  
Have in mind this is a **hobby** workflow, to have something up and running quickly, not production ready one.  
It runs tests, builds the Docker image, pushes it to the GitHub Container Registry (GHCR), and deploys the latest version to the DigitalOcean droplet via SSH.

{% raw %}
```yaml
name: Hobby APP Rails CI

on:
  push:
    branches: [ dev ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: githubusername/reponame

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v2
      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.1.2'
      - name: Build and test with RSpec
        env:
          RAILS_ENV: test
        run: |
          gem install bundler
          bundle install --jobs 4 --retry 3
          bin/rails assets:precompile
          bundle exec rspec
      - name: Upload Screenshots
        uses: actions/upload-artifact@v2
        if: failure()
        with:
          name: screenshots
          path: /home/runner/work/projectName/projectName/tmp/capybara/

  build_and_push:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v2
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v1
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Extract metadata (tags, labels) for Docker
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
      - name: Build and push
        uses: docker/build-push-action@v2
        with:
          context: .
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          labels: ${{ steps.meta.outputs.labels }}

  deploy:
    needs: build_and_push
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Setup environment variables for deployment
        run: |
          echo "IMAGE_TAG=${{ github.sha }}" > .env
          echo "APPLICATION_HOST=${{ secrets.DROPLET_IP}}" >> .env
          echo "DB_HOST=${{ secrets.DB_HOST }}" >> .env
          echo "DB_PORT=${{ secrets.DB_PORT }}" >> .env
          echo "DB_NAME=${{ secrets.DB_NAME }}" >> .env
          echo "DB_USERNAME=${{ secrets.DB_USERNAME }}" >> .env
          echo "DB_PASSWORD=${{ secrets.DB_PASSWORD }}" >> .env
      - name: Copy docker-compose.yml to DigitalOcean droplet
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.DROPLET_IP }}
          username: ${{ secrets.DROPLET_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: ".env, docker-compose.yml"
          target: /home/myapp
          overwrite: true
          rm: true
      - name: Deploy to DigitalOcean
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.DROPLET_IP }}
          username: ${{ secrets.DROPLET_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            mkdir -p /home/myapp
            cd /home/myapp
            echo "${{ secrets.RAILS_MASTER_KEY }}" > master.key
            docker-compose stop web sidekiq
            docker rm web sidekiq || true
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            docker-compose up -d web sidekiq
            docker ps -a
            docker image prune -a --force --filter "until=24h"
```
{% endraw %}


## The Docker Compose Setup (for Dev)

On the droplet, the app runs through `docker-compose.yml`.  
This file defines services for Rails (`web`), background jobs (`sidekiq`), and local dependencies (`postgres`, `redis`, `mail`).

```yaml
version: '3.8'

x-common-environment: &common-env
  DB_HOST: postgres
  DB_PORT: 5432
  DB_NAME: dbname_development
  DB_USERNAME: dbusername
  DB_PASSWORD: dbpassword
  REDIS_URL: redis://redis:6379/0
  RUN_MIGRATIONS: "false"

services:
  web:
    image: ghcr.io/skaplar/eusluge:${IMAGE_TAG}
    environment:
      <<: *common-env
      SMTP_ADDRESS: mail
      APPLICATION_HOST: ${APPLICATION_HOST}
      APPLICATION_PORT: 8080
      RAILS_ENV: development
    volumes:
      - /home/myapp/master.key:/usr/src/app/config/master.key
    ports:
      - "8080:3000"
    depends_on:
      - postgres
      - sidekiq
      - mail

  postgres:
    image: postgres:9.5.8-alpine
    ports:
      - "5433:5432"
    environment:
      POSTGRES_DB: dbname_development
      POSTGRES_USER: dbusername
      POSTGRES_PASSWORD: dbpassword
    volumes:
      - postgres_data:/var/lib/postgresql/data

  sidekiq:
    image: ghcr.io/githubusername/reponame:${IMAGE_TAG}
    command: sh -c "bundle exec rails db:migrate && bundle exec sidekiq"
    environment:
      <<: *common-env
      SMTP_ADDRESS: mail
    depends_on:
      - redis

  redis:
    image: redis:latest
    ports:
      - "127.0.0.1:6379:6379"

  mail:
    image: axllent/mailpit:latest
    ports:
      - "1025:1025"
      - "8025:8025"

volumes:
  postgres_data:
```

## A Few Notes

- The workflow builds your Rails image once, pushes it to **GHCR**, and the droplet only needs to **pull and restart**.
- All secrets are securely stored in **GitHub Actions secrets**.
- Mailpit and redis are available only locally, which is ideal for development.
- The `.env` file is generated dynamically during each deploy, so you never commit secrets.


## Next Steps

In the next post, I’ll share an improved version. Where you can separate the database to a managed one, where you can have
multistep deployment, selection of the environement and similiar.

If you’ve ever thought *“I don’t do DevOps”*, this setup shows you can, in just a single YAML file.

---


# Have you tried [SummarAIzeIT](https://summaraizeit.com) yet?

