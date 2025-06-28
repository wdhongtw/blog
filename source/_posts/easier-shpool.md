---
title: Make shpool Easier to Use
date: 2025-06-28 21:22:00
categories: [tips]
tags: [shell, terminal]
---

`shpool` is a tool like `tmux` and `screen` to keep persistent shell sessions across hangups.

It's quite new (first released 2023), and extremely simple to use.
Usually I only need one persistent shell in remote machine, and this tool, unlike `tmux`,
is designed to support **only one** shell per session.
That makes me very satisfied.

Currently there is no Bash auto-complete setup provided from the official repo.
Therefore I implemented one for myself. With that, I can make my life easier. 🤣

```sh
shpool <tab>  # complete sub-commands like attach / list / help ...
shpool attach <tab>  # complete available sessions
```

Here is the auto-complete setup script for Bash, for anyone who have same requirement.

```sh
# shellcheck shell=bash

_shpool_completions() {

  [[ ${#COMP_WORDS[@]} -eq 1 ]] && return 0
  [[ ${COMP_WORDS[0]} != "shpool" ]] && return 0

  if [[ ${#COMP_WORDS[@]} -eq 2 ]]; then
    local commands=(attach detach kill list version help)
    local partial="${COMP_WORDS[$COMP_CWORD]}"
    mapfile -t COMPREPLY < <(compgen -W "${commands[*]}" -- "${partial}")
    return 0
  fi

  [[ ${#COMP_WORDS[@]} -ne 3 ]] && return 0

  # Now, try completing session names for commands that take them
  local partial="${COMP_WORDS[$COMP_CWORD]}"
  local command="${COMP_WORDS[$COMP_CWORD-1]}"

  case "${command}" in
    attach|detach|kill)
      local -a sessions
      mapfile -t sessions < <(shpool list | tail -n +2 | awk '{print $1}')
      mapfile -t COMPREPLY < <(compgen -W "${sessions[*]}" -- "${partial}")
      return 0
      ;;
    *)
      # No need for other commands, they don't take arguments
      COMPREPLY=()
      return 0
      ;;
  esac
}

complete -F _shpool_completions shpool
```

Wish this tool gain more public notice!

- GitHub Link: <https://github.com/shell-pool/shpool>
