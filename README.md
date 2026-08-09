# snowsys-monitor

Scheduled black-box monitoring of my public web surfaces: uptime, redirects,
DNS, certificate expiry, and the public Snowblog API/access-control contract.
The workflow runs 25 probes against endpoints that are already publicly
observable. A failing run emails the owner. No credentials, no deploy access —
probes only.
