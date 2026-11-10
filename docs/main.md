# zoneinfo

## `zones()`

Every zone and alias name the database knows, sorted: "Africa/Abidjan"
through "Zulu".

## `version()`

The IANA time zone database release the zones come from, such as "2026e".

## `offset(ms, zone)`

`zone`'s offset from UTC at instant `ms`, in milliseconds: positive east of
Greenwich, so Berlin in summer is 7200000. Raises `{ kind: "parse", format:
"zone" }` for a name the database does not know, as every function here does.

## `in_zone(ms, zone)`

The wall time instant `ms` shows in `zone`: `{ year, month, day, hour,
minute, second, millisecond, weekday, yearday }` (the fields of
`datetime.components`, weekday Mon=0..Sun=6), plus `offset` (ms),
`abbreviation` ("CEST"), `dst` and `zone`. Any IANA name or alias works,
such as "Europe/Berlin" or "US/Eastern".

```ecko
in_zone(time.now(), "Asia/Tokyo").hour
```

## `from_components(zone, y, mo, d, h = 0, mi = 0, s = 0, milli = 0)`

The instant a wall time in `zone` names, as Unix ms.

A time a DST change skips moves forward by the length of the gap: 02:30 on a
night that jumps from 02:00 to 03:00 is 03:30. A time it repeats is the
first occurrence, the one still on the earlier offset.

```ecko
from_components("America/New_York", 2026, 7, 16, 17)   # 17:00 in New York
```

## `add_days(ms, n, zone)`

The instant `n` calendar days later in `zone`, at the same wall time. A day
across a DST change is 23 or 25 hours long. `n` may be negative.

## `add_months(ms, n, zone)`

The instant `n` months later in `zone`, at the same wall time, clamping the
day to the target month's length (Jan 31 + 1 month is Feb 28 or 29).

## `add_years(ms, n, zone)`

The instant `n` years later in `zone`, at the same wall time. 29 February
plus one year is 28 February.

## `start_of_day(ms, zone)`

Midnight in `zone` on the day instant `ms` falls on there. On a day a DST
change skips midnight, the first moment of the day.

## `format(ms, fmt, zone)`

Format instant `ms` as the wall time in `zone`, with a strftime-style
pattern. `%Z` writes the zone's abbreviation ("CEST"), `%z` its offset
("+0200") and `%:z` the offset with a colon ("+02:00").

```ecko
format(time.now(), "%Y-%m-%d %H:%M %Z", "Europe/Berlin")
```

## `to_iso(ms, zone)`

ISO-8601 for the wall time in `zone`, with its offset:
`2026-07-29T15:45:00.000+02:00`. `time.parse` reads it back to the same
instant.

## `parse_format(s, fmt, zone)`

Parse text `s` with a strftime-style pattern as a wall time in `zone`, read
the way `from_components` reads one. A pattern with `%z` carries its own
offset, so `zone` does not apply to it.

```ecko
parse_format("29/07/2026 17:00", "%d/%m/%Y %H:%M", "America/New_York")
```
