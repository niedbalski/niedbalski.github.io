---
title: "Writing Golang Fluent Bit Input Plugins"
date: 2023-02-25T14:40:16+02:00
draft: false
---

# Writing Golang Fluent Bit Input Plugins

## Context

Fluent Bit is a highly reliable and memory-efficient pipeline data processing engine. It is widely used as a low-forwarder in the Cloud-native landscape and supports metrics and traces as core features. While Fluent Bit itself is written in C, it provides extensible interfaces that allow input, filter, and output plugins to be written in various languages, including Golang.

At Calyptia, we have developed a Golang library to support writing input plugins for Fluent Bit. You can find the library at [https://github.com/calyptia/plugin](https://github.com/calyptia/plugin).

## How to Write a Golang Input Plugin

To demonstrate how easy it is to extend Fluent Bit's input capabilities, let's consider a scenario where we want to monitor the status of GitHub and send an alert to a Slack channel in case of an outage. We will break down the implementation into the following steps:

1. Fetch the GitHub status API.
2. Filter the response to check if the status is operational. We will implement this using a Fluent Bit Lua filter.
3. Dispatch a message to Slack. We will utilize the Fluent Bit Slack output plugin for this part.

For Step 1, we will implement a custom Fluent Bit input plugin using Golang.

### Implementing the Plugin

Implementing an input plugin in Golang requires implementing a simple interface defined in the `github.com/calyptia/plugin` package:

```go
type InputPlugin interface {
	Init(ctx context.Context, fbit *Fluentbit) error
	Collect(ctx context.Context, ch chan<- Message) error
}
```

You can find further information on implementing this interface in the `github.com/calyptia/plugin` repository.

Here's an example implementation of a Fluent Bit input plugin for monitoring GitHub status:

```go
package main

import (
	"context"
	"encoding/json"
	"errors"
	"io"
	"log"
	"net/http"
	"time"

	"github.com/calyptia/plugin"
)

const (
	GithubStatusEndpointJSON = "https://www.githubstatus.com/api/v2/status.json"
	ScrapeInterval           = time.Minute * 5
)

type GithubStatusResponse struct {
	Page struct {
		ID        string    `json:"id"`
		Name      string    `json:"name"`
		URL       string    `json:"url"`
		TimeZone  string    `json:"time_zone"`
		UpdatedAt time.Time `json:"updated_at"`
	} `json:"page"`
	Status struct {
		Indicator   string `json:"indicator"`
		Description string `json:"description"`
	} `json:"status"`
}

func init() {
	plugin.RegisterInput("go-fluentbit-github-status", "Golang input plugin for checking the status of GitHub", &GithubStatusPlugin{})
}

type GithubStatusPlugin struct{}

func (plug *GithubStatusPlugin) Init(ctx context.Context, fbit *plugin.Fluentbit) error {
	return nil
}

func (plug *GithubStatusPlugin) Collect(ctx context.Context, ch chan<- plugin.Message) error {
	tick := time.NewTicker(ScrapeInterval)
	for {
		select {
		case <-ctx.Done():
			err := ctx.Err()
			if err != nil && !errors.Is(err, context.Canceled) {
				return err
			}
			return nil
		case <-tick.C:
			resp, err := http.Get(GithubStatusEndpointJSON)
			if err != nil {
				log.Fatal(err)
			}

			responseData, err := io.ReadAll(resp.Body)
			if err != nil {
				log.Fatal(err)
			}

			var githubStatusResponse GithubStatusResponse
			err = json.Unmarshal(responseData, &githubStatusResponse)
			if err != nil {
				log.Fatal(err)
			}

			ch <- plugin.Message{
				Time: time.Now(),
				Record: map[string]interface{}{
					"data": githubStatusResponse,
				},
			}
		}
	}
}

func main() {}
```

In this example, we implement the `Collect` method, which periodically fetches the GitHub status information and sends it as a message. We use the `time` and `net/http` packages to make the API request and process the response using JSON encoding.

### Building the Plugin

To build the plugin, use the following command:

```shell
go build -trimpath -buildmode c-shared -o github_status.so .
```

If you are using an M1-based machine, compile the plugin with the following command:

```shell
CGO_ENABLED=1 \
GOOS=linux \
GOARCH=amd64 \
CC="zig cc -target x86_64-linux-gnu -isystem /usr/include -L/usr/lib/x86_64-linux-gnu" \
CXX="zig c++ -target x86_64-linux-gnu -isystem /usr/include -L/usr/lib/x86_64-linux-gnu" \
go build -trimpath -buildmode c-shared -o github_status.so .
```

The resulting `github_status.so` file should be placed in the `/fluent-bit/etc` directory.

### Configuration

To complete Steps 2 and 3, you need a Fluent Bit configuration that includes filtering the input records and sending them to Slack. Here's an example Fluent Bit configuration (`fluent-bit.yaml`) that uses the Go Fluent Bit GitHub Status plugin:

```yaml
service:
  flush: 1
  log_level: debug
  plugins_file: /fluent-bit/etc/plugins.conf
  Parsers_file: /fluent-bit/etc/parsers.conf

pipeline:
  inputs:
    - Name: go-fluentbit-github-status

  filters:
    - Name: lua
      Match: '*'
      call: filter
      code: |
        function filter(tag, timestamp, record)
          if record.status and record.status.status and record.status.status.description == "All Systems Operational" then
            record = { status = "✅ All GitHub systems are operational" }
          else
            record = { status = "❌ Issues with GitHub (check: https://www.githubstatus.com/)" }
          end
          return 2, timestamp, record
        end

  outputs:
    - Name: slack
      Match: '*'
      webhook: https://hooks.slack.com/xxxx
```

Replace `https://hooks.slack.com/xxxx` with the actual webhook URL for your Slack integration.

Additionally, create a `plugins.conf` file with the following content:

```ini
[PLUGINS]
    Path /fluent-bit/etc/go-fluentbit-github-status.so
```

### Running the Plugin

Finally, run the latest Fluent Bit release with the plugin loaded and executed using the following command:

```shell
docker run -v $(pwd)/github_status.so:/fluent-bit/etc/go-fluentbit-github-status.so -v $(pwd)/fluent-bit.yaml:/fluent-bit/etc/fluent-bit.yaml:ro -v $(pwd)/plugins.conf:/fluent-bit/etc/plugins.conf:ro fluent/fluent-bit:2.1.2 -c /fluent-bit/etc/fluent-bit.yaml
```

Adjust the volume mounts (`$(pwd)` represents the current directory) and the Fluent Bit image tag according to your environment.

With this configuration, the Go Fluent Bit GitHub Status plugin will periodically check the GitHub status and rewrite the record's "status" field to indicate whether all systems are operational or if there are issues. The output can be sent to a Slack channel using the `slack` output plugin or modified to fit your specific use case.

### Conclusion

Congratulations! You have successfully written a Golang input plugin for Fluent Bit. This demonstrates how easy it is to extend Fluent Bit's functionality using Golang and integrate custom features into your data processing pipeline. Feel free to explore other possibilities and experiment with different plugins and configurations to suit your specific requirements.
The plugin works flawless, it runs every 5 minutes and reports back  the status to the slack channel.

![Slack output](https://raw.githubusercontent.com/niedbalski/go-fluentbit-github-status/043678219646b139e62d3b9be1779b01f5868428/slack-output.png)

#### Links

The source code of this experiment can be found here: [https://github.com/niedbalski/go-fluentbit-github-status](https://github.com/niedbalski/go-fluentbit-github-status).
