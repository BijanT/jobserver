# `jobserver`

This is a job server and client for running many experiments across many test
machines. In some sense, it is like a simple cluster manager.

The high-level idea is that you have a server (the `jobserver`) that runs on
your machine (some machine that is always on, like a desktop), perhaps in a
screen session or something. You have a bunch of experiments that you want to
run from some driver application or script. You also have a bunch of machines,
possibly of different types, where you want to run jobs. `jobserver` schedules
those jobs to run on those machines and copies the results back to the host
machine (the one running the server). One interracts with the server using the
stateless CLI client.

Additionally, `jobserver` supports the following:
- An awesome CLI client, with the ability to generate shell completion scripts.
- Machine classes: similar machines are added to the same class, and jobs are
  scheduled to run on any machine in the class.
- Machine setup: the `jobserver` can run a sequence of setup machines and
  automatically add machines to its class when finished.
- Job matrices: Run jobs with different combinations of parameters.
- Automatically copies results back to the host, as long as the experiment
  driver outputs a line to `stdout` with format `RESULTS: <OUTPUT FILENAME>`.
- Good server logging.
- Tries to be a bit resilient to failures, crashes, and mismatching
  server/client version by using protobufs to save server history and
  communicate with client"

# Building

Requires:
- `rust 1.37+`

```console
> cargo build
```

The debug build seems to be fast enough for ordinary use.

# Usage

Running the server:

```console
> cargo run --bin server -- /path/to/experiment/driver /path/to/log4rs/config.yaml
```

You may want to run this in a `screen` or `tmux` session or something. The
first time you run it, you will need to pass the `--allow_snap_fail` flag,
which allows overwriting server history. The default is to fail and exit if
loading history fails. It is intended to give a little bit of safety so that if
you restart in a weird configuration it won't wipe out your server's history,
which can be annoying.

Running the client:

```console
> cargo run --bin client -- <args>
```

There are a lot of commands. They are well-documented by the CLI usage message.

I recommend creating an alias for this client. I use the alias `j`, so that I
can just run commands like:

```console
> j job ls
```

# Server Logging

The server uses the [`log4rs`][l4rs] library for logging. It is highly configurable.

[l4rs]: https://crates.io/crates/log4rs

I use the following config:

```yaml
# Check this file for config changes every 30 seconds
refresh_rate: 30 seconds

# Write log records to stdout and to the logfile
appenders:
  stdout:
    kind: console
    # Format of the log entries
    encoder:
      pattern: "[{d} {h({l})}] {f}:{L} {m}{n}"

  logfile:
    kind: file
    path: "/nobackup/scratch/jobserver.log"
    encoder:
      pattern: "[{d} {h({l})}] {f}:{L} {m}{n}"
    # To keep logs compact, don't write debugging info
    filters:
      - compact:
        kind: threshold
        level: info

loggers:
  # All log entries from the `server` binary go to the stdout and logfile appenders
  server:
    level: debug
    appenders:
      - stdout
      - logfile
```

Save these contents to a file and point the server to it using the second
argument in the command.

# Examples

This is mostly intended as a quick tour of what you can do (for the client side).
It's not comprehensive. Read the usage message (`--help`) for more info on all
the things you can do. There are a lot of nifty features!

## Adding machines to the pool

First, let's list the machines in the pool.

```console
$ j machine ls


```

Currently, there are none. Let's add some. If we have machine already set up,
we can use `j machine add`, but we can also have a machine run a setup script
and be automatically added to the pool afterwards.

```console
$ j machine setup --class foo -m my.machine.foo.com:22 -- "setup-the-thing {MACHINE} --flag --blah" "another-setup-command"
Server response: Jiresp(
    JobIdResp {
        jid: 0,
    },
)
```

Here `{MACHINE}` is replaced automatically by `my.machine.foo.com:22`. You can
also use other variables (see the `j var` commands). This can help in a few ways:

- `{MACHINE}` allows you to use the same command for multiple machines (you can
  pass `-m` multiple times).
- You can use other variables to minimize the number of secrets that end up in
  your bash history (e.g. if you need a github token or something).

At this point, the machine `my.machine.foo.com:22` will start running the
listed commands in the given order. Assuming that they all succeed, the machine
will be added to the `foo` class in the pool and will be ready to run any jobs
that request a `foo`-class machine.

## Listing and Enqueuing Jobs

The `jid: 0` in the server response above is the job ID of the setup task. We
can see the currently running tasks, including setup tasks.

```console
$ j job ls
 Job   Status  Class  Command                                  Machine                Output
   0  Running  foo    setup-the-thing {MACHINE} --flag --blah  my.machine.foo.com:22
```

Currently, the only thing running so far is the setup task we started above.

We can queue up some other jobs for `foo`-class machines to run on it when ready:

```console
$ j job add foo "bar --the --foo baz" /path/to/results/dir -x 3
Server response: Jiresp(
    JobIdResp {
        jid: 1,
    },
)
Server response: Jiresp(
    JobIdResp {
        jid: 2,
    },
)
Server response: Jiresp(
    JobIdResp {
        jid: 3,
    },
)
```

Here we enqueue 3 identical tasks to run on the first available `foo` machine.
The jobs run in the enqueued order.

```console
$ j job ls
 Job   Status  Class  Command                                  Machine                Output
   0     Done  foo    setup-the-thing {MACHINE} --flag --blah  my.machine.foo.com:22
   1  Running  foo    bar --the --foo baz                      my.machine.foo.com:22
   2  Waiting  foo    bar --the --foo baz
   3  Waiting  foo    bar --the --foo baz
```

We can look at the job's stdout (using `tail`):

```console
j job log -t 1
```

where `1` is the job ID from the table above for the running task. You can also
look at the log of completed tasks or get the path of the log file:

```console
$ j job log -l 0                # look at the log of 0 with `less`
...

$ j job log 0                   # path to the log file
/some/path/0-setup-the-thing_my.machine.foo.com:22_--flag_--blah
```

## Job Matrices

Sometimes you want to run a bunch of similar commands with slight variations
(e.g. to see the effect of varying a parameter).

You can do this with `j job matrix`:

```sh
j job matrix add foo "my-experiment-cmd --param0 {I} --param1 {J} --param2 {K}" \
    /path/to/copy/results/ I=1,2,3,4 J=linear,quadratic,exotic K=banana,rockingchair,airplane
```

This command will enqueue 4x3x3 = 36 jobs, which you can see with `j job ls` or
`j job matrix stat`.

Moreover, you can dump a CSV of the matrix and any results paths using `j job matrix csv`.

## More

Take a look at the `--help` message for the various commands and subcommand to
learn about even more goodies.
