<script lang="ts">
  import {
    activeProfile,
    discoverPortlessPorts,
    isPending,
    killPortProcess,
    portStatuses,
    savePortlessSettings,
    startPort,
    statusMessage,
    stopPort,
  } from "../stores/portManager";
  import type { PortlessGateway } from "../types";

  let baseLocalPort = 8100;
  let scheme: "http" | "https" = "https";
  let hostsText = "";
  let gateways: PortlessGateway[] = [];

  let portlessUrls = "";
  let discoveryPattern = "";
  let discoveryStartPort = 1300;
  let discoveryEndPort = 1400;
  let discovering = false;

  let prevBaseLocalPort = 8100;
  let prevScheme: "http" | "https" = "https";
  let prevHostsText = "";
  let prevGatewaysJson = "[]";

  $: statusByPort = new Map($portStatuses.map((status) => [status.port, status]));
  $: {
    const cfg = $activeProfile.portless;
    const nextHostsText = cfg.hosts.join("\n");
    const nextGatewaysJson = JSON.stringify(cfg.gateways);
    if (baseLocalPort === prevBaseLocalPort) baseLocalPort = cfg.base_local_port;
    if (scheme === prevScheme) scheme = cfg.scheme;
    if (hostsText === prevHostsText) hostsText = nextHostsText;
    if (JSON.stringify(gateways) === prevGatewaysJson) {
      gateways = cfg.gateways.map((gateway) => ({
        ...gateway,
        host: gateway.host ?? null,
      }));
    }
    prevBaseLocalPort = cfg.base_local_port;
    prevScheme = cfg.scheme;
    prevHostsText = nextHostsText;
    prevGatewaysJson = nextGatewaysJson;
  }

  $: hosts = hostsText
    .split(/\r?\n|,/)
    .map((host) => host.trim().replace(/^https?:\/\//, "").replace(/\/$/, ""))
    .filter(Boolean);

  function gatewayNameFromHost(host: string) {
    return host
      .replace(/\.localhost$/i, "")
      .split(".")
      .filter(Boolean)
      .join("-") || "portless";
  }

  function uniqueGatewayName(baseName: string, existing: PortlessGateway[]) {
    const used = new Set(existing.map((gateway) => gateway.name));
    if (!used.has(baseName)) return baseName;
    let index = 2;
    while (used.has(`${baseName}-${index}`)) index += 1;
    return `${baseName}-${index}`;
  }

  function parseQuickAdds(value: string) {
    return value
      .split(/\s+|,/)
      .map((item) => item.trim())
      .filter(Boolean)
      .map((item) => {
        const mapping = item.match(/^(\d{1,5}):(\d{1,5})$/);
        if (mapping) {
          const localPort = Number(mapping[1]);
          const remotePort = Number(mapping[2]);
          if (
            localPort < 1 ||
            localPort > 65535 ||
            remotePort < 1 ||
            remotePort > 65535
          ) {
            throw new Error("Mapping ports must be between 1 and 65535");
          }
          return {
            scheme,
            host: null,
            localPort,
            remotePort,
          };
        }

        const withScheme = /^[a-z]+:\/\//i.test(item) ? item : `https://${item}`;
        try {
          const parsed = new URL(withScheme);
          const parsedScheme = parsed.protocol.replace(":", "").toLowerCase();
          if (parsedScheme !== "http" && parsedScheme !== "https") {
            throw new Error("Unsupported scheme");
          }
          const remotePort = Number(parsed.port || (parsedScheme === "https" ? 443 : 80));
          if (!remotePort || remotePort < 1 || remotePort > 65535) {
            throw new Error("Invalid port");
          }
          return {
            scheme: parsedScheme as "http" | "https",
            host: parsed.hostname,
            localPort: remotePort,
            remotePort,
          };
        } catch {
          throw new Error(`Could not parse "${item}"`);
        }
      });
  }

  async function save(
    nextGateways = gateways,
    nextHosts = hosts,
    nextScheme: "http" | "https" = scheme
  ) {
    const base = Number(baseLocalPort);
    if (!base || base < 1 || base > 65535) {
      statusMessage.set("Base local port must be between 1 and 65535");
      return false;
    }
    const err = await savePortlessSettings(base, nextScheme, nextHosts, nextGateways);
    if (err) {
      statusMessage.set(err);
      return false;
    }
    return true;
  }

  async function addFromUrls() {
    let parsedUrls;
    try {
      parsedUrls = parseQuickAdds(portlessUrls);
    } catch (e) {
      statusMessage.set(String(e instanceof Error ? e.message : e));
      return;
    }
    if (parsedUrls.length === 0) {
      statusMessage.set("Paste one or more Portless URLs");
      return;
    }

    const nextHosts = Array.from(
      new Set([...hosts, ...parsedUrls.map((url) => url.host).filter((host): host is string => !!host)])
    );
    const firstScheme = parsedUrls[0].scheme;
    let next = [...gateways];
    for (const parsed of parsedUrls) {
      const localPort = parsed.localPort;
      const baseName = parsed.host
        ? gatewayNameFromHost(parsed.host)
        : `port-${parsed.remotePort}`;
      const sameMapping = next.find(
        (gateway) =>
          gateway.local_port === localPort &&
          gateway.remote_port === parsed.remotePort
      );
      if (sameMapping) {
        if (parsed.host && !sameMapping.host) sameMapping.host = parsed.host;
        continue;
      }
      if (next.some((gateway) => gateway.local_port === localPort)) {
        statusMessage.set(`Local port ${localPort} is already mapped`);
        return;
      }
      const existing = next.find(
        (gateway) =>
          gateway.remote_port === parsed.remotePort &&
          gateway.name === baseName
      );
      if (existing) continue;

      next = [
        ...next,
        {
          name: uniqueGatewayName(baseName, next),
          host: parsed.host,
          local_port: localPort,
          remote_port: parsed.remotePort,
        },
      ];
    }

    hostsText = nextHosts.join("\n");
    scheme = firstScheme;
    if (await save(next, nextHosts, firstScheme)) {
      const added = next.filter(
        (gateway) => !gateways.some((existing) => existing.local_port === gateway.local_port)
      );
      gateways = next;
      for (const gateway of added) {
        await startPort(gateway.local_port);
      }
      portlessUrls = "";
    }
  }

  async function discoverAndAdd() {
    const start = Number(discoveryStartPort);
    const end = Number(discoveryEndPort);
    if (!start || !end || start < 1 || end > 65535 || start > end) {
      statusMessage.set("Scan range must be between 1 and 65535");
      return;
    }

    discovering = true;
    statusMessage.set(`Scanning ${discoveryPattern} from ${start}-${end}...`);
    const result = await discoverPortlessPorts(discoveryPattern, start, end);
    discovering = false;

    if (typeof result === "string") {
      statusMessage.set(`Discovery failed: ${result}`);
      return;
    }
    if (result.ports.length === 0) {
      statusMessage.set("Discovery found no responding ports");
      return;
    }

    const nextHosts = Array.from(new Set([...hosts, result.host]));
    let next = [...gateways];
    for (const port of result.ports) {
      const existing = next.find(
        (gateway) => gateway.local_port === port && gateway.remote_port === port
      );
      if (existing) {
        if (!existing.host) existing.host = result.host;
        continue;
      }
      if (next.some((gateway) => gateway.local_port === port)) {
        continue;
      }
      next = [
        ...next,
        {
          name: uniqueGatewayName(`${gatewayNameFromHost(result.host)}-${port}`, next),
          host: result.host,
          local_port: port,
          remote_port: port,
        },
      ];
    }

    hostsText = nextHosts.join("\n");
    scheme = result.scheme;
    if (await save(next, nextHosts, result.scheme)) {
      const added = next.filter(
        (gateway) => !gateways.some((existing) => existing.local_port === gateway.local_port)
      );
      gateways = next;
      for (const gateway of added) {
        await startPort(gateway.local_port);
      }
      statusMessage.set(`Discovered ${result.ports.length} port(s)`);
    }
  }

  async function removeGateway(localPort: number) {
    await stopPort(localPort);
    const next = gateways.filter((gateway) => gateway.local_port !== localPort);
    if (await save(next)) gateways = next;
  }

  async function copyUrl(url: string) {
    await navigator.clipboard.writeText(url);
    statusMessage.set(`Copied ${url}`);
  }

  async function killGatewayProcess(localPort: number) {
    const err = await killPortProcess(localPort);
    if (!err) {
      await startPort(localPort);
    }
  }

  function ownerLabel(port: number) {
    const status = statusByPort.get(port);
    if (!status) return null;
    if (status.process_name) return `${status.process_name} (${status.owner_pid})`;
    if (status.owner_pid) return `PID ${status.owner_pid}`;
    return null;
  }

  function primaryUrl(gateway: PortlessGateway) {
    const host = gateway.host ?? hosts[0] ?? "localhost";
    return `${scheme}://${host}:${gateway.local_port}`;
  }

  function statusLabel(port: number) {
    if ($isPending) return "Pending";
    const status = statusByPort.get(port)?.status;
    if (status === "Forwarding") return "Forwarding";
    if (status === "PortInUse") return "Port In Use";
    return status ?? "Stopped";
  }
</script>

<div class="portless-section">
  <div class="section-heading">
    <h2>Portless Versions</h2>
    <button on:click={() => save()}>Save Settings</button>
  </div>

  <div class="settings-panel">
    <div class="field field-narrow">
      <label for="scheme">Scheme</label>
      <select id="scheme" bind:value={scheme}>
        <option value="https">HTTPS</option>
        <option value="http">HTTP</option>
      </select>
    </div>
    <div class="field hosts-field">
      <label for="hosts">Hosts</label>
      <input id="hosts" bind:value={hostsText} placeholder="app.localhost, api.localhost" />
    </div>
  </div>

  <div class="add-panel">
    <input
      bind:value={portlessUrls}
      placeholder="Paste URL(s) or mappings, e.g. https://ace-prompt.localhost:1355 or 1395:1935"
      on:keydown={(e) => e.key === "Enter" && addFromUrls()}
    />
    <button on:click={addFromUrls}>Add + Start</button>
  </div>

  <div class="discover-panel">
    <input
      bind:value={discoveryPattern}
      placeholder="Discovery pattern, e.g. https://ace-prompt.localhost:*"
      on:keydown={(e) => e.key === "Enter" && discoverAndAdd()}
    />
    <input
      bind:value={discoveryStartPort}
      aria-label="Scan start port"
      type="number"
      min="1"
      max="65535"
    />
    <input
      bind:value={discoveryEndPort}
      aria-label="Scan end port"
      type="number"
      min="1"
      max="65535"
    />
    <button on:click={discoverAndAdd} disabled={discovering}>
      {discovering ? "Scanning..." : "Discover + Add"}
    </button>
  </div>

  <div class="gateway-table">
    <div class="table-header">
      <span>Name</span>
      <span>Local</span>
      <span>Remote</span>
      <span>Status</span>
      <span>URL</span>
      <span></span>
    </div>
    <div class="table-body">
      {#each gateways as gateway (gateway.local_port)}
        {@const status = statusByPort.get(gateway.local_port)?.status}
        {@const url = primaryUrl(gateway)}
        {@const owner = ownerLabel(gateway.local_port)}
        <div class="gateway-row">
          <strong title={gateway.name}>{gateway.name}</strong>
          <span>:{gateway.local_port}</span>
          <span>:{gateway.remote_port}</span>
          <span class:status-up={status === "Forwarding"} title={owner ?? ""}>
            {owner ?? statusLabel(gateway.local_port)}
          </span>
          <a href={url} target="_blank" rel="noreferrer">{url}</a>
          <div class="row-actions">
            {#if status === "PortInUse"}
              <button class="danger" on:click={() => killGatewayProcess(gateway.local_port)}>
                Kill + Start
              </button>
            {:else if status === "Forwarding" || status === "RemoteDown" || status === "Reconnecting"}
              <button on:click={() => stopPort(gateway.local_port)}>Stop</button>
            {:else}
              <button on:click={() => startPort(gateway.local_port)}>Start</button>
            {/if}
            <button on:click={() => copyUrl(url)}>Copy</button>
            <button class="danger" on:click={() => removeGateway(gateway.local_port)}>Remove</button>
          </div>
        </div>
      {:else}
        <p class="empty">Paste a Portless URL above to add and start a version.</p>
      {/each}
    </div>
  </div>
</div>

<style>
  .portless-section {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
    gap: 8px;
  }

  .section-heading {
    display: flex;
    align-items: center;
    justify-content: space-between;
  }

  h2 {
    font-size: 11px;
    font-weight: 600;
    color: #6b7280;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    margin: 0;
  }

  .settings-panel,
  .add-panel,
  .discover-panel,
  .gateway-table {
    border: 1px solid #e5e7eb;
    border-radius: 8px;
    background: white;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.06), 0 1px 2px rgba(0, 0, 0, 0.04);
  }

  .settings-panel {
    display: grid;
    grid-template-columns: 95px 1fr;
    gap: 8px;
    padding: 12px;
  }

  .add-panel {
    display: grid;
    grid-template-columns: 1fr auto;
    gap: 8px;
    padding: 10px;
  }

  .discover-panel {
    display: grid;
    grid-template-columns: 1fr 76px 76px auto;
    gap: 8px;
    padding: 10px;
  }

  .field {
    display: flex;
    flex-direction: column;
    gap: 4px;
  }

  label {
    font-size: 11px;
    color: #6b7280;
    font-weight: 500;
    text-transform: uppercase;
    letter-spacing: 0.04em;
  }

  input,
  select {
    min-width: 0;
    padding: 7px 10px;
    border: 1px solid #e5e7eb;
    border-radius: 6px;
    background: white;
    font: inherit;
  }

  input:focus,
  select:focus {
    outline: none;
    border-color: #0078d4;
    box-shadow: 0 0 0 3px rgba(0, 120, 212, 0.1);
  }

  button {
    padding: 7px 10px;
    background: white;
    border: 1px solid #e5e7eb;
    border-radius: 6px;
    cursor: pointer;
    font-weight: 500;
    color: #374151;
    white-space: nowrap;
  }

  button:hover {
    background: #f0f7ff;
    border-color: #0078d4;
    color: #0078d4;
  }

  button.danger:hover {
    background: #fef2f2;
    border-color: #fca5a5;
    color: #dc2626;
  }

  .gateway-table {
    flex: 1;
    min-height: 90px;
    overflow: hidden;
    display: flex;
    flex-direction: column;
  }

  .table-header,
  .gateway-row {
    display: grid;
    grid-template-columns: 1fr 68px 76px 104px minmax(150px, 1.4fr) auto;
    gap: 8px;
    align-items: center;
    padding: 8px 10px;
  }

  .table-header {
    background: #f9fafb;
    border-bottom: 1px solid #e5e7eb;
    font-size: 11px;
    font-weight: 600;
    color: #6b7280;
    text-transform: uppercase;
    letter-spacing: 0.05em;
  }

  .table-body {
    flex: 1;
    overflow-y: auto;
  }

  .gateway-row {
    border-bottom: 1px solid #f3f4f6;
    font-size: 13px;
  }

  .gateway-row:last-child {
    border-bottom: 0;
  }

  strong,
  a {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  a {
    color: #0078d4;
  }

  .status-up {
    color: #16a34a;
  }

  .row-actions {
    display: flex;
    gap: 6px;
  }

  .empty {
    color: #9ca3af;
    font-size: 13px;
    padding: 24px;
    text-align: center;
  }
</style>
