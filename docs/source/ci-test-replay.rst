CI test replay
==============

The Travis tox test environments use ``ci_replay.ReplayDiscoverRunner`` to
replay known failures first, then interrupted tests, tests affected by changed
files, historically risky tests, and the remaining tests. The runner stops
after the first test failure. The existing ``flake8`` and ``isort`` tox jobs
are unchanged.

The replay history is best-effort. It is scoped by repository, branch, pull
request, commit, Travis run, job, tox environment, and Python runtime. This
project does not configure a persistent Travis cache for ``.ci-replay-cache``;
many jobs therefore start cold and run tests in their normal discovery order.
History is a local optimization, not durable state or a correctness
requirement. The cache is ignored by Git and invalid or unavailable history is
discarded.

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
