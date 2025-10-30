---
title: "My favorite aliases"
date: 2025-10-30
---

### 🚧 Under construction 🚧


My personal favorite aliases. Just paste in `~/.bashrc`

## Kill process on port

Useful when, for example, a previous server instance is hanging on a port that you want to reuse.

```bash
killport() {
  if [ -z "$1" ]; then
    echo "Error: Please specify a port number."
    echo "Usage: killport <PORT_NUMBER>"
  else
    echo "Attempting to kill process on port $1..."
    # Find the PID and kill the process
    lsof_output=$(lsof -t -i :$1 2>/dev/null)
    if [ -z "$lsof_output" ]; then
      echo "No process found running on port $1."
    else
      # Use xargs to safely pass the PIDs to kill
      echo "$lsof_output" | xargs kill -9
      echo "Process(es) on port $1 killed."
    fi
  fi
}
```

## Copy file contents to clipboard

Very useful when you want to copy, for example, keys from files, like ssh keys for github, GCP SA keys, etc.

```bash
clip() {
  if [ -z "$1" ]; then
    echo "Error: Please specify a file."
    echo "Usage: clip <filename>"
  else
    cat "$1" | xclip -selection clipboard
    echo "Contents of '$1' copied to clipboard."
  fi
}
```