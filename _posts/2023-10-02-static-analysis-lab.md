---
layout: post
title: "Static Analysis Lab: C"
tags: code
---

# Prelude

This script was made for the 2023/2024 Software Specification course at Instituto Superior Técnico.

# Introduction

Before we dive into the lab exercises, let's briefly discuss some essential concepts.

## Background

**Nullptr Deference.** In C and C++ programming, a nullptr dereference occurs when a program attempts to access or manipulate a memory location using a null pointer (a pointer that doesn’t point to any valid memory address). This often leads to runtime errors, crashes, or unexpected behavior in your programs. Here’s an example:

```c
#include <stdlib.h>

int main() {
	int* ptr = NULL;
	*ptr = 42; // This is a null pointer dereference
	return 0;
}
```

In the above code, we have a pointer `ptr` initialized to `NULL`. The line `*ptr = 42;` attempts to assign a value to the memory location pointed to by `ptr`, but since `ptr` is null, this results in a null pointer dereference.

**Buffer Overflow.** A buffer overflow occurs when a program writes more data to a buffer (e.g., an array) than it can hold, leading to data corruption and potentially security vulnerabilities. Here's an example:

```c
#include <stdio.h>

int main() {
    char buffer[5];
    strcpy(buffer, "Hello, World!");  // Buffer overflow
    printf("%s\n", buffer);
    return 0;
}
```

In this code, the `strcpy` function writes more characters to the `buffer` than it can accommodate, resulting in a buffer overflow.

**Use-After-Free.** Use-after-free errors occur when a program attempts to access memory that has been deallocated or freed. These errors can lead to undefined behavior. For example:

```c
#include <stdlib.h>

int main() {
    int* ptr = (int*)malloc(sizeof(int));
    free(ptr);
    *ptr = 42;  // Use-after-free
    return 0;
}
```

In this code, the memory pointed to by `ptr` is freed before it is accessed, causing a use-after-free error.

**Memory Leak.** A memory leak happens when a program allocates memory but fails to deallocate it, causing memory to be unavailable for the rest of the program's execution. For example:

```c
#include <stdlib.h>

int main() {
    int* ptr = (int*)malloc(sizeof(int));
    // Memory is not freed
    return 0;
}
```

In this code, memory is allocated but not freed, resulting in a memory leak.

**Uninitialized Variables.** Uninitialized variables are variables that are used before being assigned a value. This can lead to undefined behavior and unexpected results. Infer can identify uninitialized variable issues. For example:

```c
#include <stdio.h>

int main() {
int x;
printf("%d\n", x); // Uninitialized variable
return 0;
}
```

In this code, the variable `x` is used in the `printf` statement before being initialized, resulting in an uninitialized variable issue.

**Static Analysis.** Static analysis is a program analysis technique that examines the source code without actually executing it. It aims to identify potential issues, bugs, and vulnerabilities in the code before runtime. Static analysis tools can quickly provide valuable insights into code quality and safety. In this lab we will delve into the realm of static analysis, leveraging two cutting-edge tools designed for analysing C code:

-  **clang-analyzer:** The [clang-analyzer](https://clang-analyzer.llvm.org/scan-build.html) is a part of the Clang compiler suite that provides static analysis capabilities. It is designed to detect various types of programming errors, including null-pointer dereferences, resource leaks, and more.
- **Infer:** [Infer](https://fbinfer.com/) is another powerful static analysis tool, similar to the *clang-analyzer*, that we will use in this lab to automatically detect programming errors.

Now that you have some context, let's proceed with setting up clang-analyzer and Infer and working on the lab exercises.

## Getting Started

To begin, please download and extract the file provided on the class page (or [here](https://filipeom.github.io/assets/zip/lab-static-analysis.zip)):

```bash
$ unzip lab-static-analysis.zip
```

It's important to note that both clang-analyzer and Infer are currently only available on Unix-like operating systems, such as Linux and macOS. Therefore, if you are using Windows, we highly recommend using [WSL 2 (Windows Subsystem for Linux)](https://learn.microsoft.com/en-us/windows/wsl/install). Below, we provide step-by-step instructions on how to install clang-analyzer and Infer. Notably, for this lab, using Infer's Docker image is highly recommended as it comes pre-installed with both Infer and clang-analyzer (instructions on Docker here: [Getting Infer: Docker Image](#docker)).

## Getting Clang-Analyzer

To install clang-analyzer, you can follow these steps based on your operating system below.

It's worth noting that if you opt for the recommended Infer Docker image, clang-analyzer comes pre-installed. You can find instructions for setting up the Infer Docker image here: [Getting Infer: Docker Image](#docker).

### macOS

Simply use Homebrew:

```bash
$ brew install llvm
```

### Ubuntu and Ubuntu-like Linux Distributions

On Ubuntu and similar Linux distributions, you can use the following command:

```bash
$ sudo apt-get install clang clang-tools perl
```

## Getting Infer: Binary Releases

On macOS, the simplest way to install Infer is by using Homebrew:

```bash
$ brew install infer
```

On Linux, or if you prefer not to use Homebrew, you can download the [latest release from GitHub](https://github.com/facebook/infer/releases/) and install it locally (recommended):

```bash
$ cd lab-sa/infer
$ curl -SLO "https://github.com/facebook/infer/releases/download/\
v1.1.0/infer-linux64-v1.1.0.tar.xz"
$ tar -xvf infer-linux64-v1.1.0.tar.xz && \
ln -s "$PWD/infer-linux64-v1.1.0/bin/infer" ~/.local/bin/infer
```

Alternatively, if you wish to install Infer system-wide (install at your own risk):

```bash
$ cd lab-sa/infer
$ curl -SLO "https://github.com/facebook/infer/releases/download/\
v1.1.0/infer-linux64-v1.1.0.tar.xz"
$ sudo tar -C /opt -xvf infer-linux64-v1.1.0.tar.xz && \
sudo ln -s /opt/infer-linux64-v1.1.0/bin/infer /usr/local/bin/infer
```

## Getting Infer: Docker Image {#docker}

You also have the option to use Infer's Docker image, conveniently included in the lab zipfile. However, if you plan to run Docker inside the lab computers, please be aware that you'll need to execute a few configuration commands before building Infer's Docker image. Comprehensive instructions for this can be found in our FAQ\footnote{\url{https://rnl.tecnico.ulisboa.pt/faq/\#docker}} (reproduced below for your convenience). If you are not utilizing lab computers, you may skip the subsequent paragraph (i.e., **Setup Docker**).

**Setup Docker.** The following command should only be executed *once*:

```bash
$ setup-docker
```

Then, add the following lines to your `~/.bashrc` file (changes take place after opening a new terminal).

```bash
export PATH=$HOME/bin:$PATH
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
```

Lastly, after every login users must run the following command to use docker:

```bash
$ systemctl --user start docker
```

**Build & Run Infer.** Ensure that docker is running as explained in the paragraph above and then execute:

```bash
$ cd lab-sa/infer/docker
$ docker build -t infer .
# mount the local lab directory inside the image
$ docker run --net=host -it -v $PWD/../../:/lab-sa infer /bin/bash
# you should now be inside the docker container with a shell prompt,
# e.g. "root@hostname:/#"
$ cd lab-sa/exercise01/
$ infer -- clang -c hello.c
```

# Exercise 01: Hello, World!

This exercise revolves around the program `hello.c`, which is provided alongside the project script in the `lab-sa/exercise01` directory.

### Infer

To automatically detect bugs in the program, utilize Infer by executing the following command:

```bash
$ cd lab-sa/exercise01
$ infer run -- clang -c hello.c
```

Once the Infer run is successful, you can delve into more comprehensive reports generated by Infer by running `infer explore` from the same directory.

### Clang-Analyzer

To automatically identify and analyze potential bugs in your program, you can utilize `scan-build`. Follow these steps to execute the analysis:

First, run the following command:

```bash
$ scan-build clang -c hello.c
```

Upon running the command, you should see an output similar to the following:

```bash
hello.c:5:8: warning: Dereference of null pointer (loaded from variable
	'ptr') [core.NullDereference]
  *ptr = 42;
   ~~~ ^
1 warning generated.
scan-build: Analysis run complete.
scan-build: 1 bug found.
scan-build: Run 'scan-view /tmp/scan-build-2023-10-01-094934-161859-1'
	to examine bug reports.
```

The output provides an initial summary of detected issues. To gain a more comprehensive understanding of these issues, you can examine the bug reports using `scan-view`. Execute the following command, replacing the path with the appropriate directory:

```bash
$ scan-view /tmp/scan-build-2023-10-01-094934-161859-1
# Which produces a webserver at 127.0.0.1:8181 to examine bug reports
Starting scan-view at: http://127.0.0.1:8181
  Use Ctrl-C to exit.
```

Running the command will start a web server at `http://127.0.0.1:8181` where you can examine the bug reports in a user-friendly interface. To access the reports, open a web browser and navigate to the provided URL.

# Exercise 02: Infer Workflow

In an Infer run, there are two key phases:

- **Capture Phase:** During this phase, Infer captures compilation commands to translate the files for analysis into Infer's internal intermediate language.
- **Analysis Phase:** In this phase, Infer conducts a separate analysis of each function and method. If Infer encounters an error while analyzing a method or function, it stops the analysis for that specific entity but continues to analyze others.

This phased analysis is especially useful when continuously analysing big projects with multiple files.

In this exercise, you'll work with two programs: `proj01.c` and `proj02.c`, which are provided in the `lab-sa/exercise02` directory. The exercise encompasses four main objectives:

1. Analyse the `proj01.c` program using Infer and clang-analyzer.
2. Manually identify all bugs in the program and report them in a YAML file.
3. Correct the bugs and use Infer or clang-analyzer to verify that there are no remaining warnings.
4. Repeat the entire process, including steps 1 to 3, for the `proj02.c` program.

To accomplish these goals, follow these steps:

## Infer
### Step 1: Run Infer

Capture the C project using:

```bash
$ infer capture -- clang -c proj01.c
```

Analyse the project with Infer:

```bash
$ infer analyze
```

Alternatively, to run both phases in a single command, use:

```bash
$ infer run -- clang -c proj01.c
```

### Step 2: Identify and Report Bugs

Examine the report generated by Infer more closely with:

```bash
$ infer explore
```

Create a YAML file (e.g., `bug_report.yml`) based on the `report.yml` template provided in the directory to report the identified bugs. For example:

```yaml
report:
  - bug:
      type: Nullptr Dereference
      lineno: 42
      class: tp

  - bug:
      type: Example Bug Type
      lineno: 67
      class: fn
```

In the YAML file, specify the bug type, line number, and class (refer to `report.yml` for available class options). This structured report will help in documenting and addressing the identified issues effectively (and will also be in the class project).

### Step 3: Bug Resolution

- Address all the identified bugs.
- Create a new version of the file (e.g., `proj01-fixed.c`), ensuring that all discovered bugs are fixed, and Infer no longer produces any warnings.

## Clang-analyzer

One benefit of the clang-analyzer is that is comes with many default analysis checkers, but also optin performance and bug checkers, for more information refer to `scan-build --help`. In this exercise we will play with different checkers to discover more bugs.

### Step 1: Initial Analysis

Begin by analyzing the project using the default checkers with the following command:

```bash
$ scan-build clang -c proj01.c
```

While reviewing the report, annotate the discovered bugs for further investigation.

### Step 2: Enabling Additional Checkers

Now, progressively enable the following checkers one at a time to uncover potential issues:

1. `optin.portability.UnixAPI`: Finds implementation-defined behavior in UNIX/Posix functions.

To enable this checker, execute the following command:

```bash
$ scan-build -enable-checker optin.portability.UnixAPI clang -c proj01.c
```

2. `security.insecureAPI.strcpy`: Warn on uses of the 'strcpy' and 'strcat' functions.

To enable this checker, use the following command:

```bash
$ scan-build -enable-checker security.insecureAPI.strcpy clang -c proj01.c
```

3. `security.insecureAPI.DeprecatedOrUnsafeBufferHandling`: Warn on uses of unsafe or deprecated buffer manipulating functions

To enable this checker, execute the following command:

```bash
$ scan-build \
-enable-checker security.insecureAPI.DeprecatedOrUnsafeBufferHandling \
clang -c proj01.c
```

### Step 3: Review and Evaluate Reported Bugs

After enabling each checker, carefully review the reported bugs. Determine whether these newly reported issues are true positives (actual problems) or false positives (incorrectly flagged). You can do this by examining the code and the specific checker warnings.

# Optional: Continuous Integration with Infer

Infer offers a significant advantage compared to other existing static analysis tools: it supports compositional analysis, allowing seamless integration into continuous integration (CI) pipelines. This integration provides developers with rapid feedback on their code. The goal of this exercise is to understand how Infer is currently applied in CI pipelines ([based on this tutorial](https://fbinfer.com/docs/steps-for-ci)).

### Exercise Setup:

1. Begin by accessing the provided `exercise03.zip` file in the `lab-sa/exercise03` directory.
2. In this exercise, we will utilise Infer to detect bugs in a feature branch of a toy project managed with Git. Follow these steps to run infer on two versions of the project and compare the results:

```bash
# Navigate to the exercise directory
$ cd lab-sa/exercise03

# Unzip the exercise files
$ unzip exercise03.zip

# Switch to the feature branch
$ git checkout feature

# Obtain a list of changed files between feature and master branches
$ git diff --name-only feature..master > index.txt

# Run Infer on the feature branch
$ infer capture -- clang -c main.c
$ infer analyze --changed-files-index index.txt

# Copy the Infer report for the feature branch
$ cp infer-out/report.json report-feature.json

# Switch to the master branch
$ git checkout master

# Use the 'reactive' option to retain previously-captured files
$ infer capture --reactive -- clang -c main.c
$ infer analyze --reactive --change-files-index index.txt

# Compare the reports between feature and master branches
$ infer reportdiff --report-current report-feature.json \
--report-previous infer-out/report.json
```

Upon running the last command, you will find three files in the `infer-out/differential` directory:

1. `introduces.json`: Contains issues found in the feature branch.
2. `fixed.json`: Contains issues found in the master branch but not in the feature branch.
3. `preexisting.json`: Contains issues found in both branches.

### Addressing Bugs:

Next, follow these step to address the bugs introduced in the new feature branch:

```bash
# Save the master branch's analysis report
$ cp infer-out/report.json report-master.json

# Return to the feature branch
$ git checkout feature

# Edit 'main.c' to fix the bug ...

# Re-analyse the project on the feature branch
$ infer capture --reactive -- clang -c main.c
$ infer analyze --reactive --changed-files-index index.txt

# Compare the reports
$ infer reportdiff --report-current infer-out/report.json \
--report-previous report-master.json
```

Ensure that Infer does not produce any warnings, and the `introduces.json` file is empty indicating successful bug resolution.
