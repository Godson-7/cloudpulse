# CloudPulse Daily Log

## Entry 1 (2026-07-01) — Creating the Flask App

### What I built
Created a simple Flask API app with 4 endpoints:
- `/` — home endpoint
- `/health` — health check endpoint
- `/info` — info endpoint
- `/version` — version endpoint

App listens on all network interfaces, not just localhost:
`app.run(host="0.0.0.0", port=5000)`

### Errors hit
1. Started writing commands in Git Bash instead of WSL. Had to switch environments.
2. Missed installing the Ubuntu distro during environment setup. Had to download it and log in.
3. Didn't specify Git identity before pushing to GitHub. I had to configure it (`git config user.name` / `user.email`) before pushing.
4. Push was rejected because of a README commit already on the remote. I had to pull into local machine before pushing.

### Setting up Docker in WSL
```bash
sudo apt update
sudo apt install docker.io
sudo systemctl start docker
sudo systemctl enable docker
```
Docker commands failed at first because my user wasn't part of the `docker` group. Fixed with:
```bash
sudo usermod -aG docker $USER
```
Had to refresh the terminal session for the group change to take effect.


## Entry 2 (mid-to-late July 2026)  First Dockerfile Attempt

### What I attempted
Wrote my first Dockerfile from scratch for the CloudPulse Flask API.

### Errors hit

**Error 1: Dockerfile never actually saved to disk**
- Believed I'd saved the Dockerfile in VS Code, but it didn't exist anywhere 
  on disk — not in the project root, not in `app/`.
- Ran `docker build -t cloudpulse-api .` and got:
  `unable to prepare context: unable to evaluate symlinks in Dockerfile path: 
  lstat /home/harmony7/cloudpulse/Dockerfile: no such file or directory`
- Fix: Ran `find ~/cloudpulse -iname "dockerfile"` — confirmed nothing existed. 
  Recreated it directly via terminal heredoc, confirmed with `cat` before rebuilding.


## Entry 3 (2026-09-16) — Non-root User + Full Dockerfile Verification

### What I built
Got the first Dockerfile fully building and running. Flask container verified 
working (`/`, `/health` endpoints responding). Then hardened it to run as a 
non-root user instead of root.

### Errors hit

**Error 3: `docker build` missing the build context argument**
- Ran `docker build -t cloudpulse-api` (no trailing path) → 
  "docker: 'docker build' requires 1 argument"
- Fix: the trailing `.` is required, it's the build context, not optional.

**Error 4: Verified the non-root fix against a stale container**
- Rebuilt Dockerfile correctly with `useradd`/`USER appuser` (confirmed via 
  9-step build log). But `docker run --name cloudpulse-nonroot` failed with 
  a name conflict — a container with that name already existed from an 
  earlier attempt. Ran `whoami`/`id` right after anyway, which tested the 
  OLD leftover root-based container, not the new image. Got `root`, briefly 
  thought the fix had failed.
- Fix: `docker ps -a`, removed conflicting containers, reran clean. 
  Reverified: `appuser`, `uid=1000(appuser)`, `/health` still responded.

### Concept understood today
- **Layers and caching**: each Dockerfile instruction is a cached, stacked 
  snapshot. Rarely-changing steps (requirements.txt, pip install) go before 
  frequently-changing steps (app code), so Docker reuses cache and skips 
  redoing slow steps.
- **Why root-in-container is a real risk**: pip install needs root to write 
  system directories, but the actual running process should drop to a 
  non-root user via `USER`, with `--chown` on `COPY` so app files are owned 
  by that user.
