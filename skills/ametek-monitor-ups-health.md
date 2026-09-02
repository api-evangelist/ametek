---
name: ametek-monitor-ups-health
description: >-
  Poll an AMETEK Powervar UPS through its iSite PRO network management adapter — read live
  electrical and battery state, list active alarms with severity, and identify the adapter — using
  only the three operations AMETEK Powervar publishes.
api: openapi/ametek-powervar-isite-pro-openapi.yml
operations:
  - getWhoAreYou
  - getUpsStatus
  - getUpsAlarms
generated: '2026-09-02'
method: generated
source: openapi/ametek-powervar-isite-pro-openapi.yml
---

# Monitor an AMETEK Powervar UPS

This API is **read-only** and **device-local**. There is no AMETEK cloud endpoint. The host you
call is the customer's own iSite PRO adapter, on their own network.

## Before you start

1. **Find the adapter.** The adapter advertises itself over mDNS; AMETEK Powervar states mDNS is
   supported for dynamic discovery but is not part of the API. If you already know the address,
   skip discovery. Hostnames take the form `isitepro-<serial>`.
2. **Set the base URL** to `http://{device_host}:{port}/api/v1`. HTTP is the factory default;
   HTTPS is supported and should be preferred where the adapter is configured for it. The
   adapter's actual listener port is reported by `getWhoAreYou` in `httpd.port`.
3. **Authenticate on every call.** Authentication is required for all API features. Use either
   HTTP Basic with the adapter's local administrative credentials, or a web session token in the
   `x-auth-token` header. Credentials belong to the device, not to an AMETEK account.

## Step 1 — confirm what you are talking to (`getWhoAreYou`)

`GET /whoAreYou`

Confirm `manufacturer` is `AMETEK` and `model` is `iSitePro` before trusting anything else.
Record `serial` (the **adapter** serial), `firmware`, and `httpd.ssl`. If `httpd.ssl` is `false`
you are sending Basic credentials in the clear on the customer's LAN — say so rather than
proceeding silently.

## Step 2 — read the UPS state (`getUpsStatus`)

`GET /UpsStatus`

Takes no parameters. The fields that matter for a health verdict:

- `BatteryStatus` — health summary, e.g. `Normal`
- `EstimatedChargeRemaining` — percent
- `EstimatedMinutesRemaining` — runtime left on battery
- `OutputSource` — where the load is being fed from
- `OutputPercentLoad` — headroom
- `BatteryTemperature`, `BatteryVoltage`, `InputVoltage`, `OutputVoltage`
- `BatteryReplaceDate` — service planning

Two traps in AMETEK Powervar's own field set:

- `UpsStatus.SerialNumber` is the **UPS** serial. `WhoAreYou.serial` is the **adapter** serial.
  They are different values; do not conflate them when keying records.
- Date formats are not consistent inside one response body. `ManufactureDate` is published as
  `YYYY-MM-DD`, `BatteryReplaceDate` as `MM/DD/YYYY`. Parse them separately.

## Step 3 — read active alarms (`getUpsAlarms`)

`GET /UpsAlarms`

Returns an array. An empty array means no alarms are active — that is the healthy case, not an
error. Each entry carries `Timestamp` (RFC 3339 UTC), `Name`, `Severity` and `Description`.

Triage on `Severity`, which is fixed per alarm name:

- **3 — fault / service required.** Escalate to a human. Includes `BatteryDegraded`,
  `ChargerFailed`, `FanFailure`, `OverTemperature`, `ReplaceBattery`, `OutputOverload`,
  `LostComm`, `BackfeedRelayFailure`, `BatteryFuseBlown`, `DepletedBattery`,
  `DiagnosticsTestFailed`, `FuseFailure`, `GeneralFault`, `ServiceRequired`.
- **2 — warning.** Includes `OnBattery`, `LowBattery`, `OnBypass`, `BypassBad`, `OutputBad`,
  `InputOutOfTolerance`, `PowerMarginExceeded`, `ShutdownImminent`, `SiteWiringFault`,
  `GeneralWarning`.
- **1 — informational / state.** Includes `BatteryCharging`, `DiagnosticsTestInProgress`,
  `OutputOffAsRequested`, `UpsOffAsRequested`, `UpsOutputOff`, `UpsSystemOff`, `ShutdownPending`,
  `SystemRestartPending`, `PduSocketOff1..3`, `PduSocketOffPending1..3`.

`ShutdownImminent` is severity 2 but is the one warning that should be treated as urgent — it
means the load is about to drop.

The complete 38-value list is the `AlarmName` enum in the spec. Treat any name you do not
recognise as unknown and surface it verbatim; do not map it onto a similar-sounding one.

## Rules

- **Never attempt a write.** No POST, PUT, PATCH or DELETE is published. AMETEK Powervar's alarm
  catalogue names control actions (`UpsOffAsRequested`, `PduSocketOff1`, `SystemRestartPending`)
  but those are *reported states*, not operations this API exposes. Do not construct a control
  request by analogy — you would be inventing an endpoint against live power infrastructure.
- **No error catalogue is published.** Only 200 responses are documented. Handle any non-200 by
  surfacing the raw status and body rather than interpreting it, and do not assume a shape.
- **Tolerate unknown fields.** AMETEK Powervar states that undocumented key/value pairs are for
  internal use and must not be modified by any party unless AMETEK Powervar instructs otherwise.
  Read past them; never write them.
- **No rate limits are published.** Poll conservatively — this is an embedded management card on
  power equipment, not a cloud API. Treat a slow or refused response as a network condition, not
  as throttling.
- **Version is in the path.** `v1` is the only protocol version defined. Breaking changes iterate
  the URI version and are announced in firmware upgrade notes, and multiple versions may be live
  across a fleet at once — pin `v1` explicitly and re-check the adapter's `firmware` value rather
  than assuming a fleet is uniform.
