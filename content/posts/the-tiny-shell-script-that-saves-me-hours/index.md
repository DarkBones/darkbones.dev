+++
title = "The Tiny Shell Script That Cut My Feedback Loop to four Seconds"
date = "2025-04-14T05:29:21+01:00"
author = "DarkBones"
authorTwitter = "" #do not include @
cover = ""
tags = []
keywords = []
description = ""
showFullContent = false
readingTime = true
hideComments = true
draft = true
+++

**In the 15 years I've been writing code, I've built and collected a lot of tools to make my work easier.**
Some have been with me for so long, they feel like muscle memory. I reach for them without even thinking. Today, I want to show you one of those tools.

If you use Postgres in your projects, you'll probably find this useful. If you use a different database, you can likely adapt it with a few small tweaks.

> Debug faster. Cry less.

# 🔁 We've all been there.

You're working on a bug that takes at least 26 steps to reproduce. You make a code change, sigh, and slog through all 25 steps just to watch the 26th fail again.

> Rinse. Repeat. Raise your PR. Cry a little.

One day, I got tired of the sighing, the slogging, and the crying. I wrote a tiny shell script to snapshot and restore my local Postgres database in seconds. Now, when I’m chasing a bug or testing a tricky migration, I don’t waste time rebuilding state. I just make a code change, hit save, and I'm back to exactly where I need to be.

[Circular diagram of the OODA loop with four stages—Observe, Orient, Decide, Act—arranged clockwise around the center text "OODA LOOP." Each stage is labeled in bold black letters along the outer edge of a blue circular arrow, illustrating the continuous cycle of decision-making.]

# 🧠 The OODA Loop And Why It Matters

The OODA loop stands for **observe**, **orient**, **decide**, **act**. It's a military strategy framework that's since been adopted in tech, product design, and competitive fields where fast iteration is the key to winning.

> Whoever has the shortest OODA loop wins.

Whether you're familiar with the term or not, every time you test a change, you're going through the OODA loop. You **observe** the behaviour, **orient** your thinking based on what went wrong or right, **decide** what to tweak, and **act** by making a code change.

If it takes 60 seconds to go from *"I saved this file"* to *"I saw what happened,"* your loop is slow. If it takes four seconds, you can run dozens of iterations in the time someone else is still manually rebuilding state.

> A slow loop doesn't just slow you down. It kills your momentum. It kills your fun.

The little script brought my loop down to **four seconds**. It's only three shell files and under 100 lines of code, but it's easily one of the most valuable tools I've ever written and I use it literally all the time.

# ⚠️ Disclaimer

This isn't a shiny, production-ready open-source tool. I didn't write it for you.

I wrote it for myself, because I was tired of wasting time and brain cycles on pointless repetition. It lived in my dotfiles, undocumented and full of hardcoded defaults, for longer than I'd like to admit.

For this article, I've cleaned it up *just enough* to make sense outside my brain. If you want to use it, clone it, tweak it, or steal ideas from it, go ahead. Just don't ask me to maintain it.

# 📦 Dependencies

- **fzf**: Okay, maybe it's a *little* bit shiny. I did take the time to integrate **fzf**, so I can pick snapshots from a fuzzy menu instead of remembering them and typing them out like some animal. To get the best results, I highly recommend having [**fzf**](https://github.com/junegunn/fzf) installed on your system.
- Environment variable `DEV_HOME`: The script needs to know where you want to put your snapshot files. I have a `DEV_HOME` environment variable set on all my machines. It's usually just a directory called `Developer` that sits in my home directory. *Fun fact: I name this directory `Developer` because macs automatically add a nice 🛠️ icon to it (try it).*

# 🎬 Demo

[gif of usage]

What did you just see me **do**?
- I created a database dump named `'hello'` (I name them after JIRA IDs, but `'hello'` works for the demo)
- I queried the `content` field of the `notes` table, there's only one row with `'This is the default content'`
- I updated the `content` field to say `'Imagine this takes 26 steps to set...'`
- I ran `dbrestore`. Because I didn't specify a snapshot name, it gave me a list of available options (thanks, **fzf**!)
- The `content` field is now set to `'This is the default content'` again

# 🛠️ File 1: Utils

Every software project that has more than one file must contain at least one `utils` file. This is the law. It's also why software projects with exactly two files cannot exist. The second file instantly becomes `utils.sh` and the cycle begins.

Let's take a look at the utility functions we have here:

```sh
# zsh/.zsh/scripts/_drestore_utils.sh
#!/bin/bash

get_snapshot_dir() {
    local repo_name=$(basename "$(git config --get remote.origin.url)" .git | sed 's/^\.//')
    local snapshot_dir="$DEV_HOME/db_snapshots/$repo_name"

    mkdir -p "$snapshot_dir"
    echo "$snapshot_dir"
}

load_drestore_config() {
    local snapshot_dir="$1"
    local config_file="$snapshot_dir/config.env"

    DB_PASSWORD="password"
    EXCLUDED_TABLES="sessions"
    DB_USER="postgres"
    DB_SCHEMA="public"
    DB_HOST=127.0.0.1
    DB_NAME="postgres"

    [ -f "$config_file" ] && source "$config_file"
}

excluded_clause() {
    printf "'%s'," $EXCLUDED_TABLES | sed 's/,$//'
}
```

`get_snapshot_dir()` This just returns the directory in which I want to store my snapshots. A fancy git command returns the name of the repo I'm working on so I can have a subdirectory per repo. For example, if I'm in a repo called `my_repo`, then `get_snapshot_dir()` would return:
`~/Developer/db_snapshots/my_repo`

The script requires some configuration per project, which are loaded by `load_drestore_config()`. It comes with some sensible defaults that work for me. If you want to override the defaults, you can do so by creating a `config.env` file inside the snapshot directory. For example, if the database password is something else for `my_repo`, I can create this file:

```sh
# ~/Developer/db_snapshots/my_repo/config.env
DB_PASSWORD="some_equally_secure_password"
```

## On `EXCLUDED TABLES`

There are some tables I don't want to restore. These might be different for each project, but most projects I work on have a `sessions` table that contains user sessions. Restoring this table means I probably have to log in again before going back to debugging. No fun.

# 💾 File 2: Making a Dump

Making a dump of the current database state is handled in this script:

```sh
# zsh/.zsh/scripts/ddump.sh
#!/bin/sh
set -eu

. "$(dirname "$0")/_drestore_utils.sh"

# Require filename argument
[ $# -ge 1 ] || {
  echo "Usage: $(basename "$0") <snapshot_name>"
  exit 1
}

snapshot_dir=$(get_snapshot_dir)
load_drestore_config "$snapshot_dir"

filename=$1
case "$filename" in
  *.dump) : ;;
  *) filename="$filename.dump" ;;
esac

filepath="$snapshot_dir/$filename"

[ -f "$filepath" ] && {
  printf "%s exists. Overwrite? (y/n): " "$filepath"
  read confirm
  case "$confirm" in
    [yY]) : ;;
    *) echo "Aborting."; exit 1 ;;
  esac
}

echo "Creating snapshot: $filepath"

PGPASSWORD="$DB_PASSWORD" pg_dump \
  -U "$DB_USER" \
  -h "$DB_HOST" \
  -d "$DB_NAME" \
  -F c -b -v -f "$filepath"

echo "✅ Snapshot saved: $filepath"
```

It just gets the snapshot directory from the previous script, and runs the `pg_dump` command.

The script has just one required parameter for the filename, since it needs to know what you want to call the dump.

For example, if I want to name it `'my-dump'` I can run this command:

```
~/.dotfiles/zsh/.zsh/scripts/ddump.sh my-dump
```

This will create a file `~/Developer/db_snapshots/my_repo/my-dump.dump`

It prompts before overwriting existing files so I don't nuke a snapshot by accident.

Notice how it automatically adds `.dump` as the extension. You can add this yourself when running the script, and the script is smart enough to omit it if it's already provided (but why would you want to suffer like that?)

# 🧞 File 3: Restoring a Dump

This is the one I use most. While `ddump.sh` captures a snapshot of the current database state, `drestore.sh` lets me teleport back to that exact state whenever I want. No clicks. No forms. No migration rollbacks. Just *boom*, back in time.

```sh
# zsh/.zsh/scripts/drestore.sh
#!/bin/bash
set -euo pipefail

source "$(dirname "${BASH_SOURCE[0]}")/_drestore_utils.sh"

snapshot_dir=$(get_snapshot_dir)
load_drestore_config "$snapshot_dir"

filename="${1:-$(find "$snapshot_dir" -type f -name "*.dump" | xargs -n 1 basename | sed 's/\.dump$//' | fzf --prompt="Select snapshot: ")}" || exit 1
[[ "$filename" == *.dump ]] || filename="$filename.dump"
filepath="$snapshot_dir/$filename"

[ -f "$filepath" ] || { echo "Snapshot file $filepath does not exist. Aborting."; exit 1; }

PGPASSWORD="$DB_PASSWORD" psql -U "$DB_USER" -h "$DB_HOST" -d "$DB_NAME" -c "
DO \$\$
DECLARE
    r RECORD;
BEGIN
FOR r IN (SELECT tablename FROM pg_tables WHERE schemaname = '$DB_SCHEMA' AND tablename NOT IN ($(excluded_clause))) LOOP
        EXECUTE 'DROP TABLE IF EXISTS ' || quote_ident(r.tablename) || ' CASCADE';
    END LOOP;
END \$\$;"

set +e
PGPASSWORD="$DB_PASSWORD" pg_restore -U "$DB_USER" -h "$DB_HOST" -d "$DB_NAME" -v "$filepath"
restore_status=$?
set -e

if [ "$restore_status" -ne 0 ]; then
  echo "⚠️  pg_restore completed with warnings (exit code $restore_status)"
fi

echo "✅ Snapshot restored: $filename"
```

Let's break it down:

- If you pass a snapshot name as an argument, it restores that one.
- If you don't, it uses `fzf` to let you pick one interactively.
- It drops all tables **except** the ones listed in `EXCLUDED_TABLES` (what's the point of saving time reverting state if you have to fill in your password again?)
- Then it runs `pg_restore` to restore your snapshot.

> Simple. Reliable. Fast.

# ❤️ Why I Love This Script

You ever get to step 25 of a bug reproduction and realize your test data is slightly off? This script is your get-out-of-debugging-jail-free card. With it, you can:

- Reset the DB state in ~4 seconds
- Rerun the same steps instantly after a code change
- Get back to the exact same place with zero fuss

An added bonus is that my database (and by extension my UI) doesn't become a graveyard filled with the corpses of previous attempts to fix the bug.

# 🚀 Supercharging The Script: Enter entr

One of my favourite CLI tools is `entr`. It lets me run a command on every file change. So all I have to do is run this command:

```
ls **/**.go | entr ~/.dotfiles/zsh/.zsh/scripts/drestore.sh my-feature
```

And now, every time I change a `.go` file, my database instantly resets and I can go back to the browser to see the results of my code change.

# ✨ Final Thoughts

Is this script fancy? No.

Is it polished? Absolutely not.

Does it save me hours of frustration every single week? You bet.

Is it fun watching teammates squint when they see a primary key of a row I just created is `1`? Damn right it is.

If your feedback loop feels slow, brittle, or bloated with steps that don't move the needle, give something like this a shot. Automate the boring parts. Skip the setup. Get to the bug faster.

> Your future self (and your keyboard) will thank you.
