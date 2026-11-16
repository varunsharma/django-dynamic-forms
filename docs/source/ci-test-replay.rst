CI test replay
==============

The Travis tox test environments use ``ci_replay.ReplayDiscoverRunner`` to
replay known failures first, then interrupted tests, tests affected by changed
files, historically risky tests, and the remaining tests. The runner stops
after the first test failure. Every existing ``TOXENV`` remains represented
in the Travis job list, and each is placed in its own ordered stage so a failed
job prevents later environments from starting. This includes the existing
Django-master lanes and the ``flake8`` and ``isort`` jobs.

Replay history is best-effort. Its stable cache key is scoped by repository,
branch, pull request, head ref, tox lane, and Python runtime. Each unresolved
test or collection failure separately retains its originating repository,
pull request, head/base refs and SHAs, commit, run ID/attempt, job, tox lane,
and Python runtime until a clean pass; the bounded recent-context list is
supplemental. Provenance is not part of the cache key. The cache stores test
identifiers and bounded risk counts, never test output. The cache is ignored
by Git.

Travis restores ``.ci-replay-cache`` for pull-request builds, but only stores
cache modifications from normal pushes, so a new pull-request failure may not
persist to its next PR run. Concurrent runs sharing a stable key may restore
the same snapshot and the last saved snapshot can omit updates from another
run. Missing, stale, invalid, or unavailable history starts with the broad
candidate set. History is not durable state or a correctness requirement.
PR, base SHA, or run-attempt values may be empty when Travis does not expose
them. See https://docs.travis-ci.com/user/caching/ for cache write rules.

The adapter keeps tests in the same ``TestCase`` class together so Django's
class-level fixtures remain valid. Changed-file matching is a heuristic; tests
whose relationship to changed code is not recognized remain in the
historically risky or remaining groups and are still run unless an earlier
failure stops the job.

The ordered singleton Travis stages provide build-wide fail-fast: Travis
cancels later stages after a failed stage, while normal Django cleanup for the
failing job still runs. The former allowed-failure Django-master lanes are now
required stages so their test failures also stop the build. No branch-wide,
commit-wide, or pull-request-wide cancellation is performed.

To run the focused adapter checks locally, use::

    python -m unittest discover -s ci_replay_tests -v

These checks are separate from Django's application test discovery. The
project's normal tox command and coverage threshold remain in effect for the
application suite.
