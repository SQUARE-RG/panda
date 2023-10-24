![](images/banner.png)

## From Compilation Database to Compiler-Based Tools.

*Panda* is a compilation-independent scheduler
for pipelining compiler-based tools in parallel
based on the [JSON compilation database][link-cdb].
It allows you to execute various tools
on translation units by replaying the compilation process.
An introduction video to this tool is available from <https://youtu.be/dLG2tEzuaCw>.

The advantage of *Panda* include:

1. Compatible to customize executions of various Compiler-Based Tools
2. Avoiding interference with the build system;
3. Scheduling tool execution in a dependency-free manner
    to take full advantages of the system resources.

## Installation

*Panda* is a standalone Python script.
You can install it by directly downloading file `panda` from this repo.

```
$ curl -fsSL https://github.com/Snape3058/panda/raw/demo/panda | sudo tee /usr/bin/panda >/dev/null
$ sudo chmod +x /usr/bin/panda
```

GitHub Repo for ICSE 2024 tool demo revision: <https://github.com/Snape3058/panda/tree/demo>.
Please note that the content on the `demo` branch is ahead of the main branch.
And the functionalities on this branch will be merged to the main branch
after this tool paper gets accepted.

## Usage

Scheduling the execution of compiler-based tools requires the JSON Compilation Database.
Users can setup the environment according to the introduction from Clang
(<https://clang.llvm.org/docs/HowToSetupToolingForLLVM.html>)
or using tools like [Bear (Build EAR)][link-bear].

Execution of *Panda* requires
the *CBT Execution Configurations* (Section 2.2) to be scheduled,
as well as optional settings,
such as number of parallel workers and output path.

```
$ panda <configurations> [-f CDB] [-j JOBS] [-o OUTPUT] [options]
```

*Panda* provides built-in configurations that cover most scenes
of executing analyzers and generating desired inputs for analyzers.
The built-in configurations can be categorized as
Compiler Tool (T<sub>Compiler</sub>) Configurations,
Frontend Tool (T<sub>Frontend</sub>) Configurations,
and Compilation Database Configurations.
The first two categories have been mentioned in the paper,
and the last category of configurations are used to
generate output directly from the compilation database.

* Example 1: Generating external function map and invocation list
    to path `/tmp/csa-ctu-scan` under a concurrency of 16 processes.

```
$ panda -YM -j 16 -o /tmp/csa-ctu-scan
```

* Example 2: Executing a customized plugin description `/tmp/check/plugin.json`
    and store output to path `/tmp/check` sequentially.

```
$ panda --plugin /tmp/check/plugin.json -o /tmp/check
```

### Built-in Compilation Database Configurations

The compilation database configurations
transform the input compilation database
to generate the output file,
or summarize the output of other T<sub>Frontend</sub> configurations.

* Generate *input file list* (`-L` or `--gen-input-file-list`):
    a list of all unique `file`s with absolute path.
* Generate *source file list* (`-F` or `--gen-source-file-list`):
    a list of all unique source files and the header files included.
* Generate *invocation list* (`-Y` or `--gen-invocation-list`)
    for Cross Translation Unit Analysis of the *Clang Static Analyzer*
    under [on-demand-parsing][link-odp] strategy.
* Generate *external function map* (`-M` or `--gen-extdef-mapping`)
    for Cross Translation Unit Analysis of the *Clang Static Analyzer*
    under [on-demand-parsing][link-odp] strategy.
* Generate *external function map* (`-P` or `--gen-extdef-mapping-ast`)
    for Cross Translation Unit Analysis of the *Clang Static Analyzer*
    under [AST-loading][link-al] strategy.

### Built-in Compiler Tool Configurations

The T<sub>Compiler</sub> Configurations
mainly generate inputs in desired formats for different analyzers.

* Test command line arguments and source file syntax (`-X` or `--syntax`):
    invoke compiler with `-fsyntax-only -Wall`
* Re-compile the source file (`-C` or `--compile`):
    invoke compiler with `-c`
* Generate preprocessed source file dump (`-E` or `--preprocess`):
    invoke compiler with `-E`
* Generate Clang PCH format AST dump (`-A` or `--gen-ast`):
    invoke the clang compiler with `-emit-ast`
* Generate LLVM Bitcode in binary format (`-B` or `--gen-bc`):
    invoke the clang compiler with `-emit-llvm`
* Generate LLVM Bitcode in text cormat (`-R` or `--gen-ll`):
    invoke the clang compiler with `-emit-llvm -S`
* Generate assembly dump (`-S` or `--gen-asm`):
    invoke compiler with `-S`
* Generate dependency description dump (`-D` or `--gen-dep`):
    invoke compiler with `-M`
* Execute Clang Static Analyzer without Cross Translation Unit Analysis (`--analysis`)

### Built-in Frontend Tool Configurations

The T<sub>Frontend</sub> configurations mainly invoke Clang Tooling based tools.

* Generating external function map (as mentioned above)

### Plugins

Users can execute customized T<sub>Compiler</sub> and T<sub>Frontend</sub> tools
with plugins defined with a CBT execution configuration in JSON format.
In the description,
field `comment` is a string for commenting the description,
field `type` determines the type of the configuration,
and object `action` defines the CBT Execution Configuration object.

* Example T<sub>Compiler</sub> configuration (Figure 4a)
  of generating dependency files (option `-D` or `--gen-dep`).

```json
{
    "comment": "Example plugin for Panda driver.",
    "type": "Compiler",
    "action": {
        "prompt": "Generating dependency file",
        "tool": {
            "c": "clang",
            "c++": "clang++"
        },
        "args": ["-fsyntax-only", "-w", "-M"],
        "extension": ".d",
        "outopt": "-MF"
    }
}
```

For a T<sub>Compiler</sub> configuration, object `action` has four fields.
Field `prompt` defines the prompt string printed during executing the tool.
Field `args` is a list of command line arguments to be added during execution.
Field `extension` determines the extension name of the output file.
And field `outopt` represents the option of generating the output.

* Example T<sub>Frontend</sub> configuration (Figure 4b) of executing Clang Tidy
    with a configuration file `config.txt` in output directory
    and storing command line output of stderr stream to output file.

```json
{
    "comment": "Example plugin for Panda driver",
    "type": "Frontend",
    "action": {
        "prompt": "Generating raw external function map",
        "tool": "clang-tidy",
        "args": ["--config-file=/path/to/output/config.txt"],
        "extension": ".clang-tidy",
        "source": "stderr"
    }
}
```

For a T<sub>Frontend</sub> configuration, object `action` has five fields.
Field `prompt`, `args`, and `extension` have the same meaning as
a T<sub>Compiler</sub> configuration.
Field `tool` determines the tool to be executed.
And field `source` represents
the output of which stream will be stored to the output file.
Please note that, string `/path/to/output` will be always be replaced to
the actual output path determined with option `-o` during execution.

## Data Presentation and Open-Access

The Gantt Chart in Figure 6 can be generated with the `analyze-log` script.
And all data in the experiments are available from the Google Spreadsheet below.

### Print Execution Summary and Draw Gantt Chart

For Tool Demo paper revision,
execution logs are dumped to the output path in the format of

```
/path/to/output/logs-<strategy>-<key>-<timestamp>
```

where `<strategy>` refers to the sorting strategies mentioned in Section 2.4
that `fifo` for *First-Come-First-Service*, and `ljf` for *Longest-Processing-Time-First*.
And `<key>` represent the key of sorting the worklist.
As mentioned in Section 2.3, the number of semicolons (`semicolon`) is used by default,
whereas the number of code lines (`loc`) is also available for alternative.

To summarize a previous execution and present the Gantt Chart of all workers,
please use the `analyze-log` script provided only in this branch.

```
$ analyze-log /path/to/output/logs-<strategy>-<key>-<timestamp>
```

The `analyze-log` requires [Matplotlib][link-matplotlib] to generate the Gantt Chart.
If the Python interpreter fails to import this module,
the `analyze-log` script will **NOT** report an error and exit.

An example Gantt Chart can be found from Figure 6 in the paper.

### Selection of Key to Sort the Worklist

We select the key to sort the worklist with Pearson Correlation Coefficient.
The detailed data of calculating the data is presented in the Google Spreadsheet below.

<iframe src="https://docs.google.com/spreadsheets/d/e/2PACX-1vSf--XAfkfdPwY3p5K6QCjv-_yKoKUaV4tQcu9AiBvuOebHcZ8vuVsrGLuWseS4xQWZy3krDmX3PTlz/pubhtml?widget=true&amp;headers=false" width="800" height="600"></iframe>

It may take a while to load the data.
Please follow the above link or go to the [homepage][link-homepage] of *Panda* if the preview is not available.

### Detailed Data of Evaluation

The detailed data of the evaluation in Section 3 is presented in the Google Spreadsheet below.

<iframe src="https://docs.google.com/spreadsheets/d/e/2PACX-1vSseVVN-KKsLK3f6aHm2KWOZnEJkJ4s-S5rniYDk5lOPcZaDQBEqMCxwIv7T_NK2j_0AbuF4qRinPpw/pubhtml?widget=true&amp;headers=false" width="800" height="600"></iframe>

It may take a while to load the data.
Please follow the above link or go to the [homepage][link-homepage] of *Panda* if the preview is not available.

## Acknowledgments

* REST team, Institute of Software, Chinese Academy of Sciences
* The tool name, *Panda*, is inspired by the animated sitcom *We Bare Bears*
    and the compiler argument recorder [Bear (Build EAR)][link-bear].

Let me know if *Panda* helps you. Thanks.


[link-bear]: https://github.com/rizsotto/Bear
[link-cdb]: https://clang.llvm.org/docs/JSONCompilationDatabase.html
[link-al]: https://clang.llvm.org/docs/analyzer/user-docs/CrossTranslationUnit.html#manual-ctu-analysis
[link-odp]: https://clang.llvm.org/docs/analyzer/user-docs/CrossTranslationUnit.html#id2
[link-matplotlib]: https://matplotlib.org
[link-homepage]: https://lcs.ios.ac.cn/~maxt/Panda/#selection-of-key-to-sort-the-worklist
