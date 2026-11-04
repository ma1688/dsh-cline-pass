# dsh-cline-pass

[![npm](https://img.shields.io/npm/v/dsh-cline-pass)](https://www.npmjs.com/package/dsh-cline-pass)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

[中文](README.md) | English

Use [Cline Pass](https://cline.bot/cline-pass) subscription models directly in [DeepSeek Harness (dsh)](https://github.com/deepseek-ai), without running a separate proxy service.

## Features

- Streaming, tool calls, image input, and reasoning, depending on the model.
- Multiple accounts with single-account and round-robin modes.
- Discover, validate, pin, or exclude upstream channels, with failover before the first token.
- Region constraints (`inferenceRegion`) so a subscription model can leave its server-side virtual model config and actually honor the pinned channel.
- Web settings panel and optional `cline_pass_*` management tools.

## Quick start

Requires Node.js `>= 20.3.0`, dsh `>= 0.1.2-alpha.3 < 0.3.0`, and a valid Cline Pass API key.

```bash
dsh plugin --profile web add dsh-cline-pass
```

Restart dsh, open **Settings → Cline Pass** to save and test your API key, then select a Cline Pass model in the model picker. Keys are stored in the dsh credential store and displayed only in masked form.

Alternatively, configure the key through an environment variable:

```bash
export CLINE_PASS_API_KEY="your-cline-pass-api-key"
dsh --profile web
```

These commands use the `web` profile; replace the name if you use another profile.

## Configuration

Use the Web panel for everyday configuration. For file-based configuration, edit your profile's `$DSH_HOME/profiles/<name>/cordis.patch.yml`, for example:

```yaml
- id: cline-pass
  config:
    accountMode: roundrobin
    accounts:
      main:
        apiKeyEnv: CLINE_PASS_MAIN_KEY
      backup:
        apiKeyEnv: CLINE_PASS_BACKUP_KEY
    exposeTools: false
```

Each `apiKeyEnv` refers to a credential name or environment variable; configure a key for each account. Management tools are disabled by default. Set `exposeTools` to `true` and restart dsh to enable them.

The default gateway is `https://api.cline.bot/api/v1`. Each account allows up to `16` concurrent requests (`maxConcurrentRequests`; `0` removes the limit). See [`lib/index.js`](lib/index.js) for more options and [`lib/tools.js`](lib/tools.js) for management tools.

## Upstream channels

Expand a model row in the settings panel, click **Probe → Validate**, save your channel order or exclusions, then click **Test** to verify.

- `strict`: Restrict each attempt to its specified candidate channel.
- `preferred`: Send a preferred channel order to the gateway and try subsequent candidates when needed.

Validation sends real requests to each channel without changing saved rules. Failover for retryable errors takes place before the first output token.

### Region constraint (`inferenceRegion`)

Every `cline-pass/*` model is bound to a server-side virtual model config (VMC). That
path **drops** `providerOptions.gateway.only`, so pinning alone frequently has no
effect: the request still lands on the VMC's own private channel, and the client sees
no error at all.

`inferenceRegion` is the one field that evicts the VMC's provider from the plan: pin
that provider to a zone it does not serve, and the plan falls back to the model's
plain upstream registry, where `only` does bite.

```yaml
      cline-pass/deepseek-v4.1-flash-1017:
        upstreams: [deepseek]
        pinMode: strict
        inferenceRegion:
          scope: global
          providers:
            openai-compatible-private:
              scope: zone
              geoRegion: us
```

The management tool takes the same value as JSON text:

```
cline_pass_pin(model="cline-pass/xxx", upstreams=["deepseek"], inferenceRegion='{"scope":"global","providers":{"openai-compatible-private":{"scope":"zone","geoRegion":"us"}}}')
```

- Left unset, nothing changes and no extra field is sent.
- A malformed value (bad JSON, an array) is treated as "not configured" instead of failing the request.
- Confirm it took effect from the routing metadata: `finalProvider` should name your channel, and `provider_metadata.gateway.virtualModelConfig` should be empty.

## Development and contributing

The project uses JavaScript ES Modules with no build step. Install dependencies and run:

```bash
npm install
npm test
npm run test:client
npm run test:mount
```

Tests require compatible dsh packages; the mount test also requires a complete dsh installation. `npm run test:live`, `npm run test:live:image`, and `npm run test:live:reasoning` access the live gateway, require a valid key, and consume quota or incur charges.

[Issues](https://github.com/yhshzh/dsh-cline-pass/issues) and Pull Requests are welcome. Include versions, reproduction steps, and sanitized logs in bug reports, and run relevant tests before submitting code.

## Acknowledgements

Upstream channel and failover behavior was informed by the MIT-licensed [`cline-pass-switcher`](https://github.com/munmunjaklin458-afk/cline-pass-switcher).

## Community links

[LinuxDo](https://linux.do/) — A community for open source and technology, built on sincerity, kindness, unity, and professionalism.

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=yhshzh/dsh-cline-pass&type=Date)](https://www.star-history.com/#yhshzh/dsh-cline-pass&Date)

## License

[MIT](LICENSE)
