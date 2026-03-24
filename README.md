# AI Analysis Documents

I am starting to perform AI analysis on Jenkins core and plugins to identify
performance issues and attempt to fix them.

This is part of an overall self-led effort to make Jenkins performance comparable to other off-site hosted CI solutions which can be extremely fast such as GitHub Actions.

For Jenkins to compete, it needs an overhaul in how it views speed to verify even simple tasks.  i.e. if a Jenkins admin provides the resources Jenkins should be able to take full advantage to go as fast as it can manage.

My current analysis initiatives:

- [Ludicrous mode patch series](ludicrous-mode-analysis) which aims to solve Queueing delays when Jenkins needs to schedule work on agents.
