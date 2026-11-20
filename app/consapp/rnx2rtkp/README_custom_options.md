# rnx2rtkp custom options

Two options were added to `rnx2rtkp` in this fork of RTKLIB demo5.

```
rnx2rtkp [options] -p 0 [-st] [-w weights.txt] -o out.pos rover.obs nav.nav
```

## `-st` : treat every epoch as the first epoch's time

When a RINEX file holds several *sets* of satellite combinations that should be
solved as though they were observed at the same instant, `-st` overwrites the
time of every observation with the time of the first observation in the file.
Each epoch record is then processed as an independent solution at that same time.

- Takes no argument.
- Implemented in `procpos()` (`src/postpos.c`) via the global `global_st_flag`,
  which `postpos()` now receives as its last argument (`st_flag`).
- Because every epoch shares one timestamp, satellite positions/clock
  corrections are computed for that single time.

## `-w <file>` : per-satellite weights for the single-point least-squares solver

Overrides the measurement variance of chosen satellites in the SPP weighted
least-squares (`estpos()` in `src/pntpos.c`). Use it with single-point mode
(`-p 0`).

### File format

One satellite per line. `#` starts a comment; blank lines are ignored.

```
# sat   value   [unit]
G05     1.5               # unit defaults to sigma
E11     0.44    weight
C27     3.0     sigma
```

| unit            | meaning                          | variance used |
|-----------------|----------------------------------|---------------|
| `sigma` (default) | 1-sigma pseudorange error, metres | `value^2`     |
| `weight`        | weight `w`                        | `1 / value`   |

So `G11 100 sigma` and `G11 0.0001 weight` are equivalent.

### Behaviour

- A listed satellite's variance **replaces the whole stock variance** (elevation/SNR
  model, ionosphere, troposphere and ephemeris terms). Unlisted satellites keep
  the normal RTKLIB model.
- The weight is per satellite ID, not per time. When the same satellite appears
  more than once (e.g. with `-st`), every occurrence gets the same weight.
- Lines with an unknown satellite ID, a non-positive value or an unknown unit
  print `loadsatweights: skipping bad line: ...` to stderr and are skipped.
- If the file cannot be opened, `rnx2rtkp` prints an error and exits.
- A weight on a satellite that is excluded (below the elevation mask, excluded
  with `-x`/`exsats`, etc.) has no effect.
- Only the SPP/WLS path (`pntpos`) is affected. RTK and PPP filters are not,
  except that their initial SPP position uses the weights.

### Implementation

- `loadsatweights()` in `src/pntpos.c` parses the file into a static table
  `satvar[MAXSAT]` (0 = not set); declared in `src/rtklib.h`.
- `rescode()` uses `satvar[sat-1]` when it is positive. RAIM FDE goes through the
  same function, so it sees the same weights.
- The `-w` option is parsed in `app/consapp/rnx2rtkp/rnx2rtkp.c`.

## Building

On machines without `libgfortran` the stock makefile fails to link. The IERS
library is not needed with the default options, so build with:

```
cd app/consapp/rnx2rtkp/gcc
make LDLIBS="-lm -lrt"
```

## Other behaviour changes in this fork

- `pntpos()`: RAIM FDE now runs whenever `n>=6` and `posopt[4]` is set, even if
  the first `estpos()` call failed (previously required it to have succeeded).
- `outnmea_gga()` (`src/solution.c`): latitude degrees field format is `%02.3f`.

## Quick check

```
rnx2rtkp -p 0 -o base.pos obs.rnx nav.rnx
echo "G11 100 sigma" > w.txt
rnx2rtkp -p 0 -w w.txt -o weighted.pos obs.rnx nav.rnx   # solution should differ
```
