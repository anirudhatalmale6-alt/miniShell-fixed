# miniShell — what was wrong and what changed

The starting point ran `test.sh` correctly and compiled clean under
`-Wall -Wextra`. It also diverged from `dash` on 36 of 88 everyday shell
snippets, and segfaulted on `> file`.

Nothing below was judged by eye. Every expectation in `tests/` was produced
by running a real shell — `dash`, or `bash` for the handful of cases where
dash's exit codes are its own — never by running miniShell and writing down
what it printed. Against the original binary the suite reports
**139 passed, 148 failed**; against this one, **287 passed, 0 failed**.

```
make          build
make check    287 cases against recorded dash/bash output
make difftest same cases, compared against dash/bash live
make asan     the whole suite under ASan + UBSan + leak detection
make fuzz     1200 malformed/adversarial inputs, nothing may crash or hang
```

---

## Crashes

### 1. `> file` dereferenced a null pointer

`new_simple()` left `words` as `NULL`, and `execute_simple()` read
`s->words[nassign]` before it looked at `nwords`. Any command that is only
a redirection — which is how a script truncates a file — went straight
through a null pointer:

```
$ echo '> /tmp/x' > t.sh && ./minishell t.sh
Segmentation fault
```

`words` is now always an allocated, NULL-terminated vector.

### 2. `${` and `${x:` read past the end of the word

`strchr(str, '\0')` returns a pointer to the terminator, not `NULL`. So

```c
} else if (strchr("-=?+#%", s[i])) {
```

matched at the end of the string, took `op = '\0'`, advanced past the
terminator and then read `s[i]` beyond the buffer. AddressSanitizer:
`heap-buffer-overflow READ of size 1`. Both tests now check `s[i]` first.

### 3. A recursive function walked off the C stack

`f() { f; }; f` overflowed the stack. On Linux that is a segfault; on a
kernel with small fixed per-thread stacks it is a fault with no message at
all. Function nesting is now capped at `MAX_FUNC_DEPTH` (512) with a real
error message. Lower the constant if the thread stacks are small.

---

## Silently wrong output

### 4. Command substitution was not implemented and did not say so

There was no `$( )` or backtick handling. Rather than failing, the text
came back out:

```
echo a$(echo b)c    ->  a$(echo b ) c        dash: abc
echo `echo hi`      ->  `echo hi`            dash: hi
X=$(false); echo $?  ->  127                 dash: 1
```

A wrong answer that looks like an answer is worse than an error, and
command substitution is in nearly every script — including the standard
`i=`expr $i + 1`` loop counter, which is why the `while` and recursion
cases failed too.

Now implemented properly: `$( )` and backticks both fork, capture stdout
through a pipe, strip trailing newlines, and set `$?`. The lexer copies
the construct into the word verbatim, tracking paren depth and quoting, so
`echo $(echo "a)b")` is one word. Nesting works.

Two details that are easy to get wrong and are covered by tests:

* `X=$(false)` — a command with no command name but with a command
  substitution exits with the status of that substitution, so `$?` is 1.
* Unquoted `$(...)` is field-split; `"$(...)"` is not.

### 5. Every variable was exported

Assignments went straight to `setenv()`, so `X=1` was visible to every
child process. A shell variable is only in the environment once it has
been exported. There is now a variable table with an `exported` flag,
mirrored into the environment on `export`. `X=1; sh -c 'echo $X'` prints
nothing, as it should.

### 6. `"$@"` joined instead of splitting

`f "x y" z` saw one argument. `"$@"` must produce one field per
parameter. The expander now tracks, per character, whether it came from
quoted text, literal text or an unquoted expansion, so field splitting and
globbing apply to exactly the right characters — and `"$@"` forces a field
boundary between parameters. The POSIX corner cases are covered:

```
X=" "; echo "$X"   -> one field, one space
X=" "; echo $X     -> no fields at all
echo ""            -> one empty field
set --; echo "$@"  -> no fields (not one empty one)
```

### 7. No globbing

`ls *.c` passed the literal `*.c` to `ls`. There is now a pattern matcher
(`*`, `?`, `[a-z]`, `[!abc]`, backslash escapes) and a path walker that
expands per segment, keeps a leading dot unmatched unless the pattern
starts with one, sorts the results, and leaves the word alone when nothing
matches. Quoted metacharacters do not glob: `echo "*"` prints `*`.

The same matcher backs `case` patterns and `${x#pat}` / `${x%pat}`.

---

## Things that killed the whole script

### 8. `2>&1` was a syntax error

The lexer never produced a `>&` token, so `parse_simple_command` reported
"missing redirection filename", `parse_list` returned NULL, and *nothing
in the file ran* — not even the lines before it. `>&` and `<&` are now
lexed and implemented, including `>&-` to close a descriptor.

### 9. `( ... )` swallowed its own closing paren

```
$ echo '(echo a; echo b)' | ./minishell
minishell: syntax error: expected ')' near ''
```

`parse_simple_command` accepted `)` as an ordinary argument once the
command already had a word, so the subshell's closing paren was eaten and
the parser then ran out of input looking for it. `)` is always an
operator now. `}` is still allowed as an argument, because it is only a
reserved word at the start of a command — which is what bash does too.

### 10. `[ ! -f x ]` returned 2

`builtin_test` had no `!` at all, so a negated test fell through to the
numeric comparison branch and reported a usage error. `-a` and `-o` were
missing too. `test` is now a small recursive-descent parser: `!`, `-a`,
`-o`, parentheses, `-r -w -x -s -L -p -c -b -nt -ot -t`, and a proper
"integer expression expected" message instead of a silent wrong answer.

`! cmd` as pipeline negation was also missing — the `!` went to `execvp`
and you got 127. It is now handled in `parse_pipeline`.

---

## Missing pieces a script actually needs

Added, each with tests: `$(( ))` arithmetic (full precedence including
`<< >> & ^ | && || == != < <= > >=`, division by zero reported rather than
trapped), `case`/`esac`, `until`, heredocs (`<<`, `<<-`, quoted delimiter
suppresses expansion), `~` expansion, and the builtins `:`, `read` (`-r`,
IFS splitting, last variable takes the rest), `shift`, `set` (`--`, `-e`,
`-x`, `-u`), `local`, `printf`, `eval`, `.`/`source`, `wait`, plus `$!`.

`${...}` grew the operators it was missing: `:-` `-` `:=` `=` `:+` `+`
`:?` `?` `#` `##` `%` `%%` and `${#var}`.

`set -e` is suppressed in the places POSIX requires — `if`/`while`/`until`
conditions, all but the last command of a `&&`/`||` list, and a `!`
pipeline — so `set -e; if false; then :; fi` does not exit.

### Redirections on compound commands

```
while read l; do echo "$l"; done < file
```

Only simple commands could take a redirection, so the read loop had
nothing to read from — this is what made the `while read` case hang. `if`,
`while`, `until`, `for`, `case`, `{ }` and `( )` now all accept trailing
redirections.

---

## The interactive prompt

`run_interactive` lexed and parsed one line at a time, so a multi-line
`if` was a syntax error the moment you pressed enter after `then`. There
was also no `-c`, so nothing could embed the shell.

Input now accumulates until it is complete — checked by lexing (does a
quote, `${`, `$(`, backtick or heredoc still stand open?) and by parsing
in a quiet mode that distinguishes "this is wrong" from "this is not
finished yet". `PS2` (default `> `) prompts for the continuation. Verified
over a real pty: multi-line `if`, `for`, function definitions, heredocs at
the prompt, multi-line command substitutions and unterminated quotes all
work.

`-c`, `-s`, `-e`, `-x`, `-u` and `--` are handled, and `$0` and the
positional parameters are set correctly in all three modes (`-c`, script
file, stdin).

---

## Memory

The original never freed the AST or the token array — 5212 bytes in 330
allocations on `test.sh` alone. On Linux process exit hides it; in an OS
where the shell is long-lived it just bleeds.

The catch is that freeing it naively is a use-after-free: `add_function`
stores the body as a borrowed pointer into the tree. So the rule is:

> a program text that installs a function is retained until exit;
> everything else is freed as soon as it has run.

The global function table owns only its own name and struct, never a body.
Under ASan with leak detection on, all 287 cases and 1200 fuzz inputs are
clean. The original reports leaks on all 287.

---

## Four places where the reference shells disagree

These are recorded from bash (with a `@note` in `tests/cases.txt`), because
dash is alone:

| | dash | bash | busybox | here |
|---|---|---|---|---|
| `echo $(exit 3)$?` | 0 | 3 | 0 | 3 |
| `cat < missing; echo $?` | 2 | 1 | 1 | 1 |
| `echo x > bad/f; echo $?` | 2 | 1 | 1 | 1 |
| `cd missing; echo $?` | 2 | 1 | 2 | 1 |

`set -u` on an unset variable exits 2, which is what dash and busybox do.

---

## Not implemented

`trap`, job control (`jobs`, `fg`, `bg`, `%1`), `getopts`, `select`,
process substitution, arrays, `${x/a/b}`, `[[ ]]`, `exec cmd` (as opposed
to `exec` with only redirections, which works), `$-`, `ulimit`, `umask`,
`times`, `hash`, `type`, `command`, `alias`. None of these are needed by
an ordinary script; say the word if any of them matter.

One known pathology, shared with dash: a pattern like
`/*/*/*/*/*/*/*/../../../../*` is exponential across the filesystem and
will sit there. dash hangs on it too.

---

## Porting to the OS

Roughly the same shape as miniPython. The C is C99 with no VLAs, no
function pointers, no `long long`, and no floats. What it does use that
the toy compiler does not have yet:

* `size_t` and `ssize_t` — typedefs, mechanical
* varargs, only through `snprintf` in `printf_escape` / `builtin_printf`
* `opendir`/`readdir` for globbing — needs the OS filesystem layer
* `fork`, `execvp`, `pipe`, `dup2`, `waitpid` — the real dependency

The last one is the decision, not the syntax. Everything else is a day of
mechanical work; process creation and pipes are a design question about
the kernel, and worth talking through before anyone starts typing.
