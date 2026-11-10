# zoneinfo - Ecko Std Lib Package

IANA time zones for [Ecko](https://ecko.sh), written in Ecko. Read a timestamp
as the wall time in a named zone, build a timestamp from a wall time there, and
do calendar arithmetic that stays right across DST changes - a settlement
cut-off at 17:00 in New York, a merchant's local business day.

It works on the same Unix-millisecond UTC timestamps as `std.time` and the
[`datetime`](https://github.com/ecko-lang/datetime) package, and carries the
IANA time zone database with it, so every machine gives the same answer with no
system files.

## Install

```bash
ecko get github.com/ecko-lang/zoneinfo
```

Vendored under `./vendor/github.com/ecko-lang/zoneinfo/`, pinned by file-tree
hash in `ecko.sum`.

## Usage

```ecko
import std.time
import zoneinfo

t = time.parse("2026-07-01T12:00:00Z")

z = zoneinfo.in_zone(t, "Europe/Berlin")
z.hour                                  # 14
z.abbreviation                          # "CEST"
z.offset                                # 7200000 (ms, positive east of UTC)

zoneinfo.format(t, "%H:%M %Z %z", "Europe/Berlin")   # "14:00 CEST +0200"
zoneinfo.to_iso(t, "America/New_York")               # "2026-07-01T08:00:00.000-04:00"

# 17:00 in New York, as an instant.
cutoff = zoneinfo.from_components("America/New_York", 2026, 7, 1, 17)

# Tomorrow at the same wall time, whatever DST does overnight.
zoneinfo.add_days(t, 1, "Europe/Berlin")
```

## API

Every function takes the zone explicitly: any IANA name (`"Asia/Kolkata"`) or
alias (`"US/Eastern"`).

**Reading an instant**
- `in_zone(ms, zone)` - `{ year, month, day, hour, minute, second, millisecond, weekday, yearday }` (the fields of `datetime.components`, weekday Mon=0..Sun=6), plus `offset` (ms), `abbreviation`, `dst` and `zone`
- `offset(ms, zone)` - the offset from UTC at that instant, in ms

**Building an instant**
- `from_components(zone, y, mo, d, h?, mi?, s?, milli?)` - the instant a wall time in `zone` names

**Wall-clock arithmetic**
- `add_days(ms, n, zone)`, `add_months(ms, n, zone)`, `add_years(ms, n, zone)` - on the calendar in `zone`, keeping the wall time
- `start_of_day(ms, zone)` - local midnight on that day

**Formatting and parsing**
- `format(ms, fmt, zone)` - strftime-style, as the wall time in `zone`; `%Z` is the abbreviation, `%z` the offset (`+0200`), `%:z` with a colon (`+02:00`)
- `to_iso(ms, zone)` - `2026-07-01T14:00:00.000+02:00`, which `time.parse` reads back to the same instant
- `parse_format(s, fmt, zone)` - text as a wall time in `zone`

**The database**
- `zones()` - every zone and alias name, sorted
- `version()` - the IANA release, such as `"2026e"`

## Notes

**A zone is never part of a value.** A timestamp is always a UTC instant, so two
compare and subtract as plain `Int`s whatever zone they were built in. A zone is
how you read one, or build one from a wall time.

**Arithmetic keeps the wall time.** Berlin's 2026-03-29 is 23 hours long, so
`add_days(t, 1, "Europe/Berlin")` from noon on the 28th is noon on the 29th, 23
hours later. Adding `24 * 3600000` instead would land at 13:00.

**Times that do not exist, or happen twice.** When clocks jump forward, a wall
time in the gap moves forward by the length of the gap: 02:30 on a night that
goes from 02:00 to 03:00 is 03:30. When clocks go back, a repeated wall time is
its first occurrence, still on the earlier offset. Java and Python's `zoneinfo`
do the same. On a day whose midnight is skipped (Santiago's 2026-09-06),
`start_of_day` is the first moment of the day, 01:00.

**An unknown zone is an error**, `{ kind: "parse", format: "zone" }` naming it,
never a quiet fallback to UTC.

**The data.** The IANA time zone database is compiled into `tzdata.ecko`.
Instants past the last recorded change follow each zone's ongoing rule, so
dates far in the future work too. A new tzdata release means a new zoneinfo
release:

```bash
ecko tools/gen_tzdata.ecko /usr/share/zoneinfo tzdata.ecko
```

reads a machine's compiled zoneinfo (its `tzdata.zi` names the release) and
rewrites the file, one line per zone so a refresh reviews as a line diff.

**Cost.** Importing zoneinfo takes about 17 ms, most of it compiling the zone
code, which is why it is a package of its own rather than part of `datetime`. A
zone's first use takes about 1 ms; each lookup after that about 5 microseconds.

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

MIT - see [LICENSE](LICENSE).
