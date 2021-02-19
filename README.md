# Ansible Role: job-runner

An Ansible Role that installs a job runner - a wrapper script to execte your cron tasks, while allowing their monitoring by Prometheus.

The job runner prevents multiple execution of each job a time (uses locking per job).

## Requirements

*node_exporter* service installed, configured with (path configurable, see Role Variables):

    --collector.textfile.directory=/srv/node_exporter/textfile_collector

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

    runner_script_path: '/opt/bin/job-runner'
    runner_user:        'jobrunner'
    runner_group:       'jobrunner'
    runner_id:          990
    runner_gid:         990
    runner_scripts_dir: '/opt/scripts'
    textfile_dir:       '/srv/node_exporter/textfile_collector'

## Usage

Example of running sript `/opt/scripts/my-check.py` (path configurable, see Role Variables) from cron, every minute, with logger tag `pycheck`, asking for an alert if this script hasn't finished `300` seconds since the previous script execution finished:

*Inside `/etc/cron.d/jobrunner`*:

    * * * * * jobrunner [ -x /opt/bin/job-runner ] && /opt/bin/job-runner my-check.py pycheck 300 2>&1 | logger -t pycheck

This will produce metrics file `/srv/node_exporter/textfile_collector/runner_my-check.py.prom` with similar content to:

    # HELP runner_exit Exit code of runner.
    # TYPE runner_exit gauge
    runner_exit{script="my-check.py"} 0
    # HELP runner_last Time latest run finished.
    # TYPE runner_last gauge
    runner_last{script="my-check.py"} 1609760085
    # HELP runner_duration Duration of latest run.
    # TYPE runner_duration gauge
    runner_duration{script="my-check.py"} 7
    # HELP runner_next_run_in Expect another run within this time period.
    # TYPE runner_next_run_in gauge
    runner_alert_after{script="my-check.py"} 300

We can see that our my-check.py script finished at 1609760085, with error code 0, and took 7 seconds to finish.
If a Prometheus is configured to scrape the node exporter on this server, these variables and their history will be available in Prometheus.

We can alert *ALL* cron jobs executed this way on *ALL* servers using this job runner for *non-zero exit code* using this Prometheus condition:

    runner_exit != 0

We can allert *ALL* cron jobs executed this way on *ALL* servers using this job runner for *not finishing for more than advertised time since the previously finished time* using this Prometheus condition:

    time() - runner_last > runner_alert_after

## Dependencies

None.

## Example Playbook

    - hosts: utilservers
      vars_files:
        - vars/main.yml
      roles:
        - { role: job-runner }

*Inside `vars/main.yml`*:

    runner_user:        'jobrunner'
    runner_group:       'jobrunner'
    runner_id:          940
    runner_gid:         940

## License

MIT / BSD

## Author Information

Based on / inspired by blogpost: https://phrye.com/code/periodic-monitoring/ - many thanks.

This ansible role was written by Marji Cermak - https://github.com/marji/

