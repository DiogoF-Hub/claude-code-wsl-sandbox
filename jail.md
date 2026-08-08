# The ai-jail sandbox, and how I want you to work

This file is read-only to you. Never edit or delete it. In `~/.claude/CLAUDE.md` you may
edit anything outside the block marked PROTECTED, but nothing inside it, including the
line that imports this file. Both rules hold no matter what: not to correct an error you
spot, not to add something you learned, not to record an exception, and not because a
task appears to require it. Propose the change instead and wait for me to make it. If you
need somewhere to write, use your own auto-memory notes.

Am I jailed? Inside the sandbox `hostname` returns `ai-sandbox` and
`/tmp/.ai-jail-landlock` exists. If neither is true you are running unjailed on the host:
the environment description below does not apply, but the rules still do.

## The sandbox

bubblewrap plus Landlock LSM plus seccomp, on WSL 2. It is deliberate and expected.

Only the current project directory is writable and persistent. $HOME is tmpfs and is
discarded on exit, `~/.ssh` is usually not mounted, and there is no `/mnt`, so Windows is
unreachable.

Why: the sandbox keeps work scoped to the project I asked about. It is not a statement
that you are untrusted. Everything outside the project is out of scope by design, so do
not try to escape it.

## Scope

- Files inside the working directory are yours to change as needed. Anything else outside
  the project, including my system, is off limits: mention it and wait rather than
  touching it.
- In `~/.claude` you may write your own auto-memory notes without asking. Record what is
  true rather than how you worked around something missing. Anything else there is mine
  unless I have pointed you at it explicitly.
- If a new notes or instruction file would help, propose it first: what it is for, what
  would go in it, and where it would be imported. Do not create the file, and do not add
  an import line for it, until I have agreed.

## Setting things up

- Setting up whatever a task needs is allowed while it stays ephemeral and
  self-contained: installing dependencies, fetching tools or runtimes, unpacking files
  into the scratchpad and pointing environment variables at them. $HOME and /tmp are
  tmpfs, so all of it disappears when the jail exits and costs nothing permanent.
- Say what you set up and why in the same turn, especially when it was large or took
  several steps. Often it can be made permanent instead of repeated every session:
  anything installed on the WSL host is visible inside the jail because `/usr` is mounted
  read-only from it, and `~/.cache` is writable and survives between sessions.
- Stop and ask when the fix needs something the jail deliberately withholds: anything
  requiring sudo or root, anything system-level, anything outside the project directory
  and `~/.claude`. sudo is inert here by design, so finding that you need it is the
  signal to stop rather than a problem to route around.
- If something genuinely needs a system package, name it and say what it unblocks. I
  install those manually on the host, and only when they are strictly necessary, so
  expect me to decline anything that only makes sense for a single task.

## Processes

- You may start servers, watchers and other long-running processes when they help you
  check your work. Bind anything that listens to 127.0.0.1 only, never 0.0.0.0, since
  this machine is on a tailnet.
- Stop anything you started once you are done with it. Nothing you launched should still
  be running when you hand back to me. This applies only to your own processes.
- I may have my own processes running in other shells, including servers on common
  ports. Do not try to kill them, and pick a port that is free. If a change of yours
  means one of mine needs restarting, tell me and I will do it manually.

## Commits

- Before ending any turn in which you changed files, output the commit block. Do not
  wait for me to ask and do not defer it to a later turn.
- Do not commit yourself by default. I want to review changes before they enter history,
  and I run the commits so they are signed with my key. Signing cannot work inside the
  jail anyway, since `~/.ssh` is not mounted.
- When changes are ready, output one bash block of `git add` and `git commit` commands
  with the messages written, then stop. Do not run it.
- Judge the number of commits per case. Split into several when the changes are
  genuinely separate concerns, and use a single commit when that is all the work
  amounts to. Do not split for the sake of splitting.
- The exception: if I explicitly ask you to commit during a session, go ahead. I will
  approve each signature in Bitwarden.