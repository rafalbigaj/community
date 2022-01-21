---
status: proposed
title: Env in POD template
creation-date: '2021-12-21'
last-updated: '2021-12-21'
authors:
- '@rafalbigaj'
---

# TEP-0098: Env in POD template

<!--
**Note:** When your TEP is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary", and "Motivation" sections.
  These should be easy if you've preflighted the idea of the TEP with the
  appropriate Working Group.
- [ ] **Create a PR for this TEP.**
  Assign it to people in the SIG that are sponsoring this process.
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the TEP clarified and merged quickly.  The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.

Just because a TEP is merged does not mean it is complete or approved.  Any TEP
marked as a `proposed` is a working document and subject to change.  You can
denote sections that are under active debate as follows:

```
<<[UNRESOLVED optional short context or usernames ]>>
Stuff that is being argued.
<<[/UNRESOLVED]>>
```

When editing TEPS, aim for tightly-scoped, single-topic PRs to keep discussions
focused.  If you disagree with what is already in a document, open a new PR
with suggested changes.

If there are new details that belong in the TEP, edit the TEP.  Once a
feature has become "implemented", major changes should get new TEPs.

The canonical place for the latest set of instructions (and the likely source
of this file) is [here](/teps/NNNN-TEP-template/README.md).

-->

<!--
This is the title of your TEP.  Keep it short, simple, and descriptive.  A good
title can help communicate what the TEP is and should be considered as part of
any review.
-->

<!--
A table of contents is helpful for quickly jumping to sections of a TEP and for
highlighting any additional information provided beyond the standard TEP
template.

Ensure the TOC is wrapped with
  <code>&lt;!-- toc --&rt;&lt;!-- /toc --&rt;</code>
tags, and then generate with `hack/update-toc.sh`.
-->

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
  - [Use Cases](#use-cases)
- [Requirements](#requirements)
- [Proposal](#proposal)
  - [Notes/Caveats (optional)](#notescaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
  - [User Experience (optional)](#user-experience-optional)
  - [Performance (optional)](#performance-optional)
- [Design Details](#design-details)
- [Test Plan](#test-plan)
- [Design Evaluation](#design-evaluation)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Upgrade &amp; Migration Strategy (optional)](#upgrade--migration-strategy-optional)
- [Implementation Pull request(s)](#implementation-pull-request-s)
- [References (optional)](#references-optional)
<!-- /toc -->

## Summary

A [Pod template](https://github.com/tektoncd/pipeline/blob/main/docs/podtemplates.md) should support configuration
of environment variables, which are combined with those defined in steps and `stepTemplate`, and then passed to all step containers.
That allows to exclude common variables to the global level as well as overwrite defaults specified 
on the particular step level.

## Motivation

One of the most important motivators for this feature is the ability to eliminate redundant code from
the `PipelineRun` as well as `TaskRun` specification. 
In case of complex pipelines, which consist of dozen or even hundreds of tasks, any kind of repetition 
significantly impacts the final size of `PipelineRun` and leads to resource exhaustion.

Besides, users quite frequently are willing to overwrite environment variable values
specified in a `stepTemplate` in the single place when running pipelines. 
That helps to optionally pass settings, without a need to define additional pipeline parameters.

Having an `env` field in the pod template allows to:

- specify global level defaults, what is important to reduce the size of `TaskRun` and `PipelineRun`
- override defaults from `stepTemplate` at `TaskRun` and `PipelineRun` level

### Goals

1. The main goal of this proposal is to enable support for specification of environment variables on the global level 
(`TaskRun` and `PipelineRun`).
   
2. Environment variables defined in the Pod template at `TaskRun` and `PipelineRun` level 
   take precedence over ones defined in steps and `stepTemplate`.

### Non-Goals

<!--
What is out of scope for this TEP?  Listing non-goals helps to focus discussion
and make progress.
-->

### Use Cases

1. In the first case, common environment variables can be defined in a single place on a `PipelineRun` level. 
   Values can be specified as literals or through references. 
   Variables defined on a `PipelineRun` or `TaskRun` level are then available in all steps.
   That allows to significantly reduce the size of `PipelineRun` and `TaskRun` resource, 
   by excluding the common environment variables like: static global settings, common values coming from metadata, ...

2. Secondly, environment variables defined in steps can be easily overwritten by the ones from `PipelineRun` and `TaskRun`.
  With that, common settings like API keys, connection details, ... can be optionally overwritten in a single place.

3. Additionally, environment variables "injection" may be useful in case of relatively generic tasks. 
   The example could be a build or deployment task, which executes code from user's repository
   (and thus, may depend on custom environment variables). You may think about `make`, `golang-test`, `ansible`, ... 
   Without the possibility to provide environment variables in runtime it would be harder to deliver such a generic task, 
   and users would be forced to write specific tasks per each use case separately.

## Requirements

1. Environment variables defined in `podTemplate` on a `PipelineRun` or `TaskRun` level are passed to all steps.
2. Those values take precedence over environment variable values defined in steps and `stepTemplate`.

## Proposal

Common environment variables can be defined in a single place on a `PipelineRun` level.
Values can be specified as literals or through references, e.g.:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: mypipelinerun
spec:
  podTemplate:
    env:
    - name: TOKEN_PATH
      value: /opt/user-token
    - name: TKN_PIPELINE_RUN
      valueFrom:
        fieldRef:
          fieldPath: metadata.labels['tekton.dev/pipelineRun']
  pipelineSpec:
    tasks:
    - name: make
      taskSpec:
        spec:
          useRuntimeEnv: true
          env:
          - name:  TASK_VAR
            value: a value
```

Environment variables defined in steps can be easily overwritten by the ones from a `TaskRun`, e.g.:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: mytask
  namespace: default
spec:
  useRuntimeEnv: true
  steps:
  - name: echo-msg
    image: ubuntu
    command: ["bash", "-c"]
    args: ["echo $MSG"]
    env:
    - name: "MSG"
      value: "Default message"
---
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: mytaskrun
  namespace: default
spec:
  taskRef:
    name: mytask
  podTemplate:
    env:
    - name: "MSG"
      value: "Overwritten message"
```

The flag `useRuntimeEnv` allow to opt-in for 

### Notes/Caveats (optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above.
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

### Risks and Mitigations

<!--
What are the risks of this proposal and how do we mitigate. Think broadly.
For example, consider both security and how this will impact the larger
kubernetes ecosystem.

How will security be reviewed and by whom?

How will UX be reviewed and by whom?

Consider including folks that also work outside the WGs or subproject.
-->

### User Experience (optional)

<!--
Consideration about the user experience. Depending on the area of change,
users may be task and pipeline editors, they may trigger task and pipeline
runs or they may be responsible for monitoring the execution of runs,
via CLI, dashboard or a monitoring system.

Consider including folks that also work on CLI and dashboard.
-->

### Performance (optional)

No performance implications assuming that `TaskRun` does not contain inlined `taskSpec` with all resolved variables
(see [discussion](https://github.com/tektoncd/pipeline/issues/1606#issuecomment-561593839)).

## Design Details

N/A

## Test Plan

<!--
**Note:** *Not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy.  Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).
-->

## Design Evaluation
<!--
How does this proposal affect the api conventions, reusability, simplicity, flexibility 
and conformance of Tekton, as described in [design principles](https://github.com/tektoncd/community/blob/master/design-principles.md)
-->

## Drawbacks

<!--
Why should this TEP _not_ be implemented?
-->

## Alternatives

Users may specify common variables in task spec, but that does not solve an issue with the size of `PipelineRun`
resource when tasks have embedded specs.

## Upgrade & Migration Strategy (optional)

No impact on existing features.

## Implementation Pull request(s)

https://github.com/tektoncd/pipeline/pull/3566

## References (optional)

https://github.com/tektoncd/pipeline/issues/1606