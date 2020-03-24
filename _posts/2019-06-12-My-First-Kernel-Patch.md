---
layout: post
title: My First Kernel Patch
tags: [Linux, Kernel]
---

# Summary
Recently, I submitted [my first kernel
patch](https://lore.kernel.org/netdev/20190608021925.4060095-1-hechaol@fb.com/).
I will note down the basic workflow of submitting code to kernel in this post.
[This folder](https://www.kernel.org/doc/Documentation/process/) contains more
detailed instructions on how to become a kernel developer. My note will only
contain minimal steps for a noob like me.

# Check out kernel code
Check out the code you want to modify from [kernel git
repo](https://git.kernel.org/). My change went into bpf-next so my first step
was:
```
$ git clone https://git.kernel.org/pub/scm/linux/kernel/git/bpf/bpf-next.git
```

# Coding Style
[This doc](https://www.kernel.org/doc/Documentation/process/coding-style.rst)
is a complete guide for coding style. Here I will only note down 2 things that
are different from my personal habit.

## Indentation
**Always use tab instead of spaces for indentation and tabs are 8 characters.** The
reasons are:
1. **Make things easier to read** -- Large indentations define blocks more
   clearly than smaller ones.
2. **Prevent you from nesting your functions/conditions too deep** --
   8-character indentations push the code farther to the right so that you’re
   warned when you have a deep block.

## Braces
Put the opening brace last on the line and put the closing brace first **except
for functions** where the opening brace at the beginning of the next line. That
is:
```c
int function(int x)
{
        if (condition) {
                action();
        }
}
```

And **do not unnecessarily use braces where a single statement will do**. That
is:
```c
if (condition)
        action();
```

and

```c
if (condition)
        do_this();
else
        do_that();
```

unless one branch has more than one statements:

```c
if (condition) {
        do_this();
        do_that();
} else {
        otherwise();
}
```

To make sure I always follow the style, I installed a vim plugin
[linuxsty.vim](https://github.com/vivien/vim-linux-coding-style) to enforce it.
I also have the following 2 lines in vim to make sure it's only enforced in
certain direcotry and can be enabled on demand.
```
let g:linuxsty_patterns = [ "/home/hechaol/git/linux" ]
nnoremap <silent> <leader>a :LinuxCodingStyle<cr>
```

# Commit
When running `git commit`, you will need `-s/--signoff` option, which will add
the following line at the end of the commit message:

```
Signed-off-by: Your Name <your-email-address>
```

For detailed explanation of sign-off, see [this answer on
StackOverflow](https://stackoverflow.com/questions/1962094/what-is-the-sign-off-feature-in-git-for).

## Format Patch
You may want to split a big commit into smaller ones. If you have more than one
commit, you will need a cover letter to explain what you’ve changed in these
commits. For me, I had 3 commits and I created a folder for the patches. 

```bash
$ makedir outgoing
$ git format-patch HEAD~3 --subject-prefix="PATCH bpf-next" --cover-letter -o outgoing/
```
`--cover-letter` option will generate an empty cover letter along with the
actual patches. You will have to edit it.
```bash
$ vim outgoing/0000-cover-letter.patch
```

# Check Patch
Optionally, you can check the patches using a script before sending them out.

```bash
$ scripts/checkpatch.pl outgoing/*
```

It gives you warnings when you have style issues in your code, commit message
or cover letter.

# Send Patch
Once you are confident about the patches, you can send them to kernel mailing
list using `git send` command. First add the following config to ~/.gitconfig.

```
[sendemail]
  envelopesender = <your-email-address>
```

Then send the email by:

```
$ git send-email outgoing/* --to <a-mailing-list> --cc <people-you-want-to-CC>
```

The mailing list is different for different components. For example, for my patch, I did

```
$ git send-email outgoing/* --to bpf@vger.kernel.org --cc netdev@vger.kernel.org --cc <other people>
```

# Update Patch
After the patch is sent out, you will receive feedback from people. After
addressing review comments, you need to format a new patch. This time, in
subject prefix, you will have to add “v2”. Also include what you’ve changed in
the cover letter. For example:

```
$ git format-patch HEAD~3 --subject-prefix="PATCH v2 bpf-next" -o outgoing/
$ vim outgoing/0000-cover-letter.patch
```

**Note**
* You don't have to add "v1" for your first version! (My mistake #1)

* **NO TOP-POSTING** (My mistake #2) when you reply to people’s comment. That
  is, add your reply at the bottom of the original message instead of top.

* **PLAIN TEXT ONLY** (My mistake #3) when you send/reply emails to the mailing
  list. Even though I had configured my email client (Apple Mail app) to use
  plain text by default, for some reason it still included HTML content in my
  reply, which got rejected by kernel mail server. After that I don’t trust GUI
  email client anymore. To make sure I don’t have any HTML/rich text in my
  email, I spent some time learning and setting up Mutt, a text-based email
  client, on my laptop. See [this
  post](https://hechao.li/2019/06/11/Set-up-OfflineIMAP_Mutt_on_Mac/) for
  details.
  
* You will have to include what you have changed for each version in the cover
  letter. (My mistake #4). That is, the cover letter of the first version
  should be something like:
  ```
  From: Your Name <your-email-address>
  Subject: [PATCH bpf-next 0/3] Bla Bla Bla
  ```
  Then the cover letter of the second version should be:
  ```
  From: Your Name <your-email-address>
  Subject: [PATCH v2 bpf-next 0/3] Bla Bla Bla
  ...
  
  v2: Fixed ... (review comments addressed)
  ```
  You may also look at old commit logs for examples.

* When a person replies to your patch with "Acked-by: Their Name
  \<their-email-address\>", it means the patch looks good to them. In your next
  version (in case you have to address other people's comments), you can add
  this line under "Signed-off-by: Your Name \<your-email-address\>".

  ```
  From: Your Name <your-email-address>
  Subject: [PATCH bpf-next 0/3] Bla Bla Bla
  
  ...
  
  Signed-off-by: Your Name <your-email-address>
  Acked-by: Their Name <their-email-address>
  ```

# Wait Patiently
Once all comments are addressed, you will have to wait for the maintainer to
apply the patch. And you will be notified once it’s done.
