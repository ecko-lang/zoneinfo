# Changelog

## 0.2.0

### Fixed

- `parse_format` with `%z` or `%:z` now reads the offset in the text. It used
  to return the wall time as if it were UTC, so `"17:00 +0200"` came back as
  17:00Z instead of 15:00Z.
- `parse_format` no longer mistakes an escaped `%%z` for an offset field; the
  text is read as a wall time in the zone you pass.
- `from_components` refuses a wall time that cannot exist (month 13, 30
  February, hour 24, minute 60, a negative second) instead of rolling it over
  into a different date.
- `format` and `to_iso` write an offset that has seconds (the local mean time
  of old dates, such as Amsterdam's +00:19:32 before 1937) in full instead of
  dropping the seconds, which named a different instant.
- A wrong-typed argument (`in_zone("a", "UTC")`, a Float day count) is refused
  at the function you called, with a message naming it, instead of failing
  somewhere inside the package.
- A bad pattern in `format` or `parse_format` is reported as a `bug` naming
  the zoneinfo function; text that does not match a pattern is a `zoneinfo`
  error with reason `no_match`.

### Breaking

- An unknown zone name now throws kind `zoneinfo` with reason `unknown_zone`
  (and the name in `zone`), not kind `parse` with `format: "zone"`.
  Migration: match `get(e, "kind") == "zoneinfo"` and `get(e, "reason") ==
  "unknown_zone"` where you matched `"parse"`.
- `parse_format` throws kind `zoneinfo`, reason `no_match`, where it threw
  `time.parse_format`'s `parse` error.
  Migration: catch `"zoneinfo"` instead of `"parse"`.
