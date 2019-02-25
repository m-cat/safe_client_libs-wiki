Want to contribute? Great! There are many ways to give back to the project, whether it be writing new code, fixing bugs, or just reporting errors. All forms of contributions are encouraged!

## Reporting bugs (Issues)

Notice something amiss? Have an idea for a new feature? Feel free to to write an Issue on the [GitHub repository](https://github.com/maidsafe/safe_client_libs/issues) about anything that you feel could be fixed or improved. Examples include:

- Unclear documentation
- Bugs, crashes
- Enhancement ideas
- ... and more

Please try to be as descriptive as possible. Provide as much information as you are able (e.g. Rust version, OS, Client Libs version). If you are seeing an error, paste as much of the log as you can. We don't currently have a template for Issues, but may add one in the future to help you structure your Issues and provide useful information.

Of course, after submitting an Issue you are free to tackle the problem yourself (see "Submitting patches" below), and encouraged to do so.

## Choosing an Issue

If you want to contribute code but don't know what to work on, we try to always keep some Issues marked `help-wanted` or `good-first-issue`. We also have the tag `easy` for categorizing Issues that are fairly low-effort, which you can tackle if you are short on time or new to programming. We want everyone to be able to contribute!

## Submitting patches (Pull Requests)

Before submitting code, please either choose an existing Issue, or write your own, describing the problem you are solving. We require every PR to have an accompanying Issue, so that we know concretely what the problem is and can track its resolution.

We follow the standard procedure for submitting Pull Requests (PR for short). Please refer to the [official GitHub documentation](https://help.github.com/articles/creating-a-pull-request/) if you are unfamiliar with the procedure. If you still need help, we are more than happy to guide you along!

### Format

The PR title and git commit messages should follow the format specified in [the Rust Style document](https://github.com/maidsafe/QA/blob/master/Documentation/Rust%20Style.md#git-commit-messages). We don't have any hard-and-fast rules for the PR body or branch name, but try to look at existing PRs first to get a sense of how they are structured.

### Code Style

We are following the company-wide code style guide that you can find in the [the Rust Style document](https://github.com/maidsafe/QA/blob/master/Documentation/Rust%20Style.md). You should install `rustfmt` and `clippy` and run them automatically for each of your Git commits, and you should follow the common commit message style (for an example you can take a look at the list of commits in [our repository](https://github.com/maidsafe/safe_client_libs/commits/master).

In addition to the common code style, we have our own [coding convention](./FFI+calling+conventions) that we use for the FFI (functions interfacing with other languages).

### Running tests (CI script)

Submitted PRs are expected to pass continuous integration (CI), which, among other things, runs a test suite on your PR to make sure that your code doesn't have any bugs.

Please refer to [Testing Client Libs](./Testing-Client-Libs) and ensure you have appropriately and comprehensively run tests on your branch before submitting.