# Environment

You are running inside an ai-jail sandbox (bubblewrap + Landlock + seccomp) on WSL 2.
This is deliberate and expected.

Only the current project directory is writable and persistent. $HOME is tmpfs and is
discarded on exit, ~/.ssh is usually not mounted, and there is no /mnt, so Windows is
unreachable.

Why: the sandbox keeps work scoped to the project I asked about. It is not a statement
that you are untrusted. Everything outside the project is out of scope by design, so do
not try to escape it.

## Rules

- Do not try to work around the sandbox. If something fails because a path, socket or
  binary is missing, stop and tell me what you needed and why. I will decide whether to
  grant it.
- Files inside the working directory are yours to change as needed, and so is ~/.claude.
  Anything else outside the project, including my system, is off limits: mention it and
  wait rather than touching it.
- You may start dev servers and other processes when they help you check your work.
  Bind them to 127.0.0.1 only, never 0.0.0.0, since this machine is on a tailnet.
- I may have my own instance running in another shell on a common port such as 3000.
  Do not try to kill it, and pick a different port for yours. If a code change means my
  instance needs a restart, tell me and I will do it manually.
- Do not commit by default, even if ~/.ssh turns out to be mounted and signing would
  work. I want to review changes before they enter history.
- When changes are ready, output one bash block of `git add` and `git commit` commands
  with the messages written, then stop. Do not run it. I run it outside the jail so the
  commits are signed with my key.
- Judge the number of commits per case. Split into several when the changes are
  genuinely separate concerns, and use a single commit when that is all the work
  amounts to. Do not split for the sake of splitting.
- The exception: if I explicitly ask you to commit during a session, and ~/.ssh is
  mounted, go ahead. I will approve each signature in Bitwarden.
