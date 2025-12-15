Title: Justified

URL Source: https://www.bitecode.dev/p/justified

Published Time: 2025-12-14T17:24:47+00:00

Markdown Content:
_It’s nice to have shortcuts for commands you use often. To have normalized names for installing dependencies, running the code, or starting the tests across all your projects._

_Task runners are good at this:_`gulp`_,_`doit`_,_`rake`_... It’s so popular that build systems are sometimes used only for this feature, like with_`make`_._

_I tried the cross-platform [“just” task runner](https://just.systems/man/en/introduction.html) for a while, and it won me over. It is now my daily driver._

_Put a_`.justfile`_at the root of your project, add a few commands, and voilà, normalized self-documenting project shortcuts:_

```
set dotenv-load
set shell := [”zsh”, “-c”]
set windows-shell := [”powershell.exe”, “-NoProfile”, “-NoLogo”, “-Command”]

build_docs:
    python -m mkdocs build -f docs\mkdocs.yml
    python -m http.serve docs

runserver port=”8080”:
    python manage.py runserver 127.0.0.1:{{port}}

test *args:
    pytest tests {{args}}

format +targets=”src test”:
    ruff format {{targets}}
    ruff check --fix {{targets}}

[windows]
clean_pyc:
    Get-ChildItem -Path . -Filter *.pyc -Recurse | Remove-Item -Force

[unix]
clean_pyc:
    find . -name “*.pyc” -exec rm -f {} \;
```

_You can now list all actions in your project:_

```
❯ just --list
Available recipes:
    build_docs
    clean_pyc
    format +targets=”src test”
    runserver port=”8080”
```

_And run any of them from the comfy_`just`_wrapper:_

```
❯ just format foo/my_file.py
ruff format foo/my_file.py
1 file left unchanged
ruff check --fix foo/my_file.py
All checks passed!
```

_No need to remember all the underlying commands._

It’s one of those long days, you have been switching between 3 projects already, you are tired, and the sun went down 30 minutes ago. Not saying there are vampires out there, but given the number of bugs you had to kill today, you would not be surprised.

Now you are switching to yet another web project. It’s not installed, so you definitely need to figure that out. Is it using `poetry`, or `pip`, or `uv` ?

Ah, it’s pre-COVID at a time when you worked for one of those corps that used Anaconda. But not using the usual `conda` command, the guy who set that up decided it should be using the latest shining toy of years ago, and is using `mamba`. How does that work already?

How do you call this bloody thing? With what parameters?

You made it work somehow, piecing up bits from the readme, a deprecated StackOverflow post, and 2 wrong trials from ChatGPT.

But now you have to start the damn stuff.

So first, figuring out what framework it uses, and then the magical incantation to run the dev server. You kinda remember it has to listen to port 7777, but is that a Flask thing that goes:

`flask run --reload --port 7777`
A FastAPI thing that needs:

`uvicorn main:app --reload --port 7777`
Or a Django thing that wants:

`python manage.py runserver 7777`
Success. Now let’s figure out how to run the unit tests...

Most projects need a command to set it up, another one to run it, and another one to run the tests. Then a collection of custom ones of other dedicated stuff. Task runners normalize that, you only have to figure out what task runner to use, after that, it can list the commands available on the project, and run them for you with a short name instead of the whole thing.

There are thousands of task runners out there. The most popular is probably still `make` (which I really dislike because it was made as a builder, not a task runner, and it shows). In the Python world, I have used and recommended [doit](https://pydoit.org/) (nice article [here](https://www.bitecode.dev/p/doit-the-goodest-python-task-runner)) and [poethepoet](https://poethepoet.natn.io/index.html).

But for a few projects now, I’ve been using the Rust-based [just](https://github.com/casey/just) command. And since [uv doesn’t seem to include the feature any time soon](https://github.com/astral-sh/uv/issues/5903#issuecomment-3570848646), this is likely what I will stick to for the near future.

It’s fast, easy to use, simple to install, and works very, very well.

Allow me to demonstrate.

One of the wonderful things about Rust programs is that they usually come with tons of installation options. You will have to work very hard to find a popular system on which you cannot install `just`.

It comes with [a ridiculous number of supported package formats](https://just.systems/man/en/packages.html). You can install it with [homebrew](https://formulae.brew.sh/formula/just), [apt](https://packages.debian.org/trixie/just), [dnf](https://src.fedoraproject.org/rpms/rust-just), [snap](https://snapcraft.io/just), [nix](https://github.com/NixOS/nixpkgs/blob/master/pkgs/by-name/ju/just/package.nix), and even [npm](https://www.npmjs.com/package/rust-just) or [anaconda](https://anaconda.org/channels/conda-forge/packages/just/overview).

It’s on [pypi](https://pypi.org/project/rust-just/), so you can install it with `pip` and of course, `uv`.

I usually do:

`uv tool install rust-just`
Because it works the same on Windows, Mac, and Linux, and takes care of the PATH for me.

But frankly, on systems on which I don’t have `uv`, I directly grab the exe [from here.](https://github.com/casey/just/releases)

Even on very locked-down Windows boxes from my clients, I haven’t yet been given a system on which I can’t unzip “just-1.45.0-x86_64-pc-windows-msvc.zip”, and run directly `just.exe`.

And being able to ssh, then `wget` and unzip `just-1.45.0-x86_64-unknown-linux-musl.zip` on any vps box anywhere and run the thing as-is is fantastic. No root needed.

`just` really wants to run on your machine.

In `just`, commands live in a file **at the root of your project,** named “`.justfile`”.

You put a recipe in there:

```
# that's a comment
hello:              # that the name of the recipe
    echo “hello”    # that's the command it runs
```

And now, in your terminal, **go to your project directory**, then run it:

```
❯ just hello
echo “hello”
hello
```

(If you are on Windows, this might fail. I’ll explain later in the section “Just making it work on Windows”)

You’ll notice that `just` prints the command then runs it. You can avoid the printing by prefixing the command with a `@`:

```
hello:
    @echo “hello”
```

Which gives you:

```
❯ just hello
hello
```

You don’t have to limit yourself to one command per receipt; you can put several:

```
hello:
    @echo “hello”
    @echo “world”
```

And you don’t have to put a `@` on each command, you can put it on the recipe’s name:

```
@hello:
    echo “hello”
    echo “world”
```

Running it gives you:

```
❯ just hello
hello
world
```

That’s the basics.

In all your projects, you can now have a just file that does stuff like:

```
# Build and serve the projects technical doc
build_docs:
    python -m mkdocs build -f docs\mkdocs.yml
    python -m http.serve docs
```

(The comment will be the recipe’s doc BTW)

Now you never have to remember what command you need to build and serve the doc for each project. You do:

```
❯ just --list
Available recipes:
    build_docs    # Build and serve the projects technical doc
    hello
```

And you know what’s available.

Just files come with a lot of configuration knobs that you can activate using the `set` keyword. The most important one is the one to set the shell.

By default, it will use the `sh` shell, even on Windows. This might work if you have Cygwin or Git installed, but it might fail miserably.

You can tell it what shell to use on Windows by adding this at the begining of your `just` file:

`set windows-shell := [”powershell.exe”, “-NoProfile”, “-NoLogo”, “-Command”]`
This tells `just` to use PowerShell instead of `sh`. In this particular case, I tell it not to load the user’s profile (`-NoProfile`), not print the startup message (`-NoLogo`) and execute the next string (`-Command`, this one is required for it to work, the others are optional.

You can use `set shell` to set it to another shell for unix, and `set windows-shell` just for Windows, so you get a different shell on each.

If you read this blog, you know I am not fond of DSLs. I usually think they are too limiting, badly supported, with poor tooling and limited debugging capabilities for the little convenience they bring.

For a few successful bash, CSS, HTML, and SQL, you have a thousand Frankenstein experiments that haunt your career.

But I’m happy to report I quite like `just’s` DSL. It strikes a nice balance between usability and power: simple stuff is super simple, complicated stuff is possible. But it stops right before becoming a full-featured script language. You can code a Python script and call that from `just` instead. Plus it has good VSCode support.

So what can you do with `just`?

Well, variables, for once, which you set with `:=`:

```
# Create local variable for this script.  
tmp_dir := “./tmp”

# Make it an environnement variable accessible to all programs s 
export PATH := “./local/bin”

# Load the env variable “DEBUG” from the environement and put it  
# in a “debug” local variable. If it doesn’t exist, use the 
# default value “1”.
# You could use this with the “export” syntax above to make it 
# accessible to all commands but by default it creates a local variable.
debug := env_var_or_default(’DEBUG’, “1”)

# You can use variables defined previously, in your commands. 
# This will print:
# tmp_dir=./tmp
# PATH=./local/bin
# debug=1
@status:
    echo tmp_dir={{tmp_dir}}
    echo PATH={{PATH}}
    echo debug={{debug}}
```

But also named parameters:

```
runserver port=”8080”:
    python manage.py runserver 127.0.0.1:{{port}}
```

You can call this one with

*   `just runserver`**:** runs `manage.py runserver 127.0.0.1:8080`

*   `just runserver 7777` : runs `manage.py runserver 127.0.0.1:7777`

*   `just runserver port=7777` : runs `manage.py runserver 127.0.0.1:7777`

And sure, there are variadic versions to pass infinite parameters and proxy them to the underlying commands.

```
# 1 or more This means that “just format” requires 
# at least one but possibly many positional parameters. 
# Here we them to “ruff format” and “ruff check”
# It has a the default values of “src” and “test” 
# but anything you type will be overriding that so you 
# could type “just format your_file.py”.
format +targets=”src test”:
    ruff format {{targets}}
    ruff check --fix {{targets}}

# 0, 1 or more. This means that “just test” accepts any
# number of arguments, and passes them to “pytest tests”. 
# If you pass nothing it will run “pytest tests”. 
# But you could do “just test -k ‘filter’ --pdb”
# and it will run  “pytest tests -k ‘filter’ --pdb”.
test *args:
    pytest tests {{args}}
```

Should you need to, you can set an environment variable for a single command:

```
shell $PYTHONSTARTUP=”~/pythonstartup.py”:
    ipython
```

So you don’t need to use `export` for everything. This works cross-shell.

You will likely have to deal with cross-compatibility problems if you define recipes that must run on different OS.

You can define recipes that only exist on certain platforms:

```
# This recipe exists only on windows. Here I use a powershell command.
[windows]
clean_pyc:
    Get-ChildItem -Path . -Filter *.pyc -Recurse | Remove-Item -Force

# This recipe exists only on mac and linux, which both have find.
[unix]
clean_pyc:
    find . -name “*.pyc” -exec rm -f {} \;
```

You will find many markers like this with `just`.

E.G., this will not change the directory to the project root before running the recipe, and ask “Run recipe clean_py” before starting the show:

```
[unix, confirm, no-cd] 
clean_pyc: 
    find . -name “*.pyc” -exec rm -f {} ;
```

You may want people to define private recipes or load recipes only in some environments. In this case, you can optionally import them from another file. Start your `justfile` with this:

```
import? ‘another_file.just’

# Optional, use:
# set allow-duplicate-recipes
# To allow the same command to be defined twice and the last one
# to override all the previous ones.
```

It will attempt to include all recipes from `another_file.just` if it exists. If it doesn’t, it does nothing.

You can also tell it to load env vars from a `.env` file, which will let you configure applications depending on the context:

`set dotenv-load`
There is a lot more you can do with `just`. The DSL has conditionals, built-in functions, and unstable opt-in features. But with just what you have here, you can normalize most of your project in a convenient, fast, readable, and portable way.

LLM are good at generating justfiles. Justfiles help them to know what actions they can run on a project. It’s good documentation for everyone.

I’ll leave you with the last trick. Define:

```
default:
    @just --list
```

So that somebody calling `just` without argument will run `just --list` instead.

Because yes, you can call `just` from inside `just`.