# Notes on the CPython project

These are some personal notes on how to work with the CPython project. They
focus on triaging and merging pull requests.

## Roles

CPython uses labels for many purposes. Only triagers and core developers
can apply labels. Some may be added or removed by bots.

Triagers can also do the following:

- Close or reopen issues
- Close or reopen pull requests (sometimes useful to retrigger CI)

Core developers can:

- Merge pull requests into main and the bugfix branches
- Click the button to approve CI for new contributors
- Edit existing PR branches. This is useful when giving small
  feedback (e.g., through the suggestions UI for reviews): it may be
  quicker and easier to apply the change yourself than wait for the PR author.
- Click the "Update branch" button to update a branch with main
  (useful for old PRs where the new CLA check hasn't run yet)

## Buildbots

Every PR runs CPython's regular CI, which includes running the full
test suite on a few operating systems, building the docs, and running
`make patchcheck`.

Optionally, PRs can be tested on the buildbot fleet by adding the
"test with buildbots" label. The buildbots run on a wide variety of
configurations and systems. In my experience, the following are the
most important unique aspects:

- A few buildbots check for reference leaks. Getting reference counts
  wrong is an extremely common bug in CPython's C code, so this is
  where buildbots most frequently help find issues.
- They run on some unusual operating systems, so they exercise
  code paths that don't run on the usual CI (e.g., there are some
  big-endian systems).

I tend to run the buildbots for every nontrivial change that affects
C code. For changes that only affect Python code or the docs, the
buildbots don't add much value. It may also be useful to run the
buildbots for changes in Python modules that interact directly
with the OS, such as `ssl` or `asyncio`, but I don't usually work on
such modules.

## Backporting

In CPython, virtually all pull requests are first made against
`main`, the feature branch, which represents the next 3.x version
to be released. After a PR is merged into `main`, if there is a
"needs backport to 3.x" label on the PR, the Miss Islington bot
will automatically try to create a backport PR applying the change
to the 3.x branch.

Sometimes the bot can't create a PR because there is a conflict.
In that case, you have to use the `cherry_picker` tool on the command
line to fix the conflict and create the backport PR manually.
Occasionally the bot posts a message saying something like
"could not checkout the 3.x branch". In that case, simply remove
the label for that branch and reapply it, and the bot will try again, usually
successfully.

The bot will automatically merge backport PRs that have been approved
by a core dev after CI passes. I will usually look over the PR
to make sure it makes sense, approve it, and assign it to myself.
I check the list of PRs assigned to me regularly, so if CI doesn't
pass or something, I'll come back to the PR later to get it merged.
If you manually created the backport PR with `cherry_picker`, it
won't be automerged. My workflow is to check back a few hours later
and merge if CI has passed by then.

### What to backport?

But the most difficult question is: When do we backport? The short
answer is that bugfixes get backported and new features don't, but
what is a bug and what is a feature?

Here are some rules I've come up with:

- Security issues do get backported, even to the security-only branches.
  (The status of each branch is listed in [the devguide](https://devguide.python.org/#status-of-python-branches).)
  This includes issues that can cause the interpreter itself to crash,
  because those can trigger DoS vulnerabilities, and segfault bugs often
  allow arbitrary code execution. Only the release manager for the branch
  is allowed to merge changes into the security branches.
- Documentation improvements virtually always get backported (unless
  they apply only to a newer version). We want users on all versions
  to see improved documentation as soon as possible, and there is very
  little risk that such changes will cause problems for any user.
- New or improved tests should usually be backported (provided, of course,
  that they apply to features that exist in the backport branch). This makes us
  more confident when we make additional changes to the tested code later.
  However, some core devs may not agree.
- Performance improvements do not get backported. There is always a
  risk they break something, and users who want improvements should
  upgrade to a newer version.
- Clear new features (e.g., new functions or parameters) do not get
  backported.
- For behavior changes that are on the border between bug fix and
  enhancement, think about whether users could be relying on the old
  behavior. We want upgrades to bugfix releases (3.x.y -> 3.x.y+1) to
  be completely painless, so if users could reasonably rely on some
  behavior, we shouldn't change it.

## "skip issue" and "skip news"

Significant changes to CPython require an issue to be created, and most
also require a news entry. You can use the `blurb` command-line
tool or the `blurb-it` website to create a news entry.

### When to skip news

News entries are used to create a changelog with all user-relevant
changes since the previous release. Therefore, every change that
could potentially affect users should have a news entry. Some
categories of changes that don't need news include:

- Minor documentation changes. However, news entries may be appropriate
  for bigger changes, such as documenting a previously undocumented
  feature.
- Changes that affect only the unit tests. However, a news entry may
  still be useful if the change fixes an issue noticed by end users.
- Fixes to changes that were never released. For example, if I merge
  a PR implementing `typing.AwesomeFeature`, and then the next day
  someone notices a problem with the implementation and proposes a PR
  to fix it, the second PR doesn't need a news entry. When the next
  release comes out, users will simply see that `AwesomeFeature` was
  added and they don't need to know that we went through multiple PRs
  to get the released version.
- Purely internal changes, such as changes to the CI configuration.

There is little harm in adding a news entry if it is not strictly
required, so if I'm in doubt and the PR author has already created a
news entry, I'll usually let it stand.

### When to skip issue

News entries are keyed on the issue number, so every change that
requires a news entry also requires an issue. The converse isn't always
true: sometimes a change fixes an issue, but doesn't need a news entry.
For example, wording changes in the documentation are often discussed
in an issue first, but usually don't require a news entry. Similarly,
fixes to unreleased features may be discussed in an issue, but don't
usually need a news entry.

## Merging a PR

There is no requirement that core developers get approval from
anyone else before merging a PR, but I feel that we should strongly
avoid merging unreviewed code. For my own PRs, ideally I would
want review from another core dev, or failing that from someone else
who I trust enough to know that they will review the PR with a critical
eye.

When reviewing a PR from a non-core dev, I'll approve the PR when I
feel it's ready to merge, then wait a while to give others a chance
to chime in and tell me I'm wrong. This is especially important for
older PRs where another core dev has previously provided feedback.
If they requested changes, it's best to get explicit signoff from them
before merging.

## Closing a PR

Sometimes a PR clearly isn't going to be merged. In that case, it
should be closed. Example scenarios:

- The feature request that the PR implements was rejected in an issue.
- The PR is badly out of date (e.g., merge conflicts) and the author
  is not responding.
- The problem that the PR fixes was already addressed. Sometimes there
  are multiple PRs doing the same thing, and we only need to merge one.
- The PR makes a purely cosmetic change that doesn't clearly improve
  the code.

It's not a good experience for users when their PR gets closed,
so I try to be diplomatic.
