---
title: How to Install Kong Performance Testing Framework
badge: oss
content-type: how-to
---

The {{site.base_gateway}} codebase includes a performance testing framework. It allows Kong developers and users to evaluate the performance of Kong, and any bundled or custom plugins. The performance testing framework also allows you to create [Flame Graphs](https://github.com/brendangregg/FlameGraph) out of the data. The framework collects RPS (request per second) and latency data of Kong processing a request, and uses it to represent performance metrics under different workloads. This framework is implemented as an extension to Kong's integrated test suite.

## Install dependencies

The framework requires several Lua development libraries including [busted](https://olivinelabs.com/busted/), a unit testing library. To install all of the required dependencies follow these steps: 

1. Clone the Kong Github repository: 
    
        git clone https://github.com/Kong/kong.git

2. From within the directory run `make dev`. 


## Performance testing

The performance tests are written in Lua. One test can be shared across multiple test cases. 

This example is a standard way to write tests with the Busted framework. This test demonstrates a basic testing workflow that defines a workload and then performs load testing on Kong.

```lua
local perf = require("spec.helpers.perf")

perf.use_driver("docker")

local versions = { "git:master", "2.4.0" }

for _, version in ipairs(versions) do
  describe("perf test for Kong " .. version .. " #simple #no_plugins", function()
    local bp
    lazy_setup(function()
      local helpers = perf.setup()

      bp = helpers.get_db_utils("postgres", {
        "routes",
        "services",
      })

      local upstream_uri = perf.start_upstream([[
      location = /test {
        return 200;
      }
      ]])

      local service = bp.services:insert {
        url = upstream_uri .. "/test",
      }

      bp.routes:insert {
        paths = { "/s1-r1" },
        service = service,
        strip_path = true,
      }
    end)

    before_each(function()
      perf.start_kong(version, {
        --kong configs
      })
    end)

    after_each(function()
      perf.stop_kong()
    end)

    lazy_teardown(function()
      perf.teardown()
    end)

    it("#single_route", function()
      local results = {}
      for i=1,3 do
        perf.start_load({
          path = "/s1-r1",
          connections = 1000,
          threads = 5,
          duration = 10,
        })

        ngx.sleep(10)

        local result = assert(perf.wait_result())

        print(("### Result for Kong %s (run %d):\n%s"):format(version, i, result))
        results[i] = result
      end

      print(("### Combined result for Kong %s:\n%s"):format(version, assert(perf.combine_results(results))))

      perf.save_error_log("output/" .. version:gsub("[:/]", "#") .. "-single_route.log")
    end)
  end)
end
```

Copy the code to a file titled: `testing_spec.lua`. You can run the code from within the Kong directory using the following command: 

`bin/busted path/to/this_file_spec.lua`

The output will look like this: 

```
### Result for Kong git:96326b894f712b5d03bb1bf7ac02d531f6128cd1 (run 1):
Running 10s test @ http://10.88.145.9:8000/s1-r1
  5 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    17.51ms   61.46ms   1.04s    98.99%
    Req/Sec    14.31k     2.61k   20.80k    82.59%
  672831 requests in 10.07s, 154.07MB read
  Socket errors: connect 0, read 0, write 0, timeout 246
Requests/sec:  66803.45
Transfer/sec:     15.30MB
### Result for Kong git:96326b894f712b5d03bb1bf7ac02d531f6128cd1 (run 2):
Running 10s test @ http://10.88.145.9:8000/s1-r1
  5 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    14.71ms   10.66ms  96.77ms   66.18%
    Req/Sec    14.45k     1.79k   20.19k    70.80%
  718942 requests in 10.08s, 164.70MB read
Requests/sec:  71337.25
Transfer/sec:     16.34MB
### Result for Kong git:96326b894f712b5d03bb1bf7ac02d531f6128cd1 (run 3):
Running 10s test @ http://10.88.145.9:8000/s1-r1
  5 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    14.47ms   10.13ms  82.81ms   66.57%
    Req/Sec    14.60k     1.64k   22.98k    72.00%
  726452 requests in 10.07s, 166.42MB read
Requests/sec:  72141.41
Transfer/sec:     16.53MB
### Combined result for Kong git:96326b894f712b5d03bb1bf7ac02d531f6128cd1:
RPS     Avg: 70094.04
Latency Avg: 15.52ms    Max: 1040.00ms
```

For more testing examples, review the [spec/04-perf](https://github.com/Kong/kong/tree/master/spec/04-perf) folder in the Kong
directory.

{:.note}
>**Note:** The samples in the [spec/04-perf](https://github.com/Kong/kong/tree/master/spec/04-perf) Kong directory
contain tests with error logs, and Flame Graphs files. If you run those tests, the files will be saved to the `output` directory.


## More Information