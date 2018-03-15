# basic-boshrelease

A very very very basic BOSH release.

## What's this?

A BOSH release that can be used to quickly and easily test assumptions about the BOSH/monit lifecycle.

## Observations

Environment:
  * bosh-lite built from bosh-deployment `9b394c1f9f3287728d7aff6ef1d8bb8605f3c0e4`
  * bosh-warden-boshlite-ubuntu-trusty-go_agent 3541.8

### 1. Happy path - reasonable monit timeout and a _ctl that is quick to start

```
# configuration

monit job timeout        = 30
seconds to sleep in _ctl = 0

canary_watch_time        = 1000-60000
update_watch_time        = 1000-60000
```

```
# result

# grep basic /var/vcap/monit/monit.log
[UTC Mar 15 16:06:05] error    : 'basic' process is not running
[UTC Mar 15 16:06:05] info     : 'basic' trying to restart
[UTC Mar 15 16:06:05] info     : 'basic' start: /var/vcap/jobs/basic/bin/basic_ctl
[UTC Mar 15 16:06:16] info     : 'basic' process is running with pid 4784

# deployment output

Task 176 | 16:05:31 | Creating missing vms: basic/e8622ac9-a1cc-48f0-8212-92a6bff8fcd6 (0) (00:00:21)
Task 176 | 16:05:53 | Updating instance basic: basic/e8622ac9-a1cc-48f0-8212-92a6bff8fcd6 (0) (canary) (00:00:18)

Task 176 Started  Thu Mar 15 16:05:31 UTC 2018
Task 176 Finished Thu Mar 15 16:06:11 UTC 2018
Task 176 Duration 00:00:40
Task 176 done

Succeeded
```

```
# analysis

It takes monit 11 seconds to determine the process is running.
It take BOSH 18 seconds to determine the deployment has succeeded.
Deployment succeeded.
```
