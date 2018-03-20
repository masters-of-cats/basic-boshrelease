# basic-boshrelease

A very very very basic BOSH release.

## What's this?

A BOSH release that can be used to quickly and easily to test assumptions about the BOSH/monit lifecycle.

## TL;DR Learnings

* Make absolutely sure to include the `group vcap` line for processes in a BOSH job monit file
  * Otherwise BOSH essentially ignores the job and deployment will false-positively succeed
* monit is quick (~1 second) to transition jobs to running once the PID file has been written
* If a job takes longer than its timeout to startup the deployment can still succeed, assuming a). the canary/update timeout has not been reached, and b). it didn't take more than twice its monit timeout to start
  * We observed that if a BOSH job takes longer than twice its monit timeout to start, it enters a `Execution failed` state, at which point the BOSH deployment always fails, regardless of the canary/update timeout
  * We also observed that even though monit was reporting `Execution failed` for the job, the underlying process was actually running fine
* Each time the monit timeout is reached, monit will start another instance of the process
  * In other words, it's totally possible to end up with multiple instances of the process running at the same time
  * We guard against this in garden by checking for the existence of a PID file and bailing out early if one exists

## Experiments

### 1). What happens when a BOSH job takes longer than the monit timeout to start

```
# configuration

* monit timeout               = 30 seconds
* startup                     = 31 seconds
* BOSH canary/update timeout  = 1000-100000
```

```
# monit logs

[UTC Mar 19 16:26:32] error    : 'basic' process is not running
[UTC Mar 19 16:26:32] info     : 'basic' trying to restart
[UTC Mar 19 16:26:32] info     : 'basic' start: /var/vcap/jobs/basic/bin/basic_ctl
[UTC Mar 19 16:26:37] info     : start service 'basic' on user request
[UTC Mar 19 16:27:02] error    : 'basic' failed to start
[UTC Mar 19 16:27:02] info     : 'basic' start: /var/vcap/jobs/basic/bin/basic_ctl
[UTC Mar 19 16:27:04] info     : 'basic' started
[UTC Mar 19 16:27:04] info     : 'basic' start action done
[UTC Mar 19 16:27:14] info     : 'basic' process is running with pid 4774

# monit summary after deployment has finished

The Monit daemon 5.2.5 uptime: 3m

Process 'basic'                     running
System 'system_localhost'           running
```

```
# deployment output

Task 343 | 16:26:12 | Preparing deployment: Preparing deployment (00:00:01)
Task 343 | 16:26:13 | Preparing package compilation: Finding packages to compile (00:00:00)
Task 343 | 16:26:13 | Creating missing vms: basic/8505969a-06d6-4149-96e5-88e5d038fa62 (0) (00:00:06)
Task 343 | 16:26:20 | Updating instance basic: basic/8505969a-06d6-4149-96e5-88e5d038fa62 (0) (canary) (00:01:02)

Task 343 Started  Mon Mar 19 16:26:12 UTC 2018
Task 343 Finished Mon Mar 19 16:27:22 UTC 2018
Task 343 Duration 00:01:10
Task 343 done

Succeeded
```

```
# analysis

* The deployment succeeds
* monit detects the job as failing exactly 30 seconds (the configured monit timeout) after the initial start
* It then calls start again (thus causing 2 x processes to be running at the same time)
* 32 seconds after the first start, monit detects the job as `running` (31 second startup time + 1 second to notice)
```

### 2). What happens when a BOSH job takes significantly longer (>= twice) the monit timeout to start

```
# configuration

* monit timeout               = 30 seconds
* startup                     = 61 seconds
* BOSH canary/update timeout  = 1000-100000
```

```
# monit logs

[UTC Mar 20 08:19:33] error    : 'basic' process is not running
[UTC Mar 20 08:19:33] info     : 'basic' trying to restart
[UTC Mar 20 08:19:33] info     : 'basic' start: /var/vcap/jobs/basic/bin/basic_ctl
[UTC Mar 20 08:19:38] info     : start service 'basic' on user request
[UTC Mar 20 08:20:03] error    : 'basic' failed to start
[UTC Mar 20 08:20:03] info     : 'basic' start: /var/vcap/jobs/basic/bin/basic_ctl
[UTC Mar 20 08:20:33] error    : 'basic' failed to start
[UTC Mar 20 08:20:33] info     : 'basic' start action done
[UTC Mar 20 08:20:44] info     : 'basic' process is running with pid 4717
[UTC Mar 20 08:21:04] error    : 'basic' process PID changed from 4717 to 4793
[UTC Mar 20 08:21:14] info     : 'basic' process PID has not changed since last cycle

# monit summary after deployment has finished

The Monit daemon 5.2.5 uptime: 3m

Process 'basic'                     Execution failed
System 'system_localhost'           running
```

```
# deployment output

Task 365 | 08:19:14 | Preparing deployment: Preparing deployment (00:00:00)
Task 365 | 08:19:14 | Preparing package compilation: Finding packages to compile (00:00:00)
Task 365 | 08:19:14 | Creating missing vms: basic/bd5388d5-737d-47b0-a1cb-0d3102d96450 (0) (00:00:07)
Task 365 | 08:19:21 | Updating instance basic: basic/bd5388d5-737d-47b0-a1cb-0d3102d96450 (0) (canary) (00:01:57)
                    L Error: 'basic/bd5388d5-737d-47b0-a1cb-0d3102d96450 (0)' is not running after update. Review logs for failed jobs: basic
Task 365 | 08:21:18 | Error: 'basic/bd5388d5-737d-47b0-a1cb-0d3102d96450 (0)' is not running after update. Review logs for failed jobs: basic

Task 365 Started  Tue Mar 20 08:19:14 UTC 2018
Task 365 Finished Tue Mar 20 08:21:18 UTC 2018
Task 365 Duration 00:02:04
Task 365 error

Updating deployment:
  Expected task '365' to succeed but state is 'error'

Exit code 1
```

```
# analysis

* The deployment fails
* monit detects the job as failing exactly 30 seconds (the configured monit timeout) after the initial start
* It then calls start again (thus causing 2 x processes to be running at the same time)
* monit detects the second instance of the job as failing exactly 30 seconds (the configured monit timeout) after the second start
* 60 seconds after the inital start, monit detects the job as Started
* 71 seconds after the inital start monit reports the PID
* monit then sees the pid change (as the second instance finishes startup and overwrites the PID file)
```

### 3). What happens when a BOSH job takes just under twice the monit timeout to start

```
# configuration

* monit timeout               = 30 seconds
* startup                     = 59 seconds
* BOSH canary/update timeout  = 1000-100000
```

```
# monit logs

[UTC Mar 20 08:26:56] error    : 'basic' process is not running
[UTC Mar 20 08:26:56] info     : 'basic' trying to restart
[UTC Mar 20 08:26:56] info     : 'basic' start: /var/vcap/jobs/basic/bin/basic_ctl
[UTC Mar 20 08:27:01] info     : start service 'basic' on user request
[UTC Mar 20 08:27:26] error    : 'basic' failed to start
[UTC Mar 20 08:27:26] info     : 'basic' start: /var/vcap/jobs/basic/bin/basic_ctl
[UTC Mar 20 08:27:55] info     : 'basic' started
[UTC Mar 20 08:27:55] info     : 'basic' start action done
[UTC Mar 20 08:28:05] info     : 'basic' process is running with pid 4707
[UTC Mar 20 08:28:25] error    : 'basic' process PID changed from 4707 to 4716


# monit summary after deployment has finished

The Monit daemon 5.2.5 uptime: 1m

Process 'basic'                     running
System 'system_localhost'           running

```

```
# deployment output

Task 379 | 08:26:38 | Preparing deployment: Preparing deployment (00:00:00)
Task 379 | 08:26:38 | Preparing package compilation: Finding packages to compile (00:00:00)
Task 379 | 08:26:38 | Creating missing vms: basic/a3314456-e32c-4ecc-83fa-0758beafc59b (0) (00:00:07)
Task 379 | 08:26:45 | Updating instance basic: basic/a3314456-e32c-4ecc-83fa-0758beafc59b (0) (canary) (00:01:23)

Task 379 Started  Tue Mar 20 08:26:38 UTC 2018
Task 379 Finished Tue Mar 20 08:28:08 UTC 2018
Task 379 Duration 00:01:30
Task 379 done

Succeeded

```

```
# analysis

* The deployment succeeds
* similar monit logs to experiment 2, but in this case monit only detects the job as `failed to start` once
```

### 4). What happens if the monit timeout is longer than the BOSH canary/update timeout?

```
# configuration

* monit timeout               = 70 seconds
* startup                     = 65 seconds
* BOSH canary/update timeout  = 1000-60000
```

```
# monit logs

[UTC Mar 20 08:59:46] error    : 'basic' process is not running
[UTC Mar 20 08:59:46] info     : 'basic' trying to restart
[UTC Mar 20 08:59:46] info     : 'basic' start: /var/vcap/jobs/basic/bin/basic_ctl
[UTC Mar 20 08:59:51] info     : start service 'basic' on user request
[UTC Mar 20 09:00:52] info     : 'basic' start action done
[UTC Mar 20 09:00:53] info     : 'basic' process is running with pid 4709

# monit summary after deployment has finished

The Monit daemon 5.2.5 uptime: 1m

Process 'basic'                     running
System 'system_localhost'           running
```

```
# deployment output

Task 400 | 08:59:28 | Preparing deployment: Preparing deployment (00:00:00)
Task 400 | 08:59:28 | Preparing package compilation: Finding packages to compile (00:00:00)
Task 400 | 08:59:28 | Creating missing vms: basic/f6e26b8f-749b-4cb1-aa0f-546fc3304fd8 (0) (00:00:07)
Task 400 | 08:59:35 | Updating instance basic: basic/f6e26b8f-749b-4cb1-aa0f-546fc3304fd8 (0) (canary) (00:01:16)
                    L Error: 'basic/f6e26b8f-749b-4cb1-aa0f-546fc3304fd8 (0)' is not running after update. Review logs for failed jobs: basic
Task 400 | 09:00:51 | Error: 'basic/f6e26b8f-749b-4cb1-aa0f-546fc3304fd8 (0)' is not running after update. Review logs for failed jobs: basic

Task 400 Started  Tue Mar 20 08:59:28 UTC 2018
Task 400 Finished Tue Mar 20 09:00:51 UTC 2018
Task 400 Duration 00:01:23
Task 400 error

Updating deployment:
  Expected task '400' to succeed but state is 'error'

Exit code 1
```

```
# analysis

* The deployment fails, but the job eventually transitions to `running` afterwards as the startup time is less than the monit timeout
```
