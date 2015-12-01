# gitbot

gitbot lets you programmatically make changes to many git repositories.

## Motivation

Clever has a service-oriented architecture where each service is its own git repository.
This lets us quickly develop changes that are limited to a single service.
However, there are some changes (see examples below) that need to be made across many services.
`gitbot` takes the pain out of making changes across many repos.

## Usage

`gitbot` takes in one argument: a path to a config file.
The config file is YAML of the following form:

```yaml
# repos is a list of repositories to examine, e.g. "git@github.com:Clever/gitbot.git"
# the format of each value here must be passable to `git clone`
repos:
  - git@github.com:Clever/aviator.git
# change_cmds describes the programs that will be invoked on each repo.
# a change command must conform to the following rules:
# - it takes in one positional argument: the path to a repo to examine
# - it either
#   (a) makes changes to files within the repo, outputs a commit message to stdout, and exits with code 0
#   (b) exits with a nonzero exit code

# Basepath to prepend temp directories. Not including this option is okay and the program will assume ""
base_path: "path/to/prepend/tmpdir"
change_cmds:
  # command paths can either be absolute paths, or paths relative to the configuration file.
  - path: "/path/to/the/program"
    args: ["-a", "flag"]
# post_cmds is a list of programs to run on each repo if changes have been made.
# use post_cmds to do things like pushing branches to github, opening PRs, etc.
# post_cmds are run within the directory where a repository has been cloned.
# post_cmds are only run if the change command makes a change.
# post_cmds can assume that the change has been committed to HEAD.
# post_cmds are run with the same environment variables as `gitbot` itself.
post_cmds:
  - path: "git"
    args: ["push", "origin", "HEAD:add-something-trivial"]
  - path: "hub"
    args: ["pull-request", "-m", "Added something trivial", "-b", "Clever:master", "-h", "Clever:add-something-trivial"]
```

## Tips

* Start small: run on a single repository to start.
* Start with a single no-op `post_cmd` and run `gitbot` with `GITBOT_LEAVE_TEMPDIRS=1`.
This lets you examine the side effects of the change command without any consequences.
* Use `git diff HEAD^ HEAD` as a `post_cmd` to see the commit that your change generated.

## Note about using `hub`

[hub](https://github.com/github/hub) is very useful as a `post_cmd`.
However, it requires some setup.
Specifically you will need to create a file `~/.config/hub` that contains an oauth token:

```
github.com:
- user: <your username>
  oauth_token: <provision one by visiting https://github.com/settings/applications>
  protocol: https
```

## Install

`gitbot` can be downloaded from the [releases](https://github.com/Clever/gitbot/releases) page.

## Example use cases

- update the version of a dependency to a new version
- run static analysis tools (e.g. linters)
- add a license/contributing.md to many repos
- optimize images
- programmatically change a common configuration file present in many repos

## Changing Dependencies

### New Packages

When adding a new package, you can simply use `make vendor` to update your imports.
This should bring in the new dependency that was previously undeclared.
The change should be reflected in [Godeps.json](Godeps/Godeps.json) as well as [vendor/](vendor/).

### Existing Packages

First ensure that you have your desired version of the package checked out in your `$GOPATH`.

When to change the version of an existing package, you will need to use the godep tool.
You must specify the package with the `update` command, if you use multiple subpackages of a repo you will need to specify all of them.
So if you use package github.com/Clever/foo/a and github.com/Clever/foo/b, you will need to specify both a and b, not just foo.

```
# depending on github.com/Clever/foo
godep update github.com/Clever/foo

# depending on github.com/Clever/foo/a and github.com/Clever/foo/b
godep update github.com/Clever/foo/a github.com/Clever/foo/b
```

