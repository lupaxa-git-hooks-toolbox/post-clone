<p align="center">
    <a href="https://github.com/lupaxa-git-hooks-toolbox">
        <img src="https://raw.githubusercontent.com/the-lupaxa-project/brand-assets/master/logos/organisations/git-hooks-toolbox/readme-logo.png" alt="Organisation Logo" />
    </a>
</p>

<h1 align="center">Post Clone</h1>

Run a local action after `git clone` by wrapping the `git` command.

> [!CAUTION]
> **This is highly dangerous.** Implementing a post-clone action can run code you did not review, on your machine, every time you clone. Use it only when you fully control the repository and its contents. Proceed with caution.

## What it Does

Git has no post-clone hook. `git-wrapper` stands in front of `git` and, after a successful full `git clone`, runs a `post-clone` script from the root of the new tree. Every other git command is passed through, and so are bare and mirror clones.

Before that action, the wrapper reads `origin`. The host must be allowed, an allow rule must match, and no denied rule may match. By default it then asks before running the script.

## Install

Put the wrapper on `PATH`:

```bash
install -m 755 src/git-wrapper ~/bin/
```

Add this function to `.bashrc` or `.bash_profile`:

```bash
function git()
{
    command git-wrapper "$@"
}
```

Open a new shell so the function is defined.

## Quick Start

```bash
git clone git@github.com:lupaxa-git-hooks-toolbox/post-clone.git
```

The lists in `src/git-wrapper` ship empty, so this clone does not run `post-clone` until you add an allow rule. When a clone succeeds and the repository is allowlisted, the wrapper runs that file from the repository root.

## Repository script

Put a `post-clone` file at the root of each repository that should do something after clone. The wrapper executes that file with the repository root as its working directory.

Give it a shebang and mark it executable, in whatever language that repository uses. This repository does not ship that file.

The file name comes from `POST_CLONE_SCRIPT` at the top of `src/git-wrapper`. It defaults to `post-clone`. Change that variable when the repositories you clone use another name. The same name is used for every clone.

```bash
#!/usr/bin/env bash

echo "post-clone: ${PWD}"
```

Mark it executable with `chmod +x post-clone`. If the file is absent or not executable, the wrapper skips it and the clone still succeeds.

The wrapper does not run this file inside submodules. The parent script can do that after the submodules are checked out:

```bash
#!/usr/bin/env bash

git submodule update --init --recursive
git submodule foreach --recursive '
  if [[ -x ./post-clone ]]; then
    ./post-clone
  fi
'
```

Drop the `git submodule update` line when the clone already checked them out. If `POST_CLONE_SCRIPT` is not `post-clone`, call that file name instead.

## Default Behaviour

With the shell function in place, a full `git clone` will:

1. Run the real `git clone`
2. Use the directory given to `git clone`, or the repository name with one trailing `.git` removed
3. Read `origin` and require the host to be in `ALLOWED_HOSTS` and not in `DENIED_HOSTS`
4. Reject a denied organisation or repository, then allow an organisation or a repository
5. When the configured script exists and `CONFIRM_BEFORE_RUN` is `true`, ask before running it

If the script does not run, the clone command still succeeds. If the script runs and exits non-zero, the wrapper returns that status.

## What Triggers It

The wrapper skips Git's own options and reads the subcommand. The checks run only when that word is `clone`, the clone succeeds, and the result has a working tree.

Options in front of the subcommand do not change the decision, and they can be combined. `git --no-pager -c advice.detachedHead=false clone <url>` and `git --work-tree /path clone <url>` are still clones.

| Command                             | What happens    |
| ----------------------------------- | --------------- |
| `git clone <url>`                   | Runs the checks |
| `git --no-pager clone <url>`        | Runs the checks |
| `git -c <name>=<value> clone <url>` | Runs the checks |
| `git -C <path> clone <url>`         | Runs the checks |
| `git pull`                          | Passed through  |
| `git fetch`                         | Passed through  |
| `git --no-pager pull`               | Passed through  |
| `git -c <name>=<value> fetch`       | Passed through  |
| `git clone --bare <url>`            | Passed through  |
| `git clone --mirror <url>`          | Passed through  |

Any other subcommand, including `checkout`, `status`, and `push`, is passed through the same way. `--bare` and `--mirror` skip the checks because those clones have no working tree.

If `--no-bare` or `--no-mirror` comes later on the same command, the last of those flags wins. A working tree is then treated as a normal clone.

## Allowlist

`src/git-wrapper` allows HTTPS, `git@host:path`, `ssh://`, and `git://` origins. Denied lists are checked first and always win. A local path or `file://` URL has no host, so the script is skipped.

- `ALLOWED_HOSTS` lists hosts that may run the repository script. A plain name is exact, `*` matches every host, and any other entry is a regular expression used as written. The checked-in list is empty. Add `github.com` to allow that host.
- `DENIED_HOSTS` rejects a host even when an allow rule matches. The checked-in list is empty. With `ALLOWED_HOSTS` set to `*`, add `bitbucket.org` here to block that host.
- `CONFIRM_BEFORE_RUN` defaults to `true`. The wrapper asks `Run <script> in <directory>? [y/N]` and runs the script only for `y` or `yes`. Set it to `false` to skip the prompt. With confirmation on and no terminal, the script is not run.
- `DENIED_ORGS` rejects an organisation even when an allow rule matches. A plain name is exact. `^legacy-` rejects organisations with that prefix. The checked-in list is empty.
- `DENIED_REPOS` rejects an `org/repo` value, even inside an allowed organisation. A plain pair is exact. `^lupaxa-git-hooks-toolbox/legacy-` rejects names with that prefix.
- `ALLOWED_ORGS` approves every repository in an organisation unless that organisation or repository is denied. A plain name is exact. `^lupaxa-` approves every organisation with that prefix.
- `ALLOWED_REPOS` approves one `org/repo` value. A plain pair is exact. The checked-in list is empty. `lupaxa-git-hooks-toolbox/post-clone` approves that repository. `^lupaxa-git-hooks-toolbox/` approves that organisation.

Every list ships empty. Nothing runs until you add an allow rule.

Anything still unmatched prints `This repository is not allowed. Skipping.` and does not run the script.

A clone that used `-o` is checked under that remote name. `--no-origin` is checked from the URL on the command. The clone stays on disk, and the command still succeeds.

## Regex

Every list uses the same rules. A normal name is exact, so `github.com` does not match `notgithub.com`. `*` matches every value on that list.

Anything else is a regular expression, used exactly as written. The wrapper does not add `^` or `$`. Add `^` to match the start and `$` to match the end. `.` is one character, and `.*` is the rest of it.

| Entry                                 | Matches                                              |
| ------------------------------------- | ---------------------------------------------------- |
| `*`                                   | Every host, organisation, or repository on that list |
| `github.com`                          | Only the host `github.com`                           |
| `^lupaxa-`                            | Organisation names starting with `lupaxa-`           |
| `lupaxa-git-hooks-toolbox/post-clone` | Only that repository                                 |
| `^lupaxa-git-hooks-toolbox/`          | Every repository in `lupaxa-git-hooks-toolbox`       |
| `^lupaxa-git-hooks-toolbox/legacy-`   | Names in that organisation starting with `legacy-`   |
| `\.evil\.test$`                       | Hosts ending in `.evil.test`                         |

## Safety Notes

- The wrapper runs only in shells where the `git` function is defined.
- The repository script is code from the cloned repository. Add a repository to `ALLOWED_REPOS` only when you fully control that file.
- Add an organisation to `ALLOWED_ORGS` only when you trust every repository in it, then list exceptions in `DENIED_ORGS` or `DENIED_REPOS`.
- Add a host to `ALLOWED_HOSTS` only when you trust clones from that host, and put exceptions in `DENIED_HOSTS`.

<a href="https://github.com/the-lupaxa-project">
    <img src="https://raw.githubusercontent.com/the-lupaxa-project/brand-assets/master/logos/components/footer-for-child-orgs.svg" alt="The Lupaxa Project Footer" width="100%" />
</a>
