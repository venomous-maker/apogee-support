# apogee-support

The foundations of [Apogee](https://github.com/venomous-maker): dates and
times with real time zones (`Moment`, Carbon's API and more), intervals and
periods, typed collections, dotted-path data helpers, and text and number
helpers (Laravel's `Str` and `Number`). No dependencies.

```grav
import "std"
import "apogee-support"

let due = Moment.parse("next friday 17:00", "Africa/Nairobi")
due.diffForHumans()                                   // "in 6 days"
due.addMonths(1).startOfMonth().format("l, jS F Y")    // "Tuesday, 1st December 2026"

collect(orders)
    .where("status", "paid")
    .sortByDesc(fn(o: Order) -> double { return o.total })
    .take(5)
    .pluckText("number")
    .join(", ", " and ")

Text.plural("category")                               // "categories"
Text.slug("Héllo Wörld!")                             // "hello-world"
Number.fileSize(3_500_000)                            // "3.34 MB"
```

## Time zones

`TimeZone.of("America/New_York")` reads the system's zone file
(`/usr/share/zoneinfo`, or `$ZONEINFO`) once and keeps it. For years past the
file's table it uses the POSIX rule at the file's end, so 2100 still has
daylight saving time. It never touches the C library's process-wide `TZ`, so
threads can each use their own zone.

- **Names:** `"UTC"`, `"Z"`, offsets such as `"+05:30"` or `"-0800"`, or any
  zone name. `TimeZone.local()` is the system's zone (from `TZ`,
  `/etc/timezone`, or where `/etc/localtime` links to).
- **Lookups:** `offsetAt(t)`, `abbreviationAt(t)` (`"EAT"`, `"EDT"`),
  `isDstAt(t)` and `transitions(from, to)`.
- **Wall-clock times:** `toUnix(localSeconds)` turns one into a moment. A time
  skipped by a forward jump moves past the gap (02:30 becomes 03:30 EDT, as
  PHP does). A time that happens twice is the first one.
- **Calendar math:** `Civil` converts dates to day numbers and back
  (proleptic Gregorian, any year).

## Moment

A moment in time in a zone, to the microsecond. Moments are **immutable**:
`addDays` and every other change return a new one. A moment shared between
requests or threads never changes under anyone, which is CarbonImmutable's
behaviour. The year, month, day, time and offset are worked out once, when
the moment is made.

### Making and parsing

- **Making:**
  - `now(tz)`, `today`, `tomorrow`, `yesterday`
  - `create(y, m, d, h, i, s, tz)` lets fields roll over: month 13 is the next
    January. `createSafe` throws for an out-of-range field instead.
  - `createFromDate`, `createFromTime`
  - `fromUnix`, `fromUnixMs`, `fromUnixMicros`, `fromDateTime`
- **Parsing with `parse(text, tz)`:**
  - ISO 8601: `"2026-11-14"`, `"2026-11-14 09:30"`,
    `"2026-11-14T09:30:15.25+03:00"`, or a zone name after the time.
  - `"@1700000000"` (unix seconds).
  - Relative English:
    - `"now"`, `"today"`, `"tomorrow 9am"`, `"yesterday 11:15pm"`, `"noon"`
    - `"+2 weeks 3 days"`, `"3 days ago"`, `"a week ago"`
    - `"next friday 17:00"`, `"last monday"`, `"this friday"`, `"monday next week"`
    - `"first day of next month"`, `"last day of this month"`
    - `"March 5 2026"`, `"5th march"`
  - `tryParse` returns null instead of throwing.
- **Formats:** `createFromFormat("d/m/Y H:i", text, tz)` reads PHP's format
  letters. Fields the format doesn't give come from now. A format starting
  with `!` starts from 1970-01-01 00:00 instead, and `|` resets the fields
  that follow it.
- **Frozen clock for tests:**
  - `Moment.setTestNow(m)`, `freeze()`
  - `travel(5, "minutes")`, `travelTo(m)`, `travelBack()`
  - These also apply to `now()` and to relative parsing.

### Reading fields

- **Date and time:** `year`, `month`, `day`, `hour`, `minute`, `second`,
  `micro`, `milli`.
- **Weekdays:** `dayOfWeek` (0 is Sunday) and `isoWeekday` (1 is Monday).
- **Day and week numbers:** `dayOfYear`, `isoWeek` and `isoWeekYear`.
  `weekOfMonth` counts days 1-7 as week 1, as Carbon does; `weekNumberInMonth`
  gives the calendar row.
- **Lengths:** `quarter`, `daysInMonth`, `daysInYear`, `weeksInYear`,
  `isLeapYear`.
- **Zone:** `offset` (seconds), `offsetHours`, `tzName`, `tzAbbreviation`,
  `isDst`, `isUtc`.
- **Timestamps:** `unix`, `unixMs`, `unixMicros`.
- **Any unit by name:** `get("quarter")`.

### Changing

- **Setting fields:** `setDate`, `setTime`, `setTimeFromTimeString("14:30")`,
  `setYear` … `setMicro`, and `set("hour", 9)`.
- **Zones:** `tz("Europe/London")` (or `setTimezone`, `utc()`, `toLocal()`)
  keeps the same moment. `shiftTimezone` keeps the same wall-clock time
  instead.
- **Real time vs calendar time:** seconds, minutes and hours are real time.
  Days and longer are calendar time: a day across a DST change is 23 or 25
  hours, and 09:00 stays 09:00.
- **Adding:**
  - `addSeconds` … `addYears`, `addDecades` and `addCenturies`, with a `sub*`
    for each.
  - Singular forms: `addDay()`, `subMonth()`.
  - Business days: `addWeekdays`, `subWeekdays`.
  - By unit name: `add(90, "minutes")`.
  - An Interval: `plus(interval)`, `minus(interval)`.
- **Month overflow:** `addMonths` doesn't overflow: Jan 31 plus one month is
  Feb 28 or 29. `addMonthsWithOverflow` does it PHP's way (Mar 3).
- **Start and end of a unit:**
  - `startOfDay`, `startOfWeek`, `startOfMonth`, `startOfQuarter`,
    `startOfYear`, `startOfDecade`, `startOfCentury`, with an `endOf*` for each.
  - `startOf("week")` and `endOf("quarter")` by name.
  - Weeks start on Monday by default; `Moment.weekStartsAt` changes it.
- **Rounding:** `floor`, `ceil` and `round` to units up to a week, with a step:
  `floor("minute", 15)` gives the quarter hour.
- **Weekdays:** `next(5)` (the next Friday), `previous(1)`, `nextWeekday`,
  `nextWeekendDay`, `firstOfMonth(1)`, `lastOfMonth(5)` and
  `nthOfMonth(2, 2)` (the second Tuesday).

### Comparing

- **Operators and names:** `==`, `<`, `>`, and `eq`/`ne`/`gt`/`gte`/`lt`/`lte`,
  `isBefore`/`isAfter`, `between(a, b, inclusive)`.
- **Relative to now:** `isPast`, `isFuture`, `isToday`, `isTomorrow`,
  `isYesterday`, `isCurrentWeek`/`Month`/`Year`, `isNextWeek`, `isLastMonth`.
- **Same unit:** `isSameDay`, `isSameWeek`, `isSameMonth`, `isSameYear`,
  `isSameAs("Y-m", other)`, `isSameUnit`.
- **Kind of day:** `isWeekend`, `isMonday` … `isSunday`, `isStartOfDay`,
  `isMidday`, `isLastOfMonth`, `isBirthday`, `isLongYear`.
- **Picking:** `min`/`max`, `closest`/`farthest`, `Moment.earliest(list)`,
  `Moment.latest(list)`.

### Differences

- **In one unit:**
  - `diffInSeconds` … `diffInYears`, plus `diffInWeekdays` and
    `diffInWeekendDays`, or `diffIn("hours", other)`.
  - They give the size of the gap. Pass `absolute: false` for a signed
    result: positive when the other moment is later.
  - Days and months count on the calendar: 12:00 to 12:00 the next day is 1
    day even across a DST change, and Jan 31 to Feb 28 is 0 months.
- **As parts:** `diff(other)` gives an `Interval` of years, months, days and
  time.
- **For humans:**
  - `diffForHumans()`: `"3 days ago"`, `"in 5 hours"`, `"just now"`.
  - Against another moment: `"3 minutes before"`.
  - Options: `absolute` gives `"2 months"`, `short` gives `"1w"`, and
    `parts: 2` gives `"2 years 3 months ago"`.
- **Calendar:** `calendar()` gives `"Today at 9:30 AM"`, `"Yesterday at …"`,
  `"Tuesday at …"`, or `"Last Friday at …"`.

### Formatting

- **`format()`** takes every PHP `date()` letter: `d D j l N S w z W F m M n
  t L o Y y a A g G h H i s u v e I O P p T Z c r U`. Write `\` before a letter
  to print it as is.
- **Fixed formats:** `toDateString`, `toTimeString("minute")`,
  `toDateTimeString`, `toFormattedDateString`, `toDayDateTimeString`,
  `toIso8601String`, `toIso8601ZuluString`, `toAtomString`, `toCookieString`,
  `toRfc2822String`, `toRfc3339String`, `toRfc7231String`, `toRssString`,
  `toW3cString`.
- **JSON:** `toJson()` gives UTC with microseconds, as Carbon's does.
- **Other forms:** `toMap()` and `toDateTime()` (std's `DateTime`).

## Interval and Period

- **Making an `Interval`:**
  - From the parts: `Interval.of(years, months, days, hours, minutes, seconds, micros)`.
  - From one unit: `ofDays(3)`, `make(3, "weeks")`.
  - From text: `parse("1 day 2 hours")`, `parse("1h30m")`, ISO such as
    `parse("P1Y2M10DT2H30M")`, or a negative one like `"-1 day"`.
  - From std: `fromDuration(d)`.
- **Changing one:** `plus(2, "hours")`, `add`/`subtract`, `times(3)`,
  `negated`.
- **Normalising:** `cascade()` carries 90 minutes over to 1 hour 30 minutes.
- **Output:** `forHumans(short, parts)`, `toIso()`, and `toDuration()` (std's
  `Duration`).
- **Totals:** `totalSeconds` … `totalHours`. A month counts as 30 days and a
  year as 365, except for an interval from `diff()`, which knows the exact
  days.
- **`Period`:**
  - Making: `between(a, b, step)`, `create("2026-11-01", "2026-11-30", "1 week")`,
    or `times(start, 3, step)`.
  - Shaping: `excludeStart()`, `excludeEnd()`, `filter(f)`, `weekdays()`,
    `weekends()`.
  - Reading: `toArray`, `forEach`, `count`, `first`, `last`, `contains`,
    `overlaps`.
  - Each step is counted from the start, so months don't drift: Jan 31, then
    Feb 28, then Mar 31.

## Collections

`collect(array)` makes a `Collection<T>`. Most methods return a new
collection. `push`, `prepend`, `pop`, `shift`, `forget`, `splice` and
`transform` change it in place, as Laravel's do.

- **Reading:**
  - `first`, `firstWhere(f)`, `firstOr`, `last`, `lastWhere`
  - `sole(f)` wants exactly one match
  - `get(-1)`, `search`, `includes`, `contains`, `every`, `nth`
- **Transforming:** `map`, `mapIndexed`, `flatMap`, `filter`, `reject`,
  `reduce`, `each`, `eachWhile`, `pipe`, `tap`, `when`/`unless`.
- **By field name** (fields, getters, map keys or dotted paths):
  - `where("city", "Nairobi")`, `whereOp("age", ">=", 18)`
  - Operators: `= != < <= > >= like in`
  - `whereIn`, `whereNotIn`, `whereNull`, `whereBetween`
  - `pluck("email")`, `pluckText`, `sumOf("total")`
  - `sortByKeys(["-score", "name"])`
- **Groups:**
  - `groupBy`, `keyBy`, `countBy`, `partition`
  - `groupKeys` lists the keys in the order they first appear.
- **Splitting:** `chunk`, `chunkWhile`, `sliding` and `split`, which return
  a collection of collections, as Laravel's do.
- **Order:**
  - `sortBy` and `sortByDesc` work out each item's key once, not once per
    comparison, and are stable.
  - `sortByText`, `sort(cmp)`, `reverse`, `shuffle`, `random`
  - `unique(key)`, `distinct`, `duplicates`
- **Slicing:** `take(-3)`, `skip`, `slice`, `takeWhile`/`Until`,
  `skipWhile`/`Until`, `forPage`, `pad`.
- **Combining:** `concat`, `merge`, `zip` (an array of pairs), `diff`,
  `intersect`, `diffBy`, `intersectBy`.
- **Numbers:** `sum`, `avg`, `min`, `max`, `minBy`, `maxBy`, `median`,
  `mode`, `percentile`, and `total()`/`average()` for numeric items.
- **Text:** `join(", ", " and ")`, `implode`.

Pulling 100k integers through `filter`, `map` and `sortBy` takes about 8 ms.

## Data and Arr

- **`Data.get(value, "user.posts.*.title")`** reads into maps, arrays and
  reflected objects. A `*` collects every match.
- **Other `Data` helpers:**
  - `Data.set` writes to a path.
  - `Data.same` compares loosely, as Laravel's `where` does (`1 == "1"`).
  - `Data.compare` and `Data.like` back `where`'s operators.
- **`Arr`** works on `HashMap<string, any>`:
  - `get`, `set`, `has`, `forget`
  - `only`, `except`
  - `dot`, `undot`
  - `wrap`, `flatten`, `pluck`, `where`, `query`, `random`

## Text, Stringable and Number

- **`Text`:**
  - **Case:** `camel`, `studly`, `snake`, `kebab`, `title`, `headline`,
    `ucfirst`, `ucsplit`.
  - **Slugs:** `slug` and `ascii` (accented Latin letters become plain ASCII).
  - **Plurals:**
    - An English inflector: `plural("person")` gives `"people"`, and
      `singular` goes back.
    - It knows irregular and uncountable words, and keeps the word's case.
    - Also `pluralStudly("UserProfile")` and `pluralize(3, "item")`.
  - **Cutting:**
    - `limit`, `wordsLimit`, `excerpt`, `mask("ada@x.io", "*", 3)`
    - `after`/`afterLast`/`before`/`beforeLast`, `between`/`betweenFirst`
    - `start`, `finish`, `wrap`, `unwrap`
  - **Tests:** `is("admin/*", path)`, `containsAll`, `isUrl`, `isUuid`,
    `isUlid`, `isJson`, `isAscii`.
  - **Changing:** `squish`, `replaceArray`, `replaceLast`, `swap`, `remove`,
    `padBoth`, `wordWrap`, `length` (in characters, not bytes).
  - **Making:**
    - `random(n)` and `password(n)` use the operating system's random source
      (no modulo bias).
    - `uuid()`, `orderedUuid()` (version 7, sorted by time) and `ulid()`.
- **`Stringable`:** `Text.of(s).trim().studly().pluralStudly().toString()`
  chains the same helpers.
- **`Number`:**
  - `format(1234567.891, 2)` gives `"1,234,567.89"`
  - `percentage`, `currency(1250, "KES")`, `fileSize`
  - `abbreviate` (`"1.2K"`) and `forHumans` (`"2.3 billion"`)
  - `ordinal` (`"22nd"`), `spell(105)` (`"one hundred and five"`)
  - `clamp`, `pairs`

## Speed

These are release-build timings:

| Operation | Time |
|---|---|
| Making a moment in a zone with DST | about 100 ns |
| `addDays` | about 250 ns |
| `format("Y-m-d H:i:s")` | about 170 ns |
| An ISO parse | about 550 ns |

## Tests

```
grav test tests/support.grav
```
