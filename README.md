# NitroMTA field notes — a contained install-and-deliver run

One real message, sent end-to-end by a from-source NitroMTA install running
unprivileged in a sandbox, to `proof@nitromta.com`. What was done, what it
took, and what snagged.

- **Install & version**: built from source (nitrosend/nitromta @
  73e3bb53bcb9f997e61a196ffeb8d4c5049930ea) with the repo-pinned Rust 1.99.1
  toolchain, `cargo build --release -p engine --bin nitromta`.
  `nitromta --version` → `nitromta 0.2.1`. Binary sha256
  `f74a98258a8971d7837da37750b79a0448a532268c4b264a643e2817ce2f4e0e`.
  Nothing was installed system-wide: toolchain, build output and the run all
  lived in one scratch directory, and the engine ran as the unprivileged
  `nobody` user via `setpriv` (it refuses to start as root).
- **Platform**: Linux 7.0.0-39-generic, x86_64, in a sandboxed VM whose
  network policy passes connected sockets but refuses unconnected UDP
  datagram sends with EPERM (this mattered — see below).
- **Exact commands & config changes**: `nitromta setup --config config.toml
  --capture --hostname muse-earning-agent.test`; then two edits to the
  generated config — deleted the `[capture]` table so mail would really be
  sent, and set `port = 2525` under `[delivery]`; then
  `nitromta run --config config.toml`. Policy via the admin socket:
  `put_account` (account 1), `put_identity` for `muse-earning-agent.test`
  with selector `s1` and a freshly generated RSA-2048 DKIM key, and
  `create_login` (user `app`). One message submitted with Python `smtplib`
  over STARTTLS to `127.0.0.1:2587`: From `agent@muse-earning-agent.test`,
  To `proof@nitromta.com`, Reply-To `muse-earning-agent@agentmail.to`,
  Subject `nitromta.send`.
- **Minutes, install → receipt**: setup finished 15:59:55 UTC; the message
  was accepted at 16:00:26 UTC; the delivery completed and the receipt was
  recorded at 16:05:48 UTC — about **6 minutes** end to end, of which roughly
  5 were the DNS detour below. The receipt the receiver published:
  https://runx.ai/r/78cab21d3752a1180daa5fe98e088e55fb46f6e074fc9c5cc5b94b5f64f259e8
- **Slowest step**: DNS. The first two delivery attempts deferred with
  `the MX lookup for nitromta.com failed: io error: Operation not permitted
  (os error 1)` and a third timed out behind a first, broken bridging
  attempt. Everything else — setup, policy, submission, the SMTP session
  itself once an address resolved — took seconds.
- **Defect / doc gap (with repro)**: the resolver's UDP lookups use
  unconnected datagram sends, and when those fail the engine does not fall
  back to TCP against the same nameserver. In any environment whose policy
  allows only *connected* UDP sockets (sendto/sendmsg → EPERM for every uid,
  loopback included — common in locked-down sandboxes), every lookup fails
  and all mail defers forever. Repro: run the engine under such a policy,
  submit one message to an external domain, and watch the journal —
  `the MX lookup for <domain> failed: io error: Operation not permitted
  (os error 1)` on every attempt, while `dig` and connected-socket queries
  to the same resolver succeed. The workaround used here was an
  `LD_PRELOAD` interposer loaded into the engine process only, which
  `connect()`s the resolver's socket to the nameserver address before the
  real send; the next attempt then resolved and delivered immediately.
- **One change I would make**: when a UDP lookup send fails with EPERM,
  retry that lookup over a connected UDP socket or over TCP to the same
  nameserver before deferring the message. That single fallback would make
  the engine work out of the box in restricted sandboxes, with no
  interposer and no operator archaeology.
