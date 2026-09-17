# Agent Note: Explicit Web all-interfaces binding

Status: implemented

English | [中文](2026-09-10-web-all-interfaces-bind.zh.md)

## Problem

The Web runtime already derives LAN address trust and prints a reachable LAN URL when its server binds `0.0.0.0`, but the command-line provider rejected that explicit host before the runtime could activate. This made the documented LAN path unreachable and forced operators toward untracked proxy workarounds.

## Decision

Accept an explicit `--host 0.0.0.0` exactly like any other bind address. The existing runtime samples non-loopback IPv4 addresses and adds them to the browser Host/Origin trust fence; `--trusted-host` remains the way to authorize DNS authorities. The default bind remains `127.0.0.1`, so remote exposure is still an operator choice.

## Alternatives considered

**Require a reverse proxy or port forward.** Rejected because it bypasses the runtime's LAN address discovery, produces no canonical LAN URL, and moves the same exposure into configuration the Harness cannot validate or explain.

**Change the default bind.** Rejected because LAN exposure must remain explicit. Existing invocations continue to use loopback.

## Consequences

Trusted-network deployments can serve the authenticated Web UI directly on the LAN. Because the UI controls an agent with shell and code-execution tools, package documentation now calls out that binding all interfaces exposes this capability to every reachable client that can attempt the token-authenticated browser handshake.
