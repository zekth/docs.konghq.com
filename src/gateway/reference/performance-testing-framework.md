---
title: Performance Testing Framework
badge: oss
content-type: reference
---


## Drivers

Three "drivers" are implemented depending on different environments, accuracy
requirements, and setup complexity.

| Driver   | Test between git commits | Test between binary releases | Flame Graph | Test unreleased version    |
|----------|--------------------------|------------------------------|------------|----------------------------|
| local    | yes                      |                              | yes        | yes                        |
| docker   | yes                      | yes                          |            | yes (use_daily_image = true) |
|terraform | yes                      | yes                          | yes        | yes (use_daily_image = true) |

- **local** Driver reuses users' local environment. It's faster to run,
but the RPS and latency number may be influenced by other local programs and thus inaccurate.

    * Requires Lua development dependencies of Kong, OpenResty, and `wrk` be installed.
    * Requires SystemTap, kernel headers, and build chain to be installed if generating FlameGraph.

- **docker** Driver is solely based on Docker images. It's the most convenient driver to setup
as it requires less dependencies. But it may also be influenced by other local programs
and Docker network performance. And it doesn't support FlameGraph generation.

- **terraform** Driver runs in remote VM or bare metal machine. It's most accurate,
but it requires Terraform knowledge to operate and setup.

    * Requires the [Terraform](https://www.terraform.io/downloads.html) binary be installed.
    * Requires git binary if testing between git commits. When testing between git commits,
    the framework assumes the current directory is Kong's repo. It will stash your working
    directory and unstash after test is finished. When using the docker or terraform driver,
    the framework derives the base version of each commit and uses the matching Docker image or
    Kong binary package and puts local source code inside.

## API

### perf.use_defaults()

*syntax: perf.use_defaults()*

Use default parameters. This function sets the following:
- Looks up `PERF_TEST_DRIVER` envrionment variable and invokes `perf.use_driver`.
- Invokes `perf.set_log_level` and sets level to `"debug"`.
- Set retry count to 3.
- Looks up `PERF_TEST_METAL_PROJECT_ID` and `PERF_TEST_METAL_AUTH_TOKEN` envrionment variables
and pass to the terraform driver if it's selected.
- Looks up `PERF_TEST_USE_DAILY_IMAGE` envrionment variable and passes the variable to driver options.

### perf.use_driver

*syntax: perf.use_driver(name, options?)*

Uses driver name, which must be one of "local", "docker"  or  "terraform". Additional
parameters for the driver can be specified in options as a Lua table. Throws error if any.

The Docker and Terraform driver expect an options parameter to contain the following keys :

- **use_daily_image**: a boolean to decide whether to use daily image or not; useful when testing
unreleased commits on master branch; downloading Enterprise daily images requires a valid Docker
login.

The Terraform driver expects options to contain the following optional keys:

- **provider** The service provider name, right now only "equinix-metal".
- **tfvars** Terraform variables as a Lua table; for `equinix-metal` provider,
`packet_project_id` and `packet_auth_token` is required.

### perf.set_log_level

*syntax: perf.set_log_level(level)*

Sets the log level; expect one of `"debug"`, `"info"`, `"notice"`, `"warn"`, `"error"` and
`"crit"`. The default log level is `"info"` unless `perf.use_defaults` is called and set
to `"debug"`.

### perf.set_retry_count

*syntax: perf.set_retry_count(count)*

Sets retry time for each “driver" operation. By default every operation is retried 3 times.

### perf.setup

*syntax: helpers = perf.setup()*

Prepares environment and returns the `spec.helpers` module. Throws error if any.

The framework sets up some environment variables before importing `spec.helpers` modules.
The returned helper is just a normal `spec.helpers` module. Users can use the same pattern
in integration tests to setup entities in Kong. DB-less mode is currently not implemented.

### perf.start_upstream

*syntax: upstream_uri = perf.start_upstream(nginx_conf_blob)*

Starts upstream (Nginx/OpenResty) with given `nginx_conf_blob`. Returns the upstream
URI accessible from Kong's view. Throws error if any.

### perf.start_upstreams

*syntax: upstream_uris = perf.start_upstreams(nginx_conf_blob, port_count)*

Like `perf.start_upstream`, but starts the upstream Nginx with the given configuration and multiple ports.

### perf.start_kong

*syntax: perf.start_kong(version, kong_configs?)*

Starts Kong with given version and Kong configurations in `kong_configs` as a Lua table.
Throws error if any.

To select a git hash or tag, use `"git:<hash>"` as version. Otherwise, the framework
will treat the `version` as a release version and will download binary packages from
Kong's distribution website.

With the Docker driver, this function can be called multiple times to spawn multiple Kong
instances.

### perf.start_hybrid_kong

*syntax: perf.start_hybrid_kong(version, kong_configs?)*

Starts a hybrid mode cluster that consists of one control plane
and one data plane. Only the Docker driver supports this function. Kong configurations necessary
to bring up hybrid mode like `role` or `cluster_control_plane` do not need to be set
in `kong_configs`; the test framework will set them up automatically.

### perf.stop_kong

*syntax: perf.stop_kong()*

Stops all Kong instances. Throws error if any.

### perf.teardown

*syntax: perf.teardown(full?)*

Teardown. Throws error if any. With the terraform driver, setting full to true terminates
all infrastructure, while by default it does cleanup only to speed up successive runs.

### perf.start_stapxx

*syntax: perf.start_stapxx(stapxx_file_name, arg?)*

Starts the Stap++ script with `stapxx_file_name` exists in
[available stapxx scripts](https://github.com/Kong/stapxx/tree/kong/samples)
and additional CLI args. Throws error if any.

This function blocks test execution until the `SystemTap` module is fully prepared and inserted into the
kernel. It should be called before `perf.start_load`.

### perf.start_load

*syntax: perf.start_load(options?)*

Starts to send load to Kong using `wrk`. Throws error if any. Options is a Lua table that may contain the following:

- **path** String request path; defaults to `/ `.
- **uri** Base URI exception path; defaults to `http://kong-ip:kong-port/`.
- **connections** Connection count; defaults to 1000.
- **threads** Request thread count; defaults to 5. 
- **duration** Number of performance tests duration in seconds; defaults to 10. 
- **script** Content of `wrk` script as string; defaults to nil.

### perf.get_admin_uri

*syntax: perf.get_admin_uri()*

Return the Admin API URL. The URL may only be accessible from
the load generator and not publically available. When multiple Kong instances are started, this function
returns the first available Kong instance that exposes the admin port.

### perf.wait_result

*syntax: result = perf.start_load(options?)*

Waits for the load test to finish and returns the result as a string. Throws error if any.

Currently, this function waits indefinitely, or until both `wrk` and Stap++ processes exit.

### perf.combine_results

*syntax: combined_result = perf.combine_results(results, …)*

Calculates multiple results returned by `perf.wait_result` and returns their average and min/max.
Throws error if any.

### perf.generate_flamegraph

*syntax: perf.generate_flamegraph(path, title?)*

Generates a FlameGraph with title and saves to path. Throws error if any.

### perf.save_error_log

*syntax: perf.save_error_log(path)*

Saves Kong error log to path. Throws error if any.

### perf.enable_charts

*syntax: perf.enable_charts(on?)*

Enable charts generation, throws error if any.

Charts are enabled by default; the data of charts are fed by `perf.combine_results`.
The generated charts are `SVG` files and saved under `output` directory.

### perf.save_pgdump

*syntax: perf.save_pgdump(path)*

After setting up Kong using Blueprint, saves the PostgreSQL database dump to a path. Throws error if any.

### perf.load_pgdump

*syntax: perf.load_pgdump(path, dont_patch_service?)*

Loads the pgdump from the path and replaces the Kong database with the loaded data; this function
will also patch a services address to the upstream unless `dont_patch_service` is set to false,
it must be called after `perf.start_upstream`. Throws error if any.

## Customization

### Add new test suite

All tests are stored in `spec/04-perf/01-rps` and `spec/04-perf/02-flamegraph` of the Kong repository.
Add a new file under one of the directories and put `#tags` in the test description.

### Add new provider in terraform

Users can use the terraform driver in most major service providers as long as
it's supported by Terraform. The following contracts are made between the framework and terraform module:

The terraform files are stored in `spec/fixtures/perf/terraform/<provider>`.

Two instances are provisioned, one for running Kong and another for running an upstream
and load (worker). A firewall allows bidirectional traffic between the two instances
and from the public internet. Both instances run Ubuntu 20.04/focal.

An SSH key to login into both instances exists or will be created in
`spec/fixtures/perf/terraform/<provider>/id_rsa`. The logged-in user has root privilege.

The following are terraform output variables:

- **kong-ip** Kong node public IP.
- **kong-internal-ip** Kong node internal IP (if unavailable, provide kong-ip).
- **worker-ip** Worker node public IP.
- **worker-internal-ip** Worker node internal IP (if unavailable, provide worker-ip).
