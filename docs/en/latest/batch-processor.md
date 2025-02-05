---
title: Batch Processor
---

<!--
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
-->

The batch processor can be used to aggregate entries(logs/any data) and process them in a batch.
When the batch_max_size is set to zero the processor will execute each entry immediately. Setting the batch max size more
than 1 will start aggregating the entries until it reaches the max size or the timeout expires.

## Configurations

The only mandatory parameter to create a batch processor is a function. The function will be executed when the batch reaches the max size
or when the buffer duration exceeds.

|Name           |Requirement    |Description|
|-------        |-----          |------|
|name           |optional       |A unique identifier to identity the batch processor|
|batch_max_size |optional       |Max size of each batch, default is 1000|
|inactive_timeout|optional      |maximum age in seconds when the buffer will be flushed if inactive, default is 5s|
|buffer_duration|optional       |Maximum age in seconds of the oldest entry in a batch before the batch must be processed, default is 5|
|max_retry_count|optional       |Maximum number of retries before removing from the processing pipe line; default is zero|
|retry_delay    |optional       |Number of seconds the process execution should be delayed if the execution fails; default is 1|

The following code shows an example of how to use batch processor in your plugin:

```lua
local bp_manager_mod = require("apisix.utils.batch-processor-manager")
...

local plugin_name = "xxx-logger"
local batch_processor_manager = bp_manager_mod.new(plugin_name)
local schema = {...}
local _M = {
    ...
    name = plugin_name,
    schema = batch_processor_manager:wrap_schema(schema),
}

...


function _M.log(conf, ctx)
    local entry = {...} -- data to log

    if batch_processor_manager:add_entry(conf, entry) then
        return
    end
    -- create a new processor if not found

    -- entries is an array table of entry, which can be processed in batch
    local func = function(entries)
        -- serialize to json array core.json.encode(entries)
        -- process/send data
        return true
        -- return false, err_msg if failed
    end
    batch_processor_manager:add_entry_to_new_processor(conf, entry, ctx, func)
end
```

The batch processor's configuration will be set inside the plugin's configuration.
For example:

```shell
curl http://127.0.0.1:9080/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
      "plugins": {
            "http-logger": {
                "uri": "http://mockbin.org/bin/:ID",
                "batch_max_size": 10,
                "max_retry_count": 1
            }
       },
      "upstream": {
           "type": "roundrobin",
           "nodes": {
               "127.0.0.1:1980": 1
           }
      },
      "uri": "/hello"
}'
```

If your plugin only uses one global batch processor,
you can also use the processor directly:

```lua
local entry = {...} -- data to log
if log_buffer then
    log_buffer:push(entry)
    return
end

local config_bat = {
    name = config.name,
    retry_delay = config.retry_delay,
    ...
}

local err
-- entries is an array table of entry, which can be processed in batch
local func = function(entries)
    ...
    return true
    -- return false, err_msg if failed
end
log_buffer, err = batch_processor:new(func, config_bat)

if not log_buffer then
    core.log.warn("error when creating the batch processor: ", err)
    return
end

log_buffer:push(entry)
```

Note: Please make sure the batch max size (entry count) is within the limits of the function execution.
The timer to flush the batch runs based on the `inactive_timeout` configuration. Thus, for optimal usage,
keep the `inactive_timeout` smaller than the `buffer_duration`.
