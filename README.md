# snowsys-monitor

Scheduled black-box monitoring of my public web surfaces: uptime, redirects,
DNS, certificate expiry, and the public Snowblog API/access-control contract.
Every probe lives in
[`.github/workflows/external-acceptance.yml`](.github/workflows/external-acceptance.yml)
— that file is the probe inventory. All targets are already publicly
observable. A failing run emails the owner. No credentials, no deploy access —
probes only.
