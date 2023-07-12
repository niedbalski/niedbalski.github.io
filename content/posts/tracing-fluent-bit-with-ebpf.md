---
title: "Tracing Fluent Bit With Ebpf"
date: 2023-07-11T00:17:32+02:00
draft: false
---

# Tracing Fluent Bit with eBPF

[Fluent-bit](https://fluentbit.io/) is a lightweight, highly configurable, and extensible data processor. This post aims to illustrate how to instrument Fluent-bit using eBPF uprobes and Golang.

## Uprobes

Uprobes, provided by eBPF, permit the instrumentation of specific user-level functions in a program without modifying its source code or requiring recompilation. 

## Libbpf go

[Libbpfgo](https://github.com/aquasecurity/libbpfgo) is a Go library from Aqua Security, enabling the creation and management of BPF programs.

## Writing eBPF programs

eBPF programs have two significant limitations:

1. Limited Stack Size: eBPF programs operate with a stack size of only 512 bytes. 
2. Poor Error Reporting: eBPF verifier's error messages are often cryptic and hard to interpret without a deep understanding of eBPF internals.

Also, debugging eBPF programs often relies on the use of `bpf_trace_printk()`, which has its limitations.

## Tracing Fluent-bit

Fluent-bit processes data via an input-filter-output pipeline. To trace messages as they pass through this pipeline, mechanisms such as [Tap](https://docs.fluentbit.io/manual/administration/troubleshooting#tap-functionality) and [Vivo](https://github.com/calyptia/vivo), both developed by [Calyptia](https://calyptia.com), or classical ptrace debugging can be employed.

## eBPF Program for Fluent-bit

Given the limitations of eBPF programs, they can only communicate with userland through eBPF maps, which are versatile containers used to store and share data. There are multiple types of eBPF maps, each serving specific purposes. However, they also have limitations, such as fixed size, limited capacity, restricted key types, and adherence to the single writer principle.

A simple program is shown below that attaches to the part of the pipeline where logs are appended. This program instantiates a ring buffer map, attaches a uprobe to [`flb_input_log_append`](https://github.com/fluent/fluent-bit/blob/7c3136cb171125936932c9a0740da63ae2e2a733/src/flb_input_log.c#L82), captures two parameters through PT_REGS_*, and sends the event to the ring buffer.

```C
#include <linux/bpf.h>
#include <linux/ptrace.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>
#include <bpf/bpf_tracing.h>

#define MAX_BUFFER_SIZE 4096

struct event_data {
    __u64 buffer_size;
    char buffer[MAX_BUFFER_SIZE];
};

struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 1 << 24);
} events SEC(".maps");

SEC("uprobe/flb_input_log_append")
int uprobe__flb_input_log_append(struct pt_regs *ctx)
{
    __u64 buf_size = PT_REGS_PARM5(ctx);
    if (buf_size > MAX_BUFFER_SIZE)
        buf_size = MAX_BUFFER_SIZE;

    struct event_data *event = bpf_ringbuf_reserve(&events, sizeof(struct event_data), 0);
    if (!event)
        return 0;

    event->buffer_size = buf_size;

    if (bpf_probe_read(event->buffer, buf_size, (void *)PT_REGS_PARM4(ctx)) != 0) {
        bpf_ringbuf_discard(event, 0);
        return 0;
    }

    // Print buffer size for debugging
    bpf_printk("Buffer size: %llu\n", event->buffer_size);

    bpf_ringbuf_submit(event, 0);
    return 0;
}

char _license[] SEC("license") = "GPL";

```

A makefile to build this program could look like the following:

```makefile
CMD_CLANG ?= clang
CMD_MKDIR ?= mkdir -p

OBJ_SRC = ./c/probes.c
OUTPUT_DIR = ./dist

LINUX_ARCH := $(shell uname -m)
UNAME_M := $(shell uname -m)

ifeq ($(UNAME_M),aarch64)
	LINUX_ARCH = arm64
endif

ifeq ($(LINUX_ARCH),x86_64)
	INCLUDE_PATH := /usr/include/x86_64-linux-gnu/
else ifeq ($(LINUX_ARCH),arm64)
	INCLUDE_PATH := /usr/include/aarch64-linux-gnu/
else
    $(error Unsupported architecture: $(LINUX_ARCH))
endif

LIBBPF_PATH := /usr/lib/$(UNAME_M)-linux-gnu/libbpf.a

.PHONY: help
help:
	@echo "Available targets:"
	@echo "  bpf        Compile the eBPF object file"
	@echo "  build      Build the project"

.PHONY: bpf
bpf: check-clang check-output-dir
	$(CMD_CLANG) \
		-D__TARGET_ARCH_$(LINUX_ARCH) \
		-D__BPF_TRACING__ \
		-I /usr/local/include \
		-I $(INCLUDE_PATH) \
		-target bpf \
		-O2 -g \
		-march=bpf \
		-c $(OBJ_SRC) \
		-o $(OUTPUT_DIR)/probes.bpf.o

.PHONY: build
build: bpf
	CC=gcc CGO_CFLAGS="-I $(INCLUDE_PATH)" CGO_LDFLAGS="$(LIBBPF_PATH)" go build main.go

.PHONY: check-clang
check-clang:
	@command -v $(CMD_CLANG) >/dev/null 2>&1 || { echo "Error: $(CMD_CLANG) is not available in the system. Please install $(CMD_CLANG) or update the CMD_CLANG variable in the Makefile."; exit 1; }

.PHONY: check-output-dir
check-output-dir:
	@test -d $(OUTPUT_DIR) || { $(CMD_MKDIR) $(OUTPUT_DIR); }

```

Then, to attach a Golang program to the ring buffer, the following code can be used:

```go
package main

import (
	"bytes"
	"encoding/binary"
	"fmt"
	"os"

	bpf "github.com/aquasecurity/libbpfgo"
	"github.com/aquasecurity/libbpfgo/helpers"
)

type EventData struct {
	BufSize uint64
	Buffer  []byte `msgpack:",omitempty"`
}

func resizeMap(module *bpf.Module, name string, size uint32) error {
	m, err := module.GetMap(name)
	if err != nil {
		return err
	}

	if err = m.Resize(size); err != nil {
		return err
	}

	if actual := m.GetMaxEntries(); actual != size {
		return fmt.Errorf("map resize failed, expected %v, actual %v", size, actual)
	}

	return nil
}

func formatUprobeSymbol(symbolName string) string {
	return fmt.Sprintf("uprobe__%s", symbolName)
}

func main() {
	args := os.Args[1:]

	if len(args) < 2 {
		fmt.Fprintln(os.Stderr, "wrong syntax")
		os.Exit(-1)
	}

	binaryPath := args[0]
	symbolName := args[1]

	_, err := os.Stat(binaryPath)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(-1)
	}

	bpfModule, err := bpf.NewModuleFromFile("./dist/probes.bpf.o")
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(-1)
	}
	defer bpfModule.Close()

	if err = resizeMap(bpfModule, "events", 8192); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(-1)
	}

	bpfModule.BPFLoadObject()
	prog, err := bpfModule.GetProgram(formatUprobeSymbol(symbolName))
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(-1)
	}

	offset, err := helpers.SymbolToOffset(binaryPath, symbolName)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(-1)
	}

	_, err = prog.AttachUprobe(-1, binaryPath, offset)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(-1)
	}

	eventsChannel := make(chan []byte)
	rb, err := bpfModule.InitRingBuf("events", eventsChannel)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(-1)
	}

	defer rb.Stop()
	defer rb.Close()

	go rb.Start()

	for eventBytes := range eventsChannel {
		event := EventData{}
		buf := bytes.NewReader(eventBytes)

		err := binary.Read(buf, binary.LittleEndian, &event.BufSize)
		if err != nil {
			fmt.Println("binary.Read failed:", err)
			continue
		}

		if len(eventBytes) < int(event.BufSize)+8 {
			fmt.Println("Buffer size is larger than the slice, discard the event.")
			continue
		}

		event.Buffer = eventBytes[8 : event.BufSize+8]
		fmt.Println("Buffer size:", event.BufSize)
		fmt.Println("Buffer data:", string(event.Buffer))
	}
}

```

You can then build the Golang program using the Makefile:

```makefile
make build
```

To run:

```
sudo ./main /opt/fluent-bit/bin/fluent-bit flb_input_log_append
```

In parallel, run Fluent-bit with this configuration:

```ini
[INPUT]
    name     dummy
    Dummy    {"message": "I am a very large buffer"}
    Threaded on

[OUTPUT]
    name stdout
    match *
```

The output should display the buffer size and data as follows:

```
Buffer size: 55
Buffer data: I am a very large buffer
```

To conclude, leveraging eBPF uprobes for function-level tracing in Fluent-bit provides powerful diagnostic capabilities that goes beyond the classical techniques. Happy debugging!