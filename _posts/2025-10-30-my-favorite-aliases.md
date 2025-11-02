---
title: "My favorite aliases"
date: 2025-10-30
---

### 🚧 Under construction 🚧


My personal favorite aliases. Just paste in `~/.bashrc`

### Kill process on port

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

### Copy file contents to clipboard

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

### History grep - function to quickly search command history

```bash
histgrep() {
  if [ -z "$1" ]; then
    echo "Usage: histgrep <search_term>"
  else
    history | grep "$1" | tail
  fi
}
```

Example

```
$ histgrep docker

1833  docker run -d -p 8080:8080 a9eb47a4ec21
1834  docker logs 29 -f
1835  docker ps
```

## Other useful tips

Not aliases, but very useful tips for linux shell

### Case insensitive tab completion

```bash
# enable case-insensitive tab completion for all users
echo 'set completion-ignore-case On' >> /etc/inputrc
```

> Source: [link](https://askubuntu.com/questions/87061/can-i-make-tab-auto-completion-case-insensitive-in-bash)

### Show Git bransh colored name

```bash
# Show git branch name
force_color_prompt=yes
color_prompt=yes
parse_git_branch() {
 git branch 2> /dev/null | sed -e '/^[^*]/d' -e 's/* \(.*\)/(\1)/'
}
if [ "$color_prompt" = yes ]; then
 PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[01;31m\]$(parse_git_branch)\[\033[00m\]\$ '
else
 PS1='${debian_chroot:+($debian_chroot)}\u@\h:\w$(parse_git_branch)\$ '
fi
unset color_prompt force_color_prompt

eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
```

> Source: [link](https://askubuntu.com/questions/730754/how-do-i-show-the-git-branch-with-colours-in-bash-prompt)