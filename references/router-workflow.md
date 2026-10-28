pyenv: cannot rehash: /Users/bytedance/.pyenv/shims isn't writable
# Router workflow and troubleshooting

Use this reference while carrying out the router configuration. Labels are based on GL.iNet 4.x and LuCI on OpenWrt 22.03; adapt to the installed firmware rather than assuming exact menu placement.

## 1. Establish the Tailscale boundary

In the GL.iNet panel, open **Applications / 应用程序 > Tailscale**.

1. Enable Tailscale.
2. Open the device-binding link and let the user authenticate to the intended tailnet.
3. Verify the page says the device is connected and record the current router virtual IPv4 address.
4. Leave these off for an app-only proxy:
   - custom exit node
   - run as exit node
   - advertise WAN subnet
   - advertise LAN subnet
   - IP masquerading

The remote user must install Tailscale and join the same tailnet. An invitation link grants tailnet membership; it is not the Tinyproxy URL.

If the binding-link request fails, confirm the router has Internet access, wait for Tailscale to finish starting, and retry. If an old corporate or unwanted account is bound, unbind it before authenticating the replacement account.

## 2. Install packages

Use **Applications / 应用程序 > Software Packages / 软件包**.

Install:

- `tinyproxy`
- `luci-app-tinyproxy`
- optionally `luci-i18n-tinyproxy-zh-cn` or another matching language package

Installing a LuCI language package may pull Tinyproxy and the LuCI application as dependencies. Verify installed packages by checking that the UI offers removal and that adequate free storage remains.

If LuCI is missing, open **System / 系统 > Advanced Settings / 高级设置** and install it. LuCI normally listens on the GL.iNet-specific local port shown by the panel. The LuCI username is commonly `root`; the user should enter the router administrator password themselves.

A post-install message about failing to signal `rpcd` does not by itself prove package installation failed. Check the package state and whether LuCI opens. A router restart may be needed before dynamic LuCI pages such as **Startup / 启动项** work.

## 3. Configure Tinyproxy

Open **Services / 服务 > Tinyproxy Server / Tinyproxy服务器 > Configuration / 配置**.

Recommended baseline:

| Setting | Value |
|---|---|
| Enable Tinyproxy | On |
| Listen port | `8888` or another chosen port |
| Listen address | Router Tailscale IP when supported; otherwise leave empty |
| Bind address | Empty unless a specific outbound interface is required |
| Allowed clients | `127.0.0.1` plus `100.64.0.0/10`, or exact peer IPs |
| Allowed CONNECT ports | `443`; retain `563` only if needed |
| X-Tinyproxy header | Off unless explicitly required |

Leaving `Listen` empty causes Tinyproxy to listen on local interfaces. This is acceptable only when its allow list excludes LAN/WAN clients and the firewall does not expose the port. Binding to the Tailscale IP is cleaner, but some router builds fail to launch Tinyproxy when it binds directly to a virtual interface.

Do not set the outbound **Bind address** to the Tailscale IP: the desired public egress is the router's normal WAN connection.

## 4. Check the firewall

Open **Network / 网络 > Firewall / 防火墙 > General Settings / 常规设置**.

Expected properties:

- WAN input is reject or drop.
- No WAN port-forward exists for the proxy port.
- A `tailscale0` zone or equivalent interface is present.
- Input from `tailscale0` is accepted, or a narrow traffic rule permits TCP to the proxy port on this device.

Do not change a whole zone to accept just to expose one service when a single traffic rule suffices.

## 5. Enable and start the service

After **Save & Apply**, open **System / 系统 > Startup / 启动项**.

Find `tinyproxy` and ensure it is **Enabled / 已启用**, then click **Restart / 重启** (or **Start / 启动** if it is stopped). This performs the equivalent of:

```sh
service tinyproxy enable
service tinyproxy restart
```

Check **Status / 状态 > Processes / 系统进程** for `/usr/bin/tinyproxy -c /var/etc/tinyproxy.conf -d`. The default configuration commonly launches one main process and several workers.

## 6. Test from the right place

First confirm the client appears in the same Tailscale admin network as the router. If the admin console says there is only one machine, the router is the only member and no client-side Tailscale test can succeed yet.

From a same-tailnet client:

```bash
tailscale ping <TAILSCALE_IP>
curl -x http://<TAILSCALE_IP>:8888 https://api.ipify.org
```

The second command should return the router site's public Internet address. A browser opening the proxy URL directly is not a reliable proxy test because an HTTP proxy expects proxy-form requests or CONNECT.

For Claude Code on macOS or Linux:

```bash
HTTP_PROXY=http://<TAILSCALE_IP>:8888 \
HTTPS_PROXY=http://<TAILSCALE_IP>:8888 \
claude
```

For PowerShell:

```powershell
$env:HTTP_PROXY = "http://<TAILSCALE_IP>:8888"
$env:HTTPS_PROXY = "http://<TAILSCALE_IP>:8888"
claude
```

These assignments affect only that process environment and its children. Avoid shell profile or system-wide configuration unless broader proxy inheritance is intended.

## 7. Diagnose failures in order

### Connection refused

1. Verify Tinyproxy is enabled in its configuration.
2. Explicitly restart it from **System > Startup**.
3. Check for Tinyproxy processes.
4. If direct binding to the Tailscale IP prevents startup, clear the Listen address and enforce the ACL and firewall boundary instead.

### Connection timeout

1. Verify the client is logged into the same tailnet.
2. Verify the router is connected and its Tailscale IP has not changed.
3. Check the `tailscale0` firewall zone or the narrow input rule.
4. Confirm Tinyproxy allows the client's Tailscale source address.

### LuCI Startup page never loads

Newly installed LuCI components may not have reloaded `rpcd`. Confirm configuration is saved, then restart the router only with the user's approval. After reboot, log in again and revisit **System > Startup**.

### Proxy works but other traffic also changes

Check the client for a selected exit node and for globally exported `HTTP_PROXY` or `HTTPS_PROXY` variables. The intended design uses no exit node and launches only the target application with inline environment variables.

## Authentication choices

The default design has no separate proxy password. Tailnet membership, Tailscale ACLs, and Tinyproxy's source allow list are the access controls. For tighter scope, prefer exact peer Tailscale IPs or Tailscale ACL grants. Add Tinyproxy BasicAuth only when the user needs an additional credential and the installed build exposes or supports the `BasicAuth` directive.
