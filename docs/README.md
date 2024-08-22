# ADR, TDR, and Tech Evaluations

This is an example of how one of the most functional teams I've been on organized the decision records and other important documents.

The point of the ADRs is to preserve the context around why the system is put together the way it is, as well as some details as to how.

The point of the TDRs is to record the team standards like the decision to have a Linter/Formatter in use, and whether it should or should not block your PR from merging, or the preferred way to do things.

And the Tech Eval docs are meant to be diaries that log and evaluate a technology we are considering, if we choose to use the tech, we _should_ update the doc to say we have accepted the tech, and then we ideally would add it to the TDR directory with links to each-other.

We used [this tool](https://github.com/npryce/adr-tools) to make the creation of these docs easy. It can use custom templates (like you'll see in the tech-eval directory) and you can specify the directory which should be used when you write the `adr new "*"` command with the `.adr_dir` files.

