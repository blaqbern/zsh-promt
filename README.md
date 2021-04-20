# Lunch and Learn 20 April 2021

In this talk, we'll be creating a zsh prompt with some useful info from scratch. There are likely better ways to do this, and there are great tools out there like [oh-my-zsh](https://ohmyz.sh/) for this purpose that were written by people who actually know what they're doing in the shell. There's even a zsh builtin called [vcs_info](https://github.com/zsh-users/zsh/tree/master/Functions/VCS_Info), that captures useful version control info for you that you can then add to your prompt. This is much more feature-rich and battle-hardened than what we'll build today. But my goal for this talk was just to share my experience tackling this exercise as a way to improve my shell scripting. I still consider myself very novice, but I've learned a ton and it's been valuable in the sense that it's forced me to solve problems in different ways than I would in languages that I'm accostomed to working in like Javascript. It's also been pretty fun.

So, without further ado...

### Setting the prompt

We'll start by creating a file called prompt where we'll put all the code needed to build our prompt. Ideally, this file would be sourced in your `.zshrc`, but while we're building it, we'll just source it directly from the terminal whenever we make changes, so we can see their effect right away.

```zsh
# ./prompt

PROMPT="%~ ⏾ "
```

The `PROMPT` environment variable is used by zsh to set the prompt string. This value needs to be a string, and if you try to set it to something else--like an array, for example--zsh will complain and the prompt will be blank. We'll be creating an array of components to put into our prompt, so we'll need to do some manipulation before we set it as our `PROMPT` value. We'll talk more about that in a bit.

### Getting Git info

I like to keep my left-side prompt pretty small so that my commands don't end up getting pushed half-way across the screen. So aside from a couple tweaks we'll make, this will be it for the left-side prompt. So, it's on to the right-side prompt, which we set using the `RPROMPT` env var in zsh. This prompt will be where we add a bunch of useful info related to Git. We'll be using a combination of git commands and files and directories in the `.git` folder to find the information we're after. Let's start by adding the current branch name. We can get this by looking at the contents of the `.git/HEAD` file:

```sh
# terminal
% cat .git/HEAD
ref: refs/heads/main
```

So, this gives us the current branch name, but we'll need to strip off all the other stuff in there. Thankfully zsh has something called modifiers that can help out a lot with this. We're going to use the tail modifier specifically, which will grab everything after the last `/`, which is exactly what we need in this case. The syntax for modifiers is a `:` followed by a letter, in this case `t` for tail. In order to use modifiers, we need to wrap the string in `${}`. This is the string interpolation syntax in the shell and it will allow us to do some useful manipulations on strings. We'll see some other examples as we move forward.

```sh
# terminal
% head=$(cat .git/HEAD)
% echo ${head:t}
main
```

Here we're declaring the `head` variable and setting it to the output of `cat .git/HEAD`. You'll notice the `$()` syntax here. This basically means "the output of the command or function inside the parentheses". We're going to be using this a lot. Next, we're printing out just the filename using the tail modifier (`:t`).

There's one other thing to note here, which is that if we tried to `cat` out a file that didn't exist, we'd get an error. We're setting the output of this command to a variable and we don't want that variable to be set to an error message, so we need to send any error messages to the waste bin. We do that by routing the stderr to the `/dev/null` directory: `2>/dev/null`. The `2` here represents the stderr file descriptor. (You can read more about file descriptors [here](https://thoughtbot.com/blog/input-output-redirection-in-the-shell))

So, back in our script, we can write the following:

```zsh
# ./prompt
current_branch=${"$(cat .git/HEAD)":t}

PROMPT="%~ ⏾ "
RPROMPT=$current_branch
```

Because the contents of the `.git/HEAD` file contains a space, we need to wrap the output in `"`, so that the tail modifier acts on the whole string and not just the part after the space.

Great! that worked! Or did it? If we change directories to a different repo, the branch displayed on the right-hand prompt will stay the same instead of showing the current branch in that repo. The reason is because we need to tell zsh to run these commands every time before displaying the prompt. We do this by wrapping the commands in a function and then calling that function inside a special function called `precmd`.

```zsh
# ./prompt

precmd () {
  build_prompt
}

build_prompt () {
  local current_branch=${"$(cat .git/HEAD)":t}

  PROMPT="%~ ⏾ "
  RPROMPT=$current_branch
}
```

> Note that since we're now inside a function context, we should declare any variables used only in the function as `local` variables.

### Adding color

What would a custom zsh prompt be without some fancy colors?! Let's add some.

Zsh has a way of changing the text color that's fairly straight-forward: you simply add `%F` followed by the name of the color you want (standard terminal colors will work) inside `{}`. To reset the color to the default value, we use `%f`. But this gets pretty tedious to type after a while, and it makes the code way less readable. So instead, we're going to make some helper functions that will be easier to use and easier to read:

```zsh
cyan () {
  echo %F{cyan}$1%f
}

PROMPT="$(cyan %~) ⏾ "
```

`$1` means the first argument of the function, so we simply set the color we want, print the passed-in string, and reset the color to the default.

Having a bunch of very similar color function clogging up our script is not ideal, and there's a way to clean this up. We can create a directory for functions and then add that directory to the list of directories where zsh looks for function to make available to our scripts. This list lives in the `fpath` env var. Suppose we want to add the `./zshfn` directory to the `fpath`. We can do that like this:

```zsh
fpath=( $0:a:h/zshfn ${fpath[@]} )
```

There's a lot to unpack here. Let's start with the shell's array syntax. We can set a variable to an array using the `arr=( space separated values )` syntax. So here, we're setting `fpath` to an array with two values (it's actually more than two, but we'll get to that). Let's start with the first value. This is the directory we want to add: `$0` is an env var is zsh (and bash as well, I think) that represents the path of the currently running file. We can use zsh modifiers to turn that into an absolute path (`:a`), and grab the path to the parent directory rather than the path to the file itself (`:h`). Next we append the name of our new directory (`./zshfn`). The second item in the array is an expansion of the existing `fpath` array. Here we see our second use for the `${}` string interpolation syntax. `fpath` is an array and we're accessing a special index `@`, which string interpolation will expand into every item in the array. You can think of this sort of like the spread operator in javascript. If we were going to write this is javascript, it would look something like this:
```js
fpath = [`${__dirname}/zshrc`, ...fpath]
```

Now we can stuff all of our color functions into that directory and we'll be able to access them in our script. The rule about this is that the name of the file will be the callable name of the function and the contents of the file will be body of the function. In other words, we don't need to declare the function in the file. The file will only contain the body of the function.

```zsh
# ./zshfn/cyan
echo %F{cyan}$1%f
```
```zsh
# ./prompt
fpath=( $0:a:h/zshfn ${fpath[@]} )

PROMPT="$(cyan %~) ⏾ "
```

> This walkthrough is a WIP. More to come...

