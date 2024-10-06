+++
title = 'Shell Scripting - xargs'
summary = 'A quick look at how xargs can help us create and run commands'
tags = ["shell scripting", "xargs"]
date = 2024-10-06
showToc = true
draft = false
+++

## Introduction

`xargs` is a nifty command used to craft and run other commands. Let's take a brief look at how this tool works and why we may want to use it.

## What is xargs?

`xargs`, short for extended arguments, reads from standard input (stdin) and converts it to arguments that can be fed into a command. You might wonder, "Why would I ever want to do that?"

The simple answer is that some commands can only accept arguments and cannot read from stdin. With commands like `grep` or `base64`, you can feed input via a pipeline like so:
```
echo "foo bar biz baz" | grep "biz" 
```

However, commands like `ls` or `mv` only accept arguments. If you have a file containing several directories, you can't just pipe the contents into the aforementioned commands. We also obviously don't want to `ls` or `mv` each directory manually. 

Why not let `xargs` convert the input into arguments for your command?

## How do we use it?

Suppose you have a file containing a list of Google Cloud Storage bucket paths and you want to delete each of these files programmatically:
```
gs://my-bucket/my_dir/my_subdir_1/file.txt
gs://my-bucket/my_dir/my_subdir_1/file2.txt
gs://my-bucket/my_dir/my_subdir_2/file3.txt
...
```

You can run the following command:

```
cat my_file_paths.txt | xargs -I % sh -c "gcloud storage rm %"
```

What we did was first print out the contents of the file to standard output (stdout). Using the `|` symbol, we constructed a pipeline which connects the stdout of the first command, `cat`, to the stdin of the second command, `xargs`.

By default, `xargs` takes each line of input and puts it at the end of the command provided to it and runs it. In our case, our input is a GCS bucket path and the commands that xargs runs are:
```
gcloud storage rm gs://my-bucket/my_dir/my_subdir_1/file.txt
gcloud storage rm gs://my-bucket/my_dir/my_subdir_1/file2.txt
...
```

The command line option, "-I", allows you to specify a replacement string character or token. This allows you to substitute the input in a specific position in the command that you are creating and running. 
```
cat my_first_arg.txt | xargs -I {} sh -c "my_command --arg_1 {} --arg_2 foo --arg_3 bar"
```

Otherwise, as previously mentioned, the input gets appended to the end of the command.

## Command line options

This isn't an exhaustive list of all possible options. Refer to the manual page for xargs to learn more.

* -I replace-str
    * This option allows you to specify a replacement string so that you can substitute the input in a specific location in the command that you're building
    * If you don't set the replacement string, then xargs appends the input to the end of the command
    * Example: `cat my_file | xargs -I {} echo "some-tool --arg_1 {} --arg_2 foo -- arg_3 bar`
* -L max-lines
    * Use up to N lines from stdin when constructing the command
    * Defaults to 1
* -P max-processes
    * If you want to run multiple commands in parallel, then specify a number greater than 1
    * Defaults to 1
    * If set to 0, xargs will try to run as many processes as possible (depending on CPU cores/threads available)

## Other noteworthy tips

xargs is not the right tool to substitute multiple arguments into a command in specific positions. In the examples I shared above, I showed how we can position an entire line in a specific spot in the command using the `-I` option.

If your file contains multiple arguments per line, you'll want to prepare your command ahead of time using other tools like `sed` (stream editor), `awk`, or even `xargs` and then save it to a file. Once the commands are saved to a file, you can inspect it for correctness and then pipe it into `xargs` to run.

## Conclusion

`xargs` is useful for automating bulk operations using commands that only accept arguments as input. Next time you want to repeat an action multiple times using slightly different arguments - but you can't pipe the arguments into the command - consider using `xargs`.
