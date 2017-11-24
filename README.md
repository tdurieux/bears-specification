# Specification of the BEARS repository layout and infrastucture for bug datasets

This document specifies the layout of Git repositories and the associated infrastructure for collecting and storing bugs.

In this document, a bug dataset means a set of bugs for which there exists at least one failing test case (ie the code is compiled and executed).

In the first version of this specification, we only consider Java software built using Maven.

## Components

A BEARS repository contains:

* a set of Git branches, one branch per bug
* a continuous integration daemon that checks the validity of new entries in the dataset
* a master script, called `bears`, to interact with the repository.


## Submission of new bugs to the dataset

To propose a new bug, one makes a pull request (PR) to branch "pr_new_bug" of the repository. Submissions can be made by humans (a researcher creates the PR files and opens it) , or by robots (a robot automatically prepares the submission and creates the PR).

This PR must contain at least:
* a well-formed Maven file called `pom.xml` at the root of the branch
* the source code of the application and tests, usually in `src/(test|main)/java` but not necessarily, this is inferred from pom.xml
* a file called `metadata.json` a follows
* a file called `patch.diff`
 
Format of `metadata.json`:

```js
{ 
  "project_name" : "foo", //mandatory
  "project_git_url" : "http://github.com/foo/bar", //mandatory
  "commit_sha1" : "eeeeeee", //mandatory
  "issue_url": "http://fooo.com/jira/1253", // optional
  "comment": "free test",  // optional
  "author_name": "free test"  // optional
}
```

## The BEARS CI daemon

The BEARS CI deamon checks the following on each new pull request, ie on each new submission to the dataset:

* the presence, well-formedness and content validity of `metadata.json`
* that the PR fails when `mvn test` is run
* that the failure is due to a test failure (by looking in `target/surefire-report`)
* that `patch.diff` can be applied by running "git apply patch.diff" at the root of the PR
* that `patch.diff` only changes Java files
* that the PR succeeds when `mvn test` is run after applying the patch
* that `mvn test` fails in less than 20 minutes and then succeeds in less than 20 minutes
* that the PR does not contain a `.travis.yml` file (this file provided automatically by the dataset infrastructure)

When a PR is merged, the CI daemon does different steps:

* automatically picks the next free numerical branch identifier for this project, as given by the project name in the metadata (eg `foo002` if `foo001` already exists), 
* automatically creates `metadata-derived.json` which metadata inferred from running the buggy and the patched version
  * with the list of test names (from surefire)
  * the stack trace of test failures and errors (from surefire)
  * the coverage
  * eg some automatic repair information
* pushes the new branch to the main repository
* pushes the metadata files to special branch `all_metadata`, in a folder with the bug identifier, ie `foo001/metadata.json` and `foo001/metadata-derived.json`

## Architecture of the Git repository

The core idea is that there is one branch per bug in the dataset. There are named with a unique identifier composed of the project name (as given in `metadata.json`) suffixed with a unique identifier on 3 digits (eg `foo001`).

There are three special branches.

* branch "pr_new_bug" that only contains a `.travis.yml` and helper scripts in folder ".travis". `.travis.yml` contains the code of the BEARS CI daemon that checks the validity of the proposed bug.
* branch `master` contains scripts to use the dataset, the code of the UI dashboard to browse the dataset and a README.
* branch `all_metadata` that concatenates all the metadata for easy and efficient use.

