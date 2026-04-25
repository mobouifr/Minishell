*This project has been created as part of the 42 curriculum by mobouifr, mamir.*

<div align="center">

# minishell

### Where strings become processes.

*A Unix shell built from scratch — tokenizer, parser, expander, executor, and all the chaos in between.*

![Language](https://img.shields.io/badge/language-C-00599C?style=flat-square&logo=c&logoColor=white)
![Norm](https://img.shields.io/badge/norm-42%20style-informational?style=flat-square)
![readline](https://img.shields.io/badge/uses-GNU%20readline-green?style=flat-square)

</div>

---

## What is this?

`minishell` is a fully functional Unix shell written in C — not a wrapper, not a shortcut. Every subsystem is hand-built: the lexer tokenizes raw input character by character, the parser turns token streams into command structures, the expander handles variables and quotes, and the executor coordinates processes, pipes, file descriptors, and signals.

It reads a line. It runs a command. Simple idea. Deeply complex reality.

Building it means confronting the parts of Unix that higher-level languages quietly hide from you — `fork`, `execve`, `dup2`, `pipe`, `waitpid`, signal contexts, and file descriptor lifetime. When it works, it feels earned.

---

## What it actually does

> Every feature listed here was verified directly in the source code. Nothing is assumed.

| Feature | Status | Details |
|---|---|---|
| Interactive prompt | ✓ | `readline("minishell$ ")` loop with full history via `add_history` |
| Tokenizer | ✓ | State-machine lexer producing `WORD`, `\|`, `<`, `>`, `>>`, `<<` tokens |
| Quote handling | ✓ | Single and double quotes tokenized correctly as `WORD` segments |
| Syntax validation | ✓ | Catches unclosed quotes, invalid token positions, consecutive specials |
| Variable expansion | ✓ | `$VAR` and `$?` — undefined variables expand to empty string |
| Quote removal | ✓ | Quotes stripped during the expansion pass |
| Token merging | ✓ | Adjacent `WORD` tokens without whitespace are merged into one |
| Pipes | ✓ | Multi-stage pipelines via recursive pipe execution |
| Redirections | ✓ | `<` `>` `>>` `<<` — applied per-command in declaration order |
| Heredoc | ✓ | Reads with `>` prompt, stores in `/tmp/*`, expands `$VAR` in content |
| Builtins | ✓ | `echo`, `cd`, `pwd`, `export`, `unset`, `env`, `exit` |
| External commands | ✓ | PATH lookup and direct-path execution via `execve` |
| Signal handling | ✓ | Three distinct policies: prompt, child process, and heredoc contexts |
| Memory management | ✓ | Tracked allocator — `_malloc(..., 'm')` collects, `_malloc(0, 'f')` frees all |

---

## How it works

Every command you type travels through a pipeline of subsystems before anything runs:

```
readline("minishell$ ")
         │
         ▼
       LEXER
  raw input → tokens
  (WORD, |, <, >, >>, <<)
         │
         ▼
  SYNTAX CHECKER
  validates token stream
         │
         ▼
  EXPANDER + QUOTE REMOVAL
  $VAR → value, quotes stripped, tokens merged
         │
         ▼
       PARSER
  tokens → t_cmd + t_file linked lists
         │
         ▼
    HEREDOC RUNNER
  collects input, writes to /tmp/* files
         │
         ▼
  EXECUTION ENGINE
  builtin dispatch OR fork + execve
         │
         ▼
     EXIT STATUS
  stored in $?
```

### Pipes

When a command node has a `next`, execution enters pipe mode:

- A `pipe()` is created.
- Child 1 has its `stdout` redirected to `pipe[1]` and runs its command.
- Child 2 has its `stdin` redirected from `pipe[0]` and recursively executes the rest of the command list.
- The parent calls `waitpid` on both and captures the final status.

### Redirections

Each command carries a linked list of `t_file` descriptors built by the parser. Before execution, redirections are applied in declaration order using `open` + `dup2`. Types handled: `INRED`, `OUTRED`, `APPEND`, `HEREDOC`. Heredoc generates a temp file under `/tmp/`, which is then treated as a standard input redirection.

### Builtins vs external commands

Builtins are matched by string comparison and executed **in the current process** — stdin/stdout are backed up and restored around any redirections. Non-builtins go through `fork` + `execve`. PATH search kicks in unless the command string contains `/` or PATH is unset.

### Signals

The shell maintains three distinct signal contexts:

| Context | SIGINT | SIGQUIT |
|---|---|---|
| Prompt (idle) | Redraw prompt, set `$?` = 130 | Ignored |
| Child process | Default (terminates child) | Print `Quit (core dumped)`, set `$?` = 131 |
| Heredoc input | Exit heredoc child with 130 | Ignored |

---

## Project structure

```
minishell/
├── Makefile
├── readline.supp              # valgrind suppression for readline leaks
├── includes/
│   ├── minishell.h            # all shared types, enums, structs, prototypes
│   └── libft.h
├── src/
│   ├── main/
│   │   ├── main.c             # entry point
│   │   └── prompt_core.c      # REPL loop
│   ├── lexer/
│   │   ├── lexer.c            # tokenizer entry
│   │   ├── lexer_machine_states.c  # state machine transitions
│   │   └── lexer_utils.c
│   ├── syntax_error/
│   │   ├── syntax.c           # token stream validator
│   │   └── syntax_1.c
│   ├── expand/
│   │   ├── expand.c           # expansion driver
│   │   ├── calculation_1/2.c  # expansion helpers
│   │   ├── merge_export.c     # adjacent token merging
│   │   └── remove_quotes.c
│   ├── parsing/
│   │   ├── parser.c           # token list → command structures
│   │   ├── pipe_token_parsing.c
│   │   └── word_and_red_token_parsing.c
│   ├── heredoc/
│   │   ├── run_heredoc.c      # heredoc orchestration
│   │   ├── heredoc_expand.c
│   │   ├── heredoc_expand_content_copy/fill.c
│   │   └── heredoc_expand_var_utils.c
│   ├── execution/
│   │   ├── pipeline_execution.c   # pipe logic
│   │   ├── command_execution.c    # builtin dispatch + execve
│   │   ├── cmd_path_utils.c       # PATH resolution
│   │   └── execution_utils.c
│   └── builtins/
│       ├── echo/   cd/   env/   exit/   export/
│       └── unset.c
└── utils/
    ├── garbage.c              # tracked allocator + global cleanup
    ├── signal_handler.c       # signal policy by runtime context
    ├── exit_handler.c         # wait status → shell exit status
    ├── linked_list/           # list ops for tokens, env, cmds, files
    └── libft/                 # local libc-style helpers
```

---

## Getting started

### Requirements

- OS: Linux (POSIX APIs)
- Compiler: `cc` with `-Wall -Wextra -Werror`
- Library: GNU Readline (`-lreadline`)
- Optional: `valgrind` for leak/debug runs

### Build & run

```bash
git clone https://github.com/Cardano04class/minishell.git
cd minishell
make
./minishell
```

### Makefile rules

| Rule | Effect |
|---|---|
| `make` / `make all` | Compile `minishell` |
| `make clean` | Remove object files |
| `make fclean` | Remove object files and binary |
| `make re` | Full rebuild |
| `make sup` | Run with valgrind + readline suppression file |

### Examples

```bash
# Basic output
minishell$ echo hello world
hello world

# Environment variables
minishell$ export NAME=marvin
minishell$ echo $NAME
marvin

# Pipe
minishell$ echo one two three | wc -w
3

# Redirections chained
minishell$ cat < infile.txt | grep mini >> out.log

# Heredoc with variable expansion
minishell$ cat << EOF
> user=$NAME
> status=$?
> EOF
user=marvin
status=0
```

---

## Resources

Useful references for anyone building a shell from scratch:

- `man 2 fork` · `man 2 execve` · `man 2 pipe` · `man 2 dup2` · `man 2 waitpid` · `man 2 open` · `man 2 signal` · `man 3 readline`
- *The Linux Programming Interface* — Michael Kerrisk
- *Advanced Programming in the UNIX Environment* — Stevens & Rago
