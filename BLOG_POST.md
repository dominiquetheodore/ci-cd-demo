# Learning CI/CD: Building My First Automated Pipeline From Scratch

*Finally tackling the automation I've been putting off for way too long*

---

## The "Someday I'll Automate This" Phase

Like many developers I guess, I've been in that cycle: start a project, deploy manually a few times, promise myself "next time I'll set up CI/CD," then repeat the pattern.

This weekend, I finally stopped procrastinating and built my first proper CI/CD pipeline for a Flask app. And you know what? It was actually enjoyable once I got started.

## What I Built (My First CI/CD Project!)

This is my very first CI/CD setup, and I'm honestly surprised it works. I created a pipeline that automatically:

- Builds Docker images when I push code
- Pushes images to Docker Hub with version tags
- Deploys to my server without any manual steps
- Runs health checks to confirm everything works
- Cleans up old images to save space

When I push to `main`, everything happens automatically. This is all new to me, so I kept things as simple as possible.

## Why I Chose GitHub Actions

Sure, there are plenty of CI/CD tools—Jenkins, GitLab CI, CircleCI, Travis—but the choice paralysis is real. I went with **GitHub Actions** because it integrates seamlessly with where my code already lives.

I didn't want to juggle multiple platforms or disrupt my workflow. GitHub Actions works directly with my repositories, doesn't require maintaining another service, and honestly, has pretty good documentation. It's CI/CD without having to leave my development environment.

## What You'll Need to Follow Along

If you want to build something similar, here's what I used:

1. **GitHub account** (obviously)
2. **A Docker Hub account** (to store your container images) - [docker.com](https://docker.com)
3. **A VPS or cloud server** (I used Hetzner's cheapest option, about €5/month) - [hetzner.com](https://hetzner.com)
4. Basic familiarity with Docker and Flask (but honestly, you can learn as you go)

The total cost? About €5/month for the server. Everything else is free for personal use.

## The App (Simple But Perfect for Learning)

Here's the Flask app I'm deploying—nothing fancy, just enough to focus on the pipeline:

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Hello, World!"

@app.route("/health")
def health():
    return "Health check ok!"

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=5000, debug=False)
```

Perfect for learning because the app complexity stays out of the way.

## Testing the Waters: Manual Docker Push First

Before diving into automation, I wanted to make sure the basics worked. So I started with a manual Docker push:

```bash
docker build -t dominiquetheodore/silverlake:0.2.0 .
docker push dominiquetheodore/silverlake:0.2.0
```

Seeing that push succeed gave me confidence that:
1. My Dockerfile actually worked
2. I had proper access to Docker Hub
3. The image was buildable and pushable

Once I knew the manual process worked, I felt ready to automate it. This step-by-step approach helped me isolate issues—if the automation failed, I knew it wasn't because of a basic Docker problem.

## Starting Simple: A "Hello World" Action

Next, I tested GitHub Actions with the simplest possible workflow:

```yaml
name: Test Run

on: [push]

jobs:
  hello-world:
    runs-on: ubuntu-latest
    steps:
      - name: Say Hello
        run: echo "Hello from GitHub Actions!"
```

I added this to `.github/workflows/test.yml`, pushed, and watched it run. That little green checkmark was surprisingly satisfying. **Start with something that can't fail**—it builds confidence when you're just starting out.

## Step 1: Containerizing With Docker

Here's the Dockerfile that worked for me:

```dockerfile
FROM python:3.9-slim

WORKDIR /app

# Copy requirements first - this caching trick is a game-changer
COPY requirements.txt .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy the rest of the application
COPY . .

# Create and switch to non-root user (security best practice)
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 5000

# Gunicorn for production (Flask's dev server isn't for production)
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
```

**Key decisions:**
- **Smart caching**: Dependencies change less often than code
- **Security first**: Running as non-root is a good habit
- **Production-ready**: Gunicorn handles real traffic much better

## Step 2: The Real GitHub Actions Workflow

Here's the main pipeline (`.github/workflows/deploy.yml`):

```yaml
name: Build and Deploy Flask App

on:
  push:
    branches: [ main ]
  workflow_dispatch:  # For manual deployments when needed

env:
  IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/silverlake

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Log in to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: |
          ${{ env.IMAGE_NAME }}:latest
          ${{ env.IMAGE_NAME }}:${{ github.sha }}
        cache-from: type=registry,ref=${{ env.IMAGE_NAME }}:buildcache
        cache-to: type=registry,ref=${{ env.IMAGE_NAME }}:buildcache,mode=max
```

**What's happening here:**
1. Triggers on `main` pushes or when I manually run it
2. Sets up Docker with Buildx (better builds and caching)
3. Securely logs into Docker Hub using GitHub Secrets
4. Builds and pushes two image tags: `latest` and the commit SHA
5. Uses registry caching to speed up future builds

## Step 3: The Deployment (No Manual SSH!)

This job handles the actual deployment to my server:

```yaml
  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Deploy to server via SSH
      uses: appleboy/ssh-action@v0.1.5
      with:
        host: ${{ secrets.SERVER_HOST }}
        username: ${{ secrets.SERVER_USER }}
        key: ${{ secrets.SERVER_SSH_KEY }}
        script: |
          APP_NAME="silverlake"
          IMAGE="${{ secrets.DOCKERHUB_USERNAME }}/silverlake:latest"
          
          # Pull the latest image
          docker pull $IMAGE
          
          # Gracefully handle existing container
          docker stop $APP_NAME 2>/dev/null || true
          docker rm $APP_NAME 2>/dev/null || true
          
          # Start the new container
          docker run -d \
            --name $APP_NAME \
            --restart unless-stopped \
            -p 5000:5000 \
            $IMAGE
          
          # Give it a moment to start up
          sleep 5
          
          # Verify it's actually working
          if curl -f http://localhost:5000/health; then
            echo "🎉 Deployment successful!"
            # Clean up unused Docker images
            docker image prune -f
          else
            echo "❌ Health check failed!"
            # Show me what went wrong
            docker logs $APP_NAME
            exit 1
          fi
```

**Why this works well:**
- **Fully automated**: No manual steps required
- **Graceful updates**: Old containers are handled cleanly
- **Health verification**: We actually check that the deployment worked
- **Automatic cleanup**: Old images don't accumulate on the server

## Setting Up Secrets (One-Time Configuration)

All sensitive information lives securely in GitHub Secrets:

1. Go to your GitHub repository
2. Settings → Secrets and variables → Actions
3. Add these secrets:
   - `DOCKERHUB_USERNAME` (your Docker Hub username)
   - `DOCKERHUB_TOKEN` (generate in Docker Hub Account Settings → Security)
   - `SERVER_HOST` (your server IP or hostname)
   - `SERVER_USER` (SSH username, usually "root" or a sudo user)
   - `SERVER_SSH_KEY` (private SSH key for your server)

**Pro tip**: Use access tokens, never passwords, for Docker Hub.

## That First Successful Deployment

After getting everything configured, I made a tiny change and pushed to `main`. Watching the GitHub Actions tab update in real-time—each step turning green as it completed—was genuinely rewarding. Three minutes later, my change was live. No manual intervention. No "oops I forgot that step" moments.

It's my first time doing this, so seeing it actually work felt like a small victory.

## What I Learned (First-Time CI/CD Builder)

### 1. **Test Each Piece Separately**
Manually pushing the Docker image first helped me isolate issues. When the automation failed later, I knew it wasn't a basic Docker problem.

### 2. **Start Small and Iterate**
That "Hello World" workflow was crucial for building confidence when I was just getting started.

### 3. **GitHub Actions Is Surprisingly Approachable**
The YAML looks intimidating at first, but it's just a sequence of steps. Copy examples and tweak them.

### 4. **Documentation Is Your Friend**
The GitHub Actions marketplace shows you exactly what each action does and how to use it.

### 5. **Health Checks Are Simple But Useful**
Even as a beginner, adding that simple health check gave me more confidence in my deployments.

## What's Next: Where I Want to Take This

### 1. **Pull Request Integration**
Currently, the pipeline only runs on `main`. I want to add workflows that trigger on pull requests—maybe running tests or building preview environments. This would help catch issues before they reach production.

### 2. **Adding a Database**
My current app is stateless. To make this more realistic, I'd like to add PostgreSQL and learn how to handle:
- Database migrations as part of deployments
- Environment variables and secrets management
- Connection pooling and persistence

### 3. **Testing Pipeline**
I'm planning to integrate pytest to run tests automatically:
```yaml
- name: Run tests
  run: |
    pip install pytest pytest-cov
    pytest tests/ --cov=app --cov-report=xml
```
This is new territory for me, but it seems like a logical next step.

## Future Improvements I'm Considering

Once I'm more comfortable with the basics:
- **Staging environment**: Separate from production for testing
- **Database migration automation**: I'll need to learn how to do this properly
- **Monitoring integration**: Something like Sentry for error tracking
- **Multi-container setup**: Maybe add Redis or Nginx once I understand the single-container setup better

## See the Complete Code

You can check out the full implementation, including the Flask app, Dockerfile, and GitHub Actions workflows here:

**[https://github.com/dominiquetheodore/ci-cd-demo](https://github.com/dominiquetheodore/ci-cd-demo)**

Feel free to clone it, fork it, or use it as a starting point for your own CI/CD journey. It's my first attempt, so I'm sure there are improvements to be made!

## Wrapping Up

Building this pipeline took me a weekend. It's my very first CI/CD project, so I kept things intentionally simple. Starting with a manual Docker push, then a simple GitHub Action, then the full pipeline helped me build confidence step by step.

If you've been putting off learning CI/CD like I was, my advice is simple: pick a small project, test each piece manually first, then automate one step at a time. Don't worry about making it perfect or production-ready. Just make it work. The satisfaction of seeing your first automated deployment complete successfully is worth the effort.

The hardest part really is just starting. Once you do, each piece naturally leads to the next.

---

**Questions or suggestions?** I'm still learning! Check out the [code on GitHub](https://github.com/dominiquetheodore/ci-cd-demo) and feel free to open an issue or pull request if you have improvements. This is my first CI/CD project, so I welcome feedback from more experienced developers!