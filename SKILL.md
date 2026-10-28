---
name: glinet-tailscale-tinyproxy
description: Configure and troubleshoot a GL.iNet or OpenWrt router as a Tailscale-only HTTP/HTTPS proxy using Tinyproxy, including package installation, LuCI setup, access restrictions, service startup, and per-application client commands. Use when the user wants selected applications to use a remote router's public IP without enabling a full-device exit node or exposing a proxy to WAN.
---

# GL.iNet Tailscale Tinyproxy

Build an explicit HTTP/HTTPS proxy reachable through Tailscale while leaving ordinary client traffic on its local network.

## Operating invariants

- Do not enable a Tailscale exit node, WAN/LAN subnet advertising, IP masquerading, or WAN port forwarding unless the user explicitly asks for those behaviors.
- Never expose Tinyproxy directly to the public Internet. Keep WAN input rejected or dropped and create no WAN forwarding rule for the proxy port.
- Treat the Tailscale device identity and tailnet membership as the default authentication boundary. Add Tinyproxy BasicAuth only when the user explicitly wants a separate proxy username and password.
- Keep device-specific values variable. Discover the router's current Tailscale IPv4 address instead of reusing an example address.
- Have the user enter router passwords, account credentials, and MFA themselves. Do not request that they paste secrets into chat.

## Workflow

1. Inspect the router model, firmware, underlying OpenWrt version, free storage, current Tailscale state, and firewall zones. Do not flash firmware as part of this workflow.
2. Configure Tailscale first. Bind the router to the intended personal tailnet and verify that the router appears connected with a `100.x.y.z` address. Keep exit-node and subnet-routing features off for an app-scoped proxy.
3. Install `tinyproxy` and `luci-app-tinyproxy`; add the matching LuCI language package only when useful. Install LuCI from the GL.iNet advanced settings page if it is unavailable.
4. Configure Tinyproxy with a user-selected port, normally `8888`. Allow localhost and either the Tailscale CGNAT range `100.64.0.0/10` or a tighter list of known peer addresses. Allow CONNECT to `443`; keep other CONNECT ports only when required.
5. Prefer binding Tinyproxy to the router's Tailscale IP. If the daemon fails to start on that virtual address, leave `Listen` unset so it binds locally, then rely on both the Tinyproxy allow list and the firewall boundary. Never compensate by opening the WAN port.
6. Confirm that the `tailscale0` firewall zone accepts input. If it does not, create the narrowest rule possible: source interface or zone `tailscale0`, destination this device, TCP, proxy port only. Keep WAN input blocked.
7. Save the configuration, then explicitly enable and restart Tinyproxy under LuCI **System > Startup**. A LuCI “Save & Apply” may write UCI configuration without starting the init service.
8. Verify a Tinyproxy master process and workers are running. Test from a different device that is actually in the same tailnet; do not interpret a timeout from a non-member device as a router-side failure.
9. Give the client an inline per-process proxy command. Avoid global environment variables unless the user intentionally wants other applications to inherit the proxy.

For detailed Chinese/English UI labels, verification commands, and failure recovery, read [references/router-workflow.md](references/router-workflow.md).

## Client handoff

Use placeholders in commands and replace them with observed values:

```bash
HTTP_PROXY=http://<TAILSCALE_IP>:<PORT> \
HTTPS_PROXY=http://<TAILSCALE_IP>:<PORT> \
claude
```

Verify independently:

```bash
tailscale ping <TAILSCALE_IP>
curl -x http://<TAILSCALE_IP>:<PORT> https://api.ipify.org
```

Explain that the proxy address is only the first hop: the selected application sends its HTTP/HTTPS requests to Tinyproxy, and the router forwards them using its own Internet connection. Other applications remain local unless they inherit the proxy variables. Tailscale itself only routes tailnet traffic unless an exit node is selected.

## Completion criteria

- Router is connected to the intended tailnet and has a current Tailscale IP.
- Tinyproxy is enabled at boot and has live processes.
- Tinyproxy ACL includes the intended Tailscale clients and excludes ordinary LAN/WAN clients.
- WAN has no proxy port exposure.
- A same-tailnet client can make an HTTPS request through the proxy, or the lack of a same-tailnet test device is reported explicitly.
- The user receives the exact proxy URL and an app-scoped invocation command.
