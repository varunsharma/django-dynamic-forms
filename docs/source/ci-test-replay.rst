CI test replay
==============

The Travis tox test environments use ``ci_replay.ReplayDiscoverRunner`` to
replay known failures first, then interrupted tests, tests affected by changed
files, historically risky tests, and the remaining tests. The runner stops
after the first test failure. The existing ``flake8`` and ``isort`` tox jobs
are unchanged.

The replay history is best-effort. It is scoped by repository, branch, pull
request, commit, Travis run, job, tox environment, and Python runtime. Travis
caches ``.ci-replay-cache`` between normal pushes and restores caches for
pull-request builds, but only stores cache modifications from normal pushes.
New failures from a pull-request build are not guaranteed to persist to its
next run. A missing, stale, invalid, or unavailable cache starts with the broad
candidate set, and the runner still stops at the
first failure. History is not durable state or a correctness requirement.
PR, base SHA, or run-attempt values may be empty when Travis does not expose
them to a job. See https://docs.travis-ci.com/user/caching/ for cache scope
and write rules. The cache is ignored by Git.

Each tox environment is one Travis job. Fail-fast stops later tests in that
job, while normal Django test runner cleanup still runs. Travis does not
provide reliable cancellation of already running sibling matrix jobs when
one job fails, so other matrix jobs may continue. No branch-wide, commit-wide,
or pull-request-wide cancellation is performed.

The adapter keeps tests in the same ``TestCase`` class together so Django's
class-level fixtures remain valid. Changed-file matching is a heuristic; tests
whose relationship to changed code is not recognized remain in the
historically risky or remaining groups and are still run unless an earlier
failure stops the job.

To run the focused adapter checks locally, use::

    python -m unittest discover -s ci_replay_tests -v

These checks are separate from Django's application test discovery. The
project's normal tox command and coverage threshold remain in effect for the
application suite.

Unresolved test and collection failures keep their originating repository, pull request, head/base refs and SHAs, run ID/attempt, job, tox lane, and Python runtime attached per test ID. That provenance has no 12-run expiry and is removed only after a clean pass; the bounded recent-context list is supplemental. The cache schema change starts once from cold history when older cached data is present.
