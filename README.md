# delegpo

**Dele**gate **po**rts to your projects — a tiny Bash CLI that remembers which
port each of your projects uses and how to start it, then launches it by ID and
shows you what is running.

```
 delegpo · projects

   ID   PORT   STATUS  PROCESS       DIR            COMMAND
   ─────────────────────────────────────────────────────────────────────────
   web  5173   ● live  node:24305    ~/code/web     pnpm dev --port $PORT
   api  3000   ○ idle  -             ~/code/api     npm run dev

   2 project(s) · 1 live
```

## Requirements

- Bash 3.2+ (the default macOS Bash works)
- `lsof` (preinstalled on macOS and most Linux distros)

## Installation

delegpo is a single script. To call it as `delegpo` from any directory, copy it
into a `bin` folder that is on your `$PATH`.

### 1. Get the script

```sh
git clone git@github.com:kerupuksambel/delegpo.git
cd delegpo
```

### 2. Copy it into a bin folder

**System-wide** (needs `sudo`):

```sh
sudo cp delegpo /usr/local/bin/delegpo
sudo chmod +x /usr/local/bin/delegpo
```

> On macOS, use `/usr/local/bin`, not `/usr/bin`: `/usr/bin` is protected by
> System Integrity Protection and is read-only even with `sudo`. On Linux,
> `/usr/local/bin` is also the right place for tools you install yourself.

**Just for your user** (no `sudo`):

```sh
mkdir -p ~/.local/bin
cp delegpo ~/.local/bin/delegpo
chmod +x ~/.local/bin/delegpo
```

If `~/.local/bin` is not on your `$PATH` yet, add it to your shell config and
reload it:

```sh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc   # or ~/.bashrc for Bash
source ~/.zshrc
```

### 3. Run it from anywhere

```sh
which delegpo     # should print /usr/local/bin/delegpo or ~/.local/bin/delegpo
delegpo help
```

If your shell still says `command not found`, open a new terminal or run
`hash -r` so it picks up the new command.

### Updating

Pull the latest version and copy it over the old one:

```sh
cd delegpo && git pull
sudo cp delegpo /usr/local/bin/delegpo        # or: cp delegpo ~/.local/bin/delegpo
```

> **Tip:** if you'd rather have `git pull` update it automatically, symlink
> instead of copying: `ln -s "$PWD/delegpo" /usr/local/bin/delegpo`.

### Uninstalling

```sh
sudo rm /usr/local/bin/delegpo     # or: rm ~/.local/bin/delegpo
rm -r ~/.config/delegpo            # optional: also delete your registered projects and logs
```

## Quick start

```sh
# 1. Register a project: ID, port, command (directory defaults to where you are)
cd ~/code/api
delegpo add api 3000 npm run dev

# 2. Launch it from anywhere
delegpo api

# 3. See what's running and which ports are taken
delegpo status
```

## Commands

| Command | Description |
| --- | --- |
| `delegpo <id>` | Run the project's command in its directory (foreground). |
| `delegpo <id> -d` | Run it detached in the background, logging to a file. |
| `delegpo add [id] [port] [command]` | Register a project. Prompts for anything you leave out. |
| `delegpo edit <id>` | Edit a project interactively (Enter keeps the current value). |
| `delegpo edit` | Open the raw data file in `$EDITOR`. |
| `delegpo rm <id> [-f]` | Remove a project (`-f` skips confirmation). |
| `delegpo ls` | List all projects with live status, process and PID. |
| `delegpo info <id>` | Show one project in detail. |
| `delegpo status` | Port monitor: active projects plus every other occupied port. |
| `delegpo stop <id> [-f]` | Stop whatever is listening on the project's port (`-f` = `kill -9`). |
| `delegpo logs <id>` | Follow the log of a detached (`-d`) run. |
| `delegpo help` / `version` | Show help / version. |

Running `delegpo` with no arguments shows the project list (or the help if
nothing is registered yet).

### Registering projects

```sh
# Simple: everything after the port is the command
delegpo add api 3000 npm run dev

# Choose a different working directory
delegpo add web 5173 --dir ~/code/web pnpm dev

# Use -- when your command has its own flags, or quote it
delegpo add web 5173 --dir ~/code/web -- pnpm dev --port 5173
delegpo add docs 8000 'python3 -m http.server $PORT'

# Interactive: just run add and answer the prompts
delegpo add
```

Options for `add`:

- `-d, --dir DIR` — working directory (default: current directory)
- `-f, --force` — overwrite an existing ID / allow a duplicate port without asking

delegpo will warn you if the ID already exists or the port is already
registered to another project. IDs may contain letters, digits, `.`, `_` and
`-`, and cannot clash with command names (`ls`, `add`, `status`, …).

### Running projects

When you run `delegpo <id>`, delegpo:

1. Checks whether the port is already in use. If so, it tells you which process
   holds it and lets you **[k]ill it and continue**, **[r]un anyway**, or
   **[a]bort**.
2. `cd`s into the project directory.
3. Exports `PORT` and `DELEGPO_ID`, so your command can use `$PORT`
   (single-quote it when registering so it isn't expanded too early).
4. Executes the command with your `$SHELL` (override with `DELEGPO_SHELL`).

Add `-d` to run in the background. Output is appended to
`~/.config/delegpo/logs/<id>.log`; view it with `delegpo logs <id>` and stop the
project with `delegpo stop <id>`.

### Monitoring

```
$ delegpo status

 delegpo · port monitor

 ACTIVE PROJECTS
   ● web              :5173   node (pid 24305)

 OTHER OCCUPIED PORTS
     :5432   postgres (pid 2671)
     :7000   ControlCenter (pid 1384)
```

A project is considered **live** when something is listening on its registered
TCP port. delegpo does not verify the listener belongs to that project — for
example, a Docker-based project will show the Docker process.

## Configuration

| Variable | Default | Purpose |
| --- | --- | --- |
| `DELEGPO_HOME` | `~/.config/delegpo` (or `$XDG_CONFIG_HOME/delegpo`) | Where data and logs are stored. |
| `DELEGPO_SHELL` | `$SHELL` | Shell used to execute project commands. |
| `NO_COLOR` | unset | Set to disable colored output. |

## Data format

Projects are stored in `$DELEGPO_HOME/projects.tsv`, one per line, with
tab-separated fields:

```
<id>	<port>	<directory>	<command>
```

It's plain text — back it up, sync it, or edit it with `delegpo edit`.
Lines starting with `#` are ignored.
