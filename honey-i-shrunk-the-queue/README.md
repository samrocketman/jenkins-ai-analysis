# Honey I shrunk the queue patch series

When launching a Jenkinsfile with the `parallel` step with 100 build agents in
parallel with the following flow nodes:

```
Map parallelAgents = [failFast: false]
for (int i = 1; i <= 100; i++) {
    String agentNum = "Agent ${agentNum}"
    parallelAgents["Agent ${agentNum}"] = { ->
        stage(agentNum) {
            node('some-label') {
                docker.image('alpine').inside('--init') {
                    sshagent(['some-credentials']) {
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
