# zoneinfo

IANA time zones for [Ecko](https://ecko.sh): read a timestamp as the wall time
in a named zone, build one from a wall time there, and do calendar arithmetic
that stays right across DST changes. Pure computation - no capabilities.

It works on the same Unix-millisecond UTC timestamps as `std.time` and the
[`datetime`](https://github.com/ecko-lang/datetime) package, and carries the
IANA time zone database with it, so every machine gives the same answer with no
system files.

## Install

```bash
ecko get github.com/ecko-lang/zoneinfo
```

```ecko
import zoneinfo
```

## Usage

```ecko
import std.time
import zoneinfo

t = time.parse("2026-07-01T12:00:00Z")

z = zoneinfo.in_zone(t, "Europe/Berlin")
print(z.hour, z.abbreviation, z.offset)                 # 14 CEST 7200000
print(zoneinfo.format(t, "%H:%M %Z %z", "Europe/Berlin"))  # 14:00 CEST +0200
print(zoneinfo.to_iso(t, "America/New_York"))           # 2026-07-01T08:00:00.000-04:00

# A settlement cut-off at 17:00 in New York, as an instant.
cutoff = zoneinfo.from_components("America/New_York", 2026, 7, 1, 17)
print(time.format(cutoff, "%Y-%m-%dT%H:%MZ"))           # 2026-07-01T21:00Z

# Tomorrow at the same wall time: Berlin's 29 March is 23 hours long.
noon = zoneinfo.from_components("Europe/Berlin", 2026, 3, 28, 12)
next = zoneinfo.add_days(noon, 1, "Europe/Berlin")
print((next - noon) // 3600000)                         # 23

# Text that carries its own offset ignores the zone argument.
got = zoneinfo.parse_format("29/07/2026 17:00 +0200", "%d/%m/%Y %H:%M %z", "America/New_York")
print(time.format(got, "%Y-%m-%dT%H:%MZ"))              # 2026-07-29T15:00Z

# A wall time that cannot exist is refused, never rolled over.
e = try { zoneinfo.from_components("UTC", 2026, 2, 30) } catch (err) { err }
print(e.kind, e.reason)                                 # zoneinfo out_of_range
```

Every function takes the zone explicitly: any IANA name (`"Asia/Kolkata"`) or
alias (`"US/Eastern"`).

## API

### Reading an instant

| function | returns |
|---|---|
| `in_zone(ms, zone)` | `{ year, month, day, hour, minute, second, millisecond, weekday, yearday, offset, abbreviation, dst, zone }`: the wall time `ms` shows in `zone` (weekday Mon=0..Sun=6, offset in ms) |
| `offset(ms, zone)` | the zone's offset from UTC at that instant, in ms, positive east of Greenwich |

### Building an instant

| function | returns |
|---|---|
| `from_components(zone, y, mo, d, h = 0, mi = 0, s = 0, milli = 0)` | the instant (Unix ms) a wall time in `zone` names; every component is range-checked |

### Wall-clock arithmetic

| function | returns |
|---|---|
| `add_days(ms, n, zone)` | the instant `n` calendar days later in `zone`, at the same wall time |
| `add_months(ms, n, zone)` | the instant `n` months later, the day clamped to the month's length |
| `add_years(ms, n, zone)` | the instant `n` years later; 29 February becomes 28 February |
| `start_of_day(ms, zone)` | local midnight on that day, or the first moment of a day that skips it |

### Formatting and parsing

| function | returns |
|---|---|
| `format(ms, fmt, zone)` | the wall time in `zone` as text, strftime-style; `%Z` is the abbreviation, `%z` the offset (`+0200`), `%:z` with a colon (`+02:00`) |
| `to_iso(ms, zone)` | ISO-8601 with the offset, such as `2026-07-01T14:00:00.000+02:00` |
| `parse_format(s, fmt, zone)` | the instant text `s` names, read with `fmt` as a wall time in `zone`, or at its own offset when `fmt` has `%z`/`%:z` |

### The database

| function | returns |
|---|---|
| `zones()` | every zone and alias name, sorted |
| `version()` | the IANA release the data comes from, such as `"2026e"` |

## Notes

### Errors

| kind | reason | when |
|---|---|---|
| `zoneinfo` | `unknown_zone` | the zone name is not in the database (the error carries `zone`) |
| `zoneinfo` | `out_of_range` | a `from_components` component is outside its range: month 13, 30 February, hour 24 (carries `field` and `value`) |
| `zoneinfo` | `no_match` | `parse_format` text does not match the pattern (carries `text` and `format`) |
| `zoneinfo` | `ambiguous_offset` | `parse_format` text has more than one `+hhmm` run that reads as the offset, giving different instants |
| `bug` | `bad_format` | `format`/`parse_format` got a pattern chrono cannot use, such as a trailing `%` |
| `bug` | `bad_argument` | an argument has the wrong type: `ms`, `n` or a component not an Int, `zone` or `fmt` not a String |

### A zone is never part of a value

A timestamp is always a UTC instant, so two compare and subtract as plain
`Int`s whatever zone they were built in. A zone is how you read one, or build
one from a wall time.

### Arithmetic keeps the wall time

Berlin's 2026-03-29 is 23 hours long, so `add_days(t, 1, "Europe/Berlin")`
from noon on the 28th is noon on the 29th, 23 hours later. Adding
`24 * 3600000` instead would land at 13:00.

### Times that do not exist, or happen twice

When clocks jump forward, a wall time in the gap moves forward by the length of
the gap: 02:30 on a night that goes from 02:00 to 03:00 is 03:30. When clocks go
back, a repeated wall time is its first occurrence, still on the earlier
offset. Java and Python's `zoneinfo` do the same. On a day whose midnight is
skipped (Santiago's 2026-09-06), `start_of_day` is the first moment of the
day, 01:00.

### Offsets with seconds

Before standard time, zones kept local mean time, whose offset has seconds
(Amsterdam was +00:19:32 until 1937). `format` and `to_iso` write such an
offset in full, `+001932` and `+00:19:32`, as Python does. RFC 3339 has no
form for it, so `time.parse` refuses that `to_iso` string rather than reading a
different instant; every offset from the 20th century on reads back exactly.

### Parsing an offset

`parse_format` reads the `%z`/`%:z` field itself: it accepts `+hhmm` and
`+hh:mm`, and `%%z` in a pattern is the literal text `%z`. The pattern still
needs a minute field next to an hour: `time.parse_format`, which reads the
rest, ignores an `%H` with no `%M`.

### The data

The IANA time zone database is compiled into `tzdata.ecko`. Instants past the
last recorded change follow each zone's ongoing rule, so dates far in the
future work too. A new tzdata release means a new zoneinfo release:

```bash
ecko tools/gen_tzdata.ecko /usr/share/zoneinfo tzdata.ecko
```

reads a machine's compiled zoneinfo (its `tzdata.zi` names the release) and
rewrites the file, one line per zone so a refresh reviews as a line diff.

### Cost

Importing zoneinfo takes about 17 ms, most of it compiling the zone code, which
is why it is a package of its own rather than part of `datetime`. A zone's
first use takes about 1 ms; each lookup after that about 5 microseconds.

## Testing

```bash
ecko test
```

Offline and deterministic. The tests check 254 instants and 14 wall times
against values taken from Python's `zoneinfo`, across northern and southern
DST, a 30-minute DST (Lord Howe), negative DST (Dublin), rules that change at
24:00 (Santiago) and -1:00 (Nuuk), a 45-minute offset (Chatham), Casablanca,
and dates after 2037.

## License

[MIT](LICENSE)
