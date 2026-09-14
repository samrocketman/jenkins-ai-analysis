# Honey I shrunk the queue patch series

When launching a Jenkinsfile with the `parallel` step with 100 build agents in
parallel a simple `echo hello` build can take over 20 minutes in my production
environment.  I've identified the root cause across three plugins.

- [docker-workflow-plugin] blocks CPS VM thread preventing CPS pipeline
  continuing (forced serialization) when running parallel step.
- [ssh-agent-plugin] blocks CPS VM thread, preventing CPS pipeline continuing
  (forced serialization) when running parallel step.
- [github-checks-plugin] synchronously blocks the creation and completion of CPS
  pipeline flow nodes which causes pipelines to grind to crawl unless you
  suppress progress updates in pipeline settings.  Though the parts it still
  does notify on is still problematic just not as problematic as all progress
  updates.

[docker-workflow-plugin]: https://github.com/jenkinsci/docker-workflow-plugin
[ssh-agent-plugin]:  https://github.com/jenkinsci/ssh-agent-plugin
[github-checks-plugin]: https://github.com/jenkinsci/github-checks-plugin

Related pull requests:

- https://github.com/jenkinsci/docker-workflow-plugin/pull/756

Example Jenkinsfile
-------------------

The following Jenkinsfile can produce the problematic behavior.

```groovy
Map parallelAgents = [failFast: false]
for (int i = 1; i <= 100; i++) {
    String agentNum = "Agent ${agentNum}"
    parallelAgents["Agent ${agentNum}"] = { ->
        stage(agentNum) {
            echo 'before node'
            node('some-label') {
                echo 'before docker'
                docker.image('alpine').inside('--init') {
                    echo 'before sshagent'
                    sshagent(['some-credentials']) {
                        echo 'before sh'
                        sh "echo hello from ${agentNum}"
                    }
                }
            }
        }
    }
}
parallel(parallelAgents)
```

The build can take over 10 minutes.  This is because `docker.inside` and
`sshagent` steps block the CPS VM thread which bottlenecks the execution and
serializing it.  Making all 100 nodes behave as if they were serialized.

This patch series reduces that "100 nodes in paralllel" to under 30 seconds
total runtime including agent orchestration.

These bugs were discovered during pressure testing of
https://github.com/jenkinsci/ec2-plugin/pull/2036

The fixes are closely related to my vendetta against slow Jenkins pipelines from
my "ludicrous mode" patch series across other Jenkins plugins.
