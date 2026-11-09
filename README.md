# firewalla-msp-broker

Fine-grained access control for the Firewalla MSP API. One upstream token in, many scoped keys out: per-box (or box-group) scope, per-resource read/write permissions, an audit trail with recorded previous values and reversion, key expiry, IP allowlists and rotation windows. Built for AI agents and scripts that should not hold a Full Access token.

The binary, Docker image and command are called `mspbroker`.

> **Disclaimer:** this is an independent community project. It is not affiliated with, endorsed by or supported by Firewalla Inc. "Firewalla" is a trademark of Firewalla Inc., used here only to describe what the software works with.

Status: pre-alpha, under construction. See [docs/SPEC.md](docs/SPEC.md).

## License

GNU Affero General Public License v3.0 (AGPL-3.0). See [LICENSE](LICENSE).
