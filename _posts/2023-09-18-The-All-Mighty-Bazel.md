---
layout: post
title: "The All-Mighty Bazel"
date: 2023-09-18 17:07:40 +0530
tags: [ bazel, build ]
categories: ["engineering"]
---

Bazel is a nifty tool that makes building and testing code extremely easy. One of the major reasons why Bazel is amazing
is the ease with which it can be extended for other languages and other CPU architectures, dynamically generate targets
etc. Basically, do what looks like magic to an external observer (And no, it will not fill the void left by you not
receiving your Hogwarts acceptance letter, I know I tried).

This extensibility or ability to do magic is achieved through the use of macros and rules.

`Macros`: In Bazel’s context a macro is a function that instantiates a rule. A rule can be instantiated by manually
writing a target in the BUILD file. However, when a rule needs to be instantiated multiple times it becomes tedious and
potentially makes the BUILD file complex or too repetitive.

`Rules`: A Rule defines the actual build steps to be executed on some input and returns a set of outputs. This output
set is generally an executable along with the necessary files required by the executable to run. Rules can be written to
compile code for new/custom languages or even different CPU architectures, generate and push container images, generate
configs etc.

In this article, we will be writing a bazel rule to take a file as input and encode the file in bas64 when run. But
before we dive into writing the rule I want to talk a little bit about what happens when a Bazel build command is run.

## Bazel Build

When the build command is run, the specified target and its dependencies get built. This build process consists of 3
main phases: -

`Loading Phase`: In this phase, all the build files are read and evaluated. Macros are executed and rules get
instantiated. Bazel builds a graph of the targets and maps their dependencies. Basically, this is the phase Bazel
figures out what all has to be built.

`Analysis Phase`: In this phase, Bazel traverses through the graph generated in the previous phase and executes the
implementation function of the instantiated rules. This in turn instantiates a set of actions (Build steps) to be
executed which again is stored in a graph. This new graph is called the “action graph”.

`Execution phase`: In this phase, bazel traverses the action graph and executes the “actions” and generates the output
which can be found in the bazel-out directory if the build is successful.

There are other phases like the fetching phase, caching phase etc which I will not be talking about in this article.

I recommend checking out [Jay Conrad’s blog][Jay-blog]. Jay’s blog has some pretty great articles on a lot of things
bazel being one of them.

## Writing The Rule

We will be writing an executable rule which when built will output an executable shell script. This shell script will
encode the file to base64 when run.

- This tutorial assumes the bazel workspace is already setup.
- I heavily borrow from the official documentation and [bazel examples][bazel-examples].

Let us start by creating a directory “base64_rule” to house our rule. Inside this directory create the following files:

`to_base64.bzl`: The file with the rule definition and implementation.

`BUILD.bazel`: A build file to tell bazel this is part of the workspace.

The rule needs to be importable so that it can be instantiated in other build files. We make the rule importable by
adding the following contents to the BUILD.bzl file.

```python
# This will let the rule be imported from anywhere within the workspace.
package(default_visibility = ["//visibility:public"])

exports_files(glob(["*.bzl"]))
```

Now to define the rule in the `to_base64.bzl` file.

```python
base64_encode = rule(
    implementation = _base64_encode_impl,
    attrs = {
        "file": attr.label(
            allow_single_file = True,
            mandatory = True,
        ),
    },
    executable = True,
)
```

This defines a rule called base64_encode with the following parameters:

- `implementation`: This parameter takes a function as input. This is the function which gets executed in the Analysis
  phase. The naming convention for this function is generally `_<rule_name>_impl` .
- `attrs`: This defines all the attributes the rule will accept when it is instantiated. All rules accept name and tags
  by default. Here, we have defined an attribute called file which is mandatory and takes a single file as input.
- `executable`: This tells bazel if the instantiated rule can be executed or not. This tells bazel if the file generated
  when the target is built is an executable or not. Here, we will be generating a shell script which will be executable
  hence it is set to true.

Now that the rule is defined, we need to define the implementation function.

```python
def _base64_encode_impl(ctx):
    out = ctx.actions.declare_file(ctx.label.name + ".sh")
    ctx.actions.write(
        output=out,
        content="base64 -i " + ctx.file.file.path,
        is_executable=True,
    )
    return [DefaultInfo(executable=out, runfiles=ctx.runfiles(files=[ctx.file.file]))]

```

In the function, we first declare an output file (the shell script). This will create shell script `<target_name>.sh`.

`ctx.actions.write` Adds an action to the action graph during the analysis phase. This action gets executed in the
execution phase. In this case, this action generates the shell script (or the out file).

The implementation function returns a list of providers. A Provider in its most simple form is a data structure in *
*Starlark**. Here we are returning DefaultInfo object. This provider gives general information about a target’s direct
and transitive files. Every rule type has this provider, even if it is not returned explicitly by the rule’s
implementation function.

Now that the rule is defined, we can instantiate the rule with the following content in any `BUILD.bzl` file in the
workspace (in this case we will be instantiating the rule at the root of the workspace).

```python
load("//bas64_rule:to_base64.bzl", "base64_encode")

base64_encode(
    name="testfile",
    file="testfile.txt",
)
```

Here, testfile.txt is the file that needs to be encoded in base64.

To build the shell script, run:

```bash
bazel build //:testfile
```

Output:

```bash
INFO: Analyzed target //:testfile (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
Target //:testfile up-to-date:
bazel-bin/testfile.sh
INFO: Elapsed time: 0.503s, Critical Path: 0.01s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
```

To get the base64 output, run:

```bash
bazel run //:testfile
```

Output:

```bash
INFO: Analyzed target //:testfile (1 packages loaded, 2 targets configured).
INFO: Found 1 target...
Target //:testfile up-to-date:
bazel-bin/testfile.sh
INFO: Elapsed time: 0.747s, Critical Path: 0.01s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
INFO: Running command line: bazel-bin/testfile.sh
YWhzZ2ZoZnNrc2RqZGtz
```

While encoding a file in base64 is trivial this is simply meant to show how powerful Bazel actually is.


[Jay-blog]: https://jayconrod.com/posts/106/writing-bazel-rules--simple-binary-rule

[bazel-examples]: https://github.com/bazelbuild/examples/tree/main/rules
