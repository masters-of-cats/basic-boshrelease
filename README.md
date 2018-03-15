# basic-boshrelease

A very very very basic BOSH release.

## What's this?

A BOSH release that can be used to quickly and easily test assumptions about the BOSH/monit lifecycle.

## Observations

Environment:
  * bosh-lite built from bosh-deployment `9b394c1f9f3287728d7aff6ef1d8bb8605f3c0e4`
  * bosh-warden-boshlite-ubuntu-trusty-go_agent 3541.8

### Experiment 1

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
[UTC Mar 15 16:18:31] error    : 'basic' process is not running
[UTC Mar 15 16:18:31] info     : 'basic' trying to restart
[UTC Mar 15 16:18:31] info     : 'basic' start: /var/vcap/jobs/basic/bin/basic_ctl
[UTC Mar 15 16:18:42] info     : 'basic' process is running with pid 4774

# cat /var/vcap/sys/log/basic/basic_ctl.log
15/03/2018 - 16:18:31 begin sleep
15/03/2018 - 16:18:31 end sleep
15/03/2018 - 16:18:31 wrote pidfile
15/03/2018 - 16:18:31 starting basic

# deployment output

Task 182 | 16:17:58 | Creating missing vms: basic/dcb989c5-b2b3-4915-ac14-3b62dfd849d0 (0) (00:00:20)
Task 182 | 16:18:19 | Updating instance basic: basic/dcb989c5-b2b3-4915-ac14-3b62dfd849d0 (0) (canary) (00:00:18)

Task 182 Started  Thu Mar 15 16:17:58 UTC 2018
Task 182 Finished Thu Mar 15 16:18:37 UTC 2018
Task 182 Duration 00:00:39
Task 182 done

Succeeded
```

```
# analysis

monit starts the process at `16:18:31`
the pidfile gets written at `16:18:31`
monit detects the process running at `16:18:42` (11 seconds after pidfile is written)
deployment is successful
```

### Experiment 2

```
# configuration

monit job timeout        = 10
seconds to sleep in _ctl = 20

canary_watch_time        = 1000-60000
update_watch_time        = 1000-60000
```

```
# result

# grep basic /var/vcap/monit/monit.log
[UTC Mar 15 16:24:24] error    : 'basic' process is not running
[UTC Mar 15 16:24:24] info     : 'basic' trying to restart
[UTC Mar 15 16:24:24] info     : 'basic' start: /var/vcap/jobs/basic/bin/basic_ctl
[UTC Mar 15 16:24:34] error    : 'basic' failed to start
[UTC Mar 15 16:24:44] info     : 'basic' process is running with pid 4716

# cat /var/vcap/sys/log/basic/basic_ctl.log
15/03/2018 - 16:24:24 begin sleep
15/03/2018 - 16:24:44 end sleep
15/03/2018 - 16:24:44 wrote pidfile
15/03/2018 - 16:24:44 starting basic

# deployment output

Task 189 | 16:23:52 | Creating missing vms: basic/1e59cd71-3023-448b-be77-bf71cc9c23ca (0) (00:00:20)
Task 189 | 16:24:12 | Updating instance basic: basic/1e59cd71-3023-448b-be77-bf71cc9c23ca (0) (canary) (00:00:18)

Task 189 Started  Thu Mar 15 16:23:52 UTC 2018
Task 189 Finished Thu Mar 15 16:24:30 UTC 2018
Task 189 Duration 00:00:38
Task 189 done

Succeeded
```

```
# analysis

monit starts the process at `16:24:24`
monit notices the process hasn't started at `16:24:34`
the pidfile gets written at `16:24:44`
monit detects the process running at `16:24:44` (0 seconds after pidfile is written)
deployment is successful
```
