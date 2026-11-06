# RACE.md

One section per milestone. Fill each one in order as you go.

The prediction in milestone 1 is written BEFORE you run your race test. A TA
may ask about it. Commit it on its own so the order is visible.

Keep it short and specific. Point at methods, call sites, and the invariant.

---

## Milestone 1: from the ticket to a test that fails every time

### Suspect list

Everything your review turned up that touches shared state, with a one-line
verdict for each. Add or remove rows as needed.

| # | What you found (method, field, or class) | Verdict, in one line |
|---|---|---|
| 1 | `BookingService.book`: `store.hasOverlap(...)` (line 70), then `store.insert(...)` (line 74) | **Check-then-act race.** Each store call locks separately, so the lock is free between the check and the insert, and two bookers can both see "free" and both insert CONFIRMED. Explains the ticket. |
| 2 | `BookingStats` counters (`confirmed++` and the rest) | Lost updates are real, but documented and accepted. Nothing reads the counters, and the ticket says the bad state is in the booking data. Not the ticket. |
| 3 | `BookingService.sequence` (`AtomicLong`) via `nextId()` | Atomic, so ids are unique. The ticket confirms the two bookings had different ids. Safe. |
| 4 | `BookingService.catalog` (`HashMap`) | Written only in the constructor and published through a `final` field. Read-only afterwards. Safe. |
| 5 | `InMemoryBookingStore.bookings` and its four methods | Every method is `synchronized`, the list never escapes, and callers get copies. Each call is atomic, but only on its own. That gap is what makes #1 possible. |
| 6 | `InMemoryBookingStore.cancelAndPromote` | Cancel, re-check, and promote all happen under one lock, so it is safe by itself. The ticket rules out cancellation. Not yet tested: a promotion could land in #1's gap (same defect, a different second writer). |
| 7 | `BookingService.joinWaitlist` → `insert(WAITLISTED)` | Inserts with no check, but WAITLISTED blocks nothing and is not CONFIRMED. Cannot break the invariant. |
| 8 | `BookingLog` | No mutable state. The formatter is immutable and `println` is synchronized per line. Safe. |

**Which one explains the ticket, and how you know.** #1. The invariant is "a room has at most one
CONFIRMED booking covering any minute." Only two paths create a CONFIRMED booking: `book()` and
`cancelAndPromote()`. `cancelAndPromote()` checks and writes under one lock. `book()` checks under
one lock acquisition and writes under a second one, so its "free" answer can be out of date by the
time it inserts. The `BookingService` javadoc assumes that composing thread-safe calls is
thread-safe, and it isn't. The ticket fits:

- The two confirmations are 1 ms apart with nothing logged between them, so both checks passed before either insert.
- It happens in the first minute after a slot opens, when one slot is busiest.
- It could not be reproduced by hand, because human timing is milliseconds and the window is microseconds.
- The suite stays green because all 11 tests are single-threaded.

### Predicted interleaving (write this before you run your race test)

Two threads, one row per step, in the form from lecture. Describe what each
thread does at that step and leave the other column empty. Add or remove rows
as your prediction needs.

| Step | Thread A | Thread B |
|---|---|---|
| t1 | `book(WEH-5409, 540, 600)` → `hasOverlap` returns **false** (store empty). `PausingStore` parks A before `book()` sees the answer. | |
| t2 | | `book(WEH-5409, 540, 600)` → `hasOverlap` returns **false** (A has not inserted) |
| t3 | | `nextId()` → `bk-1`, `insert` CONFIRMED, logs `booking.confirmed booking=bk-1` |
| t4 | (released) acts on its stale "false": `nextId()` → `bk-2`, `insert` CONFIRMED, logs `booking.confirmed booking=bk-2` | |
| | **End state:** two CONFIRMED bookings (`bk-1`, `bk-2`) for WEH-5409 `[540,600)`, both callers told CONFIRMED. The test fails with expected 1, was 2. The log shows two `booking.confirmed` lines, `bk-1` first, a few ms apart, with nothing in between, the same shape as the ticket. | |

If the run turned out different from your prediction, add one line on what
you missed. Leave the table as you first wrote it.

### Your race test

**How you aimed `PausingStore`, and the result.** Method name and call index, why
that pair pins the schedule you predicted, and the output of two separate
runs (assertion failure and log lines).

Test: `RaceTest.sameSlotBookedTwiceLeavesOneConfirmedBooking`.

**Aim: `hasOverlap`, call 1.** `PausingStore` parks a call *after* the store
answers. Parking A's first `hasOverlap` therefore holds A between its check
(already "free") and its `insert`, which is the gap in `BookingService.book`.

- **Why call 1 is A's:** B starts only after `awaitPaused()` returns true. So B's check is call 2, and it passes straight through the gap.
- **Why not `insert`, call 1:** that would park A *after* its write. B would then be rejected correctly, and the race would stay hidden.
- **Guards against a false pass:** the test asserts that the gate fired and that both threads finished before it checks the end state.
- **What it asserts:** exactly one CONFIRMED booking covering WEH-5409 `[540,600)` in the real store. It never asserts the order of events.

Run 1 (`mvn -B test -Dtest=RaceTest`):

```
2026-11-06T03:11:17.743Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-06T03:11:17.751Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.125 s <<< FAILURE! -- in edu.cmu.cs214.lab10.harness.RaceTest
org.opentest4j.AssertionFailedError: CONFIRMED bookings covering WEH-5409 540..600; store holds [Booking[id=bk-1, roomId=WEH-5409, startMinute=540, endMinute=600, status=CONFIRMED], Booking[id=bk-2, roomId=WEH-5409, startMinute=540, endMinute=600, status=CONFIRMED]] ==> expected: <1> but was: <2>
	at edu.cmu.cs214.lab10.harness.RaceTest.sameSlotBookedTwiceLeavesOneConfirmedBooking(RaceTest.java:58)
[INFO] BUILD FAILURE
```

Run 2 (`mvn -B test -Dtest=RaceTest`):

```
2026-11-06T03:11:21.613Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-06T03:11:21.619Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.108 s <<< FAILURE! -- in edu.cmu.cs214.lab10.harness.RaceTest
org.opentest4j.AssertionFailedError: CONFIRMED bookings covering WEH-5409 540..600; store holds [Booking[id=bk-1, roomId=WEH-5409, startMinute=540, endMinute=600, status=CONFIRMED], Booking[id=bk-2, roomId=WEH-5409, startMinute=540, endMinute=600, status=CONFIRMED]] ==> expected: <1> but was: <2>
	at edu.cmu.cs214.lab10.harness.RaceTest.sameSlotBookedTwiceLeavesOneConfirmedBooking(RaceTest.java:58)
[INFO] BUILD FAILURE
```

**Prediction against what ran.** It matched. Both runs fail at line 58, the end-state
assertion, which means the gate-fired and threads-finished checks above it passed. Each run
shows two `booking.confirmed` lines for the same room and range, `bk-1` first, 6–8 ms apart,
with nothing logged between them. That is the ticket's shape.

One limit: the log does not name threads. That `bk-1` is B's booking is inferred from the
code (`nextId()` runs after the check, while A is parked), not observed directly.

Full suite at this commit: `Tests run: 12, Failures: 1`. Only this test fails; the 11
original tests stay green. CI is expected to be red until milestone 3.

**Follow-up checks (run after the commit above; the text above is left as written).**
These used a throwaway test class with the same `PausingStore` wiring. It recorded
each thread's `BookingResult` and was deleted after the run, so it is not in the repo.

```
aim=hasOverlap#1  gateFired=true  A=CONFIRMED(bk-2)  B=CONFIRMED(bk-1)  confirmedIn540..600=2
aim=insert#1      gateFired=true  A=CONFIRMED(bk-1)  B=REJECTED         confirmedIn540..600=1
cancel-in-gap     gateFired=true  promoted=bk-w      A=CONFIRMED(bk-1)  confirmedIn540..560=2
```

- **Thread ids:** the inference above holds. B gets `bk-1` and A gets `bk-2`, now seen directly from each thread's result.
- **The `insert` aim:** parking `insert` call 1 hides the race. B is rejected and one booking is confirmed, as the aim rationale says.
- **Suspect #6:** confirmed, no longer just reasoned. The setup puts `bk-y` CONFIRMED on `[480,540)` and `bk-w` WAITLISTED on `[500,560)` straight into the store.
  - A calls `book(540, 600)` and is parked after `hasOverlap` returns false.
  - `cancelBooking(bk-y)` promotes `bk-w`. Then A inserts.
  - Result: `bk-w` and `bk-1` are both CONFIRMED over minutes 540–559.
  - This is the same check-then-act gap as #1, crossed by a different second writer. A milestone 3 fix has to cover `cancelAndPromote` as well as concurrent `book()` calls.

---

## Milestone 2: the patch a teammate sent

**Your milestone-1 test on the patched code.** Green or red. Paste the result.

**What the patch does.** It adds a second `store.hasOverlap(...)` (patched line 78), after
`nextId()` and just before `store.insert(...)` (patched line 81). The check and the insert are
still two separately locked store calls.

**Result: green.** The gate fired, because the test's `awaitPaused()` assertion passed.
- A is parked at call 1, its *first* check, before the re-check.
- B checks, gets `bk-1`, re-checks, and inserts.
- On release, A's re-check sees `bk-1` and rejects.

```
2026-11-06T03:22:50.186Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-06T03:22:50.201Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

**The re-aimed run.** Method name, call index, the pasted result, and one line
on why that aim tests the patch rather than sidestepping it.

Test: `RaceTest.sameSlotBookedTwiceAfterTheReCheckLeavesOneConfirmedBooking`.

**Aim: `hasOverlap`, call 2.** A runs alone until it is parked, so calls 1 and 2 are A's first
check and A's re-check. Parking call 2 holds A after the patch's own re-check, which is its
freshest "free". B then books through exactly the window the patch says is closed. That makes
this a test of the patch's last check rather than a way around it. The milestone 1 test and its
oracle are unchanged. Result: **red on both runs**, at the end-state assertion (line 98), so
the gate fired.

```
2026-11-06T03:28:03.090Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
2026-11-06T03:28:03.098Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
org.opentest4j.AssertionFailedError: CONFIRMED bookings covering WEH-5409 540..600; store holds [Booking[id=bk-2, roomId=WEH-5409, startMinute=540, endMinute=600, status=CONFIRMED], Booking[id=bk-1, roomId=WEH-5409, startMinute=540, endMinute=600, status=CONFIRMED]] ==> expected: <1> but was: <2>
	at edu.cmu.cs214.lab10.harness.RaceTest.sameSlotBookedTwiceAfterTheReCheckLeavesOneConfirmedBooking(RaceTest.java:98)
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0
```

```
2026-11-06T03:28:07.035Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
2026-11-06T03:28:07.043Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
org.opentest4j.AssertionFailedError: CONFIRMED bookings covering WEH-5409 540..600; store holds [Booking[id=bk-2, roomId=WEH-5409, startMinute=540, endMinute=600, status=CONFIRMED], Booking[id=bk-1, roomId=WEH-5409, startMinute=540, endMinute=600, status=CONFIRMED]] ==> expected: <1> but was: <2>
	at edu.cmu.cs214.lab10.harness.RaceTest.sameSlotBookedTwiceAfterTheReCheckLeavesOneConfirmedBooking(RaceTest.java:98)
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0
```

The test was also red in two earlier whole-class runs (03:23:26Z and 03:23:30Z) before it was
disabled.

**Status in the repo:** `@Disabled`. The patch is rejected and not applied. On the shipped code
a lone booking makes only one `hasOverlap` call, so this aim would never fire. To rerun it:

```
git apply proposed-fix.diff
mvn -B test -Dtest='RaceTest#sameSlotBookedTwiceAfterTheReCheckLeavesOneConfirmedBooking' '-Djunit.jupiter.conditions.deactivate=org.junit.*DisabledCondition'
git apply -R proposed-fix.diff
```

**Your verdict.** Accept or reject, and what your milestone-1 result does and
does not show about the patch.

**Reject.** The re-check repeats the check but does not make it atomic with the insert. The
store's lock is still released between `hasOverlap` (line 78) and `insert` (line 81), so the
check-then-act gap moved rather than closed. The deterministic red run above shows two
CONFIRMED bookings for the same minutes.

- **What the green milestone 1 result shows:** one schedule is now caught, a booking that lands between A's first check and its re-check.
- **What it does not show:** that no schedule breaks the invariant. A green race test proves only the interleaving it pins.
- **The teammate's evidence does not support the claim either:**
  - The suite is single-threaded, so it cannot see a race.
  - The stress runner says "narrowed, not fixed." It was run 5 times on the patch (1/100, 0/100, 1/100, 2/2000, 2/2000 bad rounds), against 21/100, 22/100, 20/100 and 63/2000 on the shipped code. These numbers are non-deterministic and supporting only.
  - "A few runs, no bad rounds" is the same kind of evidence that failed to reproduce the ticket.
- **Side effect the patch doesn't mention:** `nextId()` (line 73) now runs before the re-check, so an attempt the re-check rejects still uses up an id. In the green run above, A drew `bk-2` and was rejected. `bk-2` never appears in the log or the store, which fits. Ids no longer map to confirmed bookings one to one. That is harmless to the invariant, but it is an unannounced behavior change.

---

## Milestone 3: the fix you would ship

**Which option, and what you changed.** Name the lock or the atomic operation
and the two operations it now covers. Paste the diff of your change and the
passing run of your milestone-1 test and the full suite.

**Option 2: an atomic check-and-insert in the store.** The new method is
`BookingStore.insertIfFree(Booking)`, a plain interface method. In
`InMemoryBookingStore` it is `synchronized` on the store, so the overlap check and the
`add` run under **one** hold of the store's lock. The two operations it covers are the
overlap check (`hasOverlap`) and the insert (`insert`).

`book()` keeps its `hasOverlap` pre-check as a quick way to reject a slot that is already
taken, but the decision that counts is now `insertIfFree`. Why this option rather than
`synchronized` or a `ReentrantLock` on the service:
- **Same lock as `cancelAndPromote`:** the store's lock also guards `cancelAndPromote`, so a promotion can no longer land in the gap.
- **Covers every service instance:** it covers every service that shares the store (web front end and kiosk). A service-level lock covers only one instance.

Adding the method to the interface broke the build, as the handout predicted:

```
[ERROR] .../harness/PausingStore.java:[40,14] edu.cmu.cs214.lab10.harness.PausingStore is not abstract and does not override abstract method insertIfFree(edu.cmu.cs214.lab10.domain.Booking) in edu.cmu.cs214.lab10.store.BookingStore
```

`PausingStore` now passes the call through its `gate(...)`, so it can still be aimed at.

```diff
--- a/src/main/java/edu/cmu/cs214/lab10/store/BookingStore.java
+++ b/src/main/java/edu/cmu/cs214/lab10/store/BookingStore.java
+    /**
+     * Stores the booking only if the room has no CONFIRMED booking touching its
+     * range, as one indivisible step: no other call can change the room between
+     * the check and the insert. Use this instead of {@link #hasOverlap} followed
+     * by {@link #insert} whenever the insert depends on the check.
+     * ...
+     */
+    boolean insertIfFree(Booking booking);
--- a/src/main/java/edu/cmu/cs214/lab10/store/InMemoryBookingStore.java
+++ b/src/main/java/edu/cmu/cs214/lab10/store/InMemoryBookingStore.java
+    @Override
+    public synchronized boolean insertIfFree(Booking booking) {
+        // Check and add under one hold of this store's lock, so no other call
+        // can confirm an overlapping booking in between.
+        if (hasOverlap(booking.roomId(), booking.startMinute(), booking.endMinute())) {
+            return false;
+        }
+        bookings.add(booking);
+        return true;
+    }
--- a/src/main/java/edu/cmu/cs214/lab10/harness/PausingStore.java
+++ b/src/main/java/edu/cmu/cs214/lab10/harness/PausingStore.java
-     *     name: {@code hasOverlap}, {@code insert}, {@code bookingsForRoom}, or
-     *     {@code cancelAndPromote}
+     *     name: {@code hasOverlap}, {@code insert}, {@code insertIfFree},
+     *     {@code bookingsForRoom}, or {@code cancelAndPromote}
+    @Override
+    public boolean insertIfFree(Booking booking) {
+        return gate("insertIfFree", () -> delegate.insertIfFree(booking));
+    }
--- a/src/main/java/edu/cmu/cs214/lab10/service/BookingService.java
+++ b/src/main/java/edu/cmu/cs214/lab10/service/BookingService.java
- * through {@link BookingStore}, whose methods are each synchronized. Since the
- * store is thread safe, the operations built on top of it are too. The only
- * other field that changes after construction is {@link BookingStats}, and its
- * numbers are documented as approximate.
+ * through {@link BookingStore}. Each store call is atomic on its own, but a
+ * sequence of calls is not, so a write that depends on a check is made in one
+ * store call ({@link BookingStore#insertIfFree}). The only other field that
+ * changes after construction is {@link BookingStats}, and its numbers are
+ * documented as approximate.
         Booking booking = new Booking(nextId(), roomId, startMinute, endMinute, BookingStatus.CONFIRMED);
-        store.insert(booking);
+        // The check above only turns away a slot that is already taken. It
+        // releases the store's lock before the write, so the decision that
+        // counts is made here, where the store checks and inserts in one step.
+        if (!store.insertIfFree(booking)) {
+            return reject(roomId, startMinute, endMinute, "slot already held");
+        }
         stats.recordConfirmed();
```

Full suite, two runs (`mvn -B test`). The race test lines come first:

```
# run 1
03:37:01.924Z INFO booking.cancelled booking=bk-y                                    <- cancellation test
03:37:01.945Z INFO booking.promoted booking=bk-w room=WEH-5409 start=500 end=560
03:37:01.956Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
03:37:01.967Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600    <- milestone 1 test (B)
03:37:01.967Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held   <- (A)
[WARNING] Tests run: 3, Failures: 0, Errors: 0, Skipped: 1 -- in edu.cmu.cs214.lab10.harness.RaceTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0 -- in edu.cmu.cs214.lab10.service.BookingServiceTest
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0 -- in edu.cmu.cs214.lab10.domain.BookingTest
[WARNING] Tests run: 14, Failures: 0, Errors: 0, Skipped: 1
[INFO] BUILD SUCCESS

# run 2
03:37:06.563Z INFO booking.cancelled booking=bk-y
03:37:06.584Z INFO booking.promoted booking=bk-w room=WEH-5409 start=500 end=560
03:37:06.599Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
03:37:06.611Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
03:37:06.611Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
[WARNING] Tests run: 14, Failures: 0, Errors: 0, Skipped: 1
[INFO] BUILD SUCCESS
```

The one skip is the milestone 2 test, which stays `@Disabled` because it is aimed at the
rejected patch.

- **Test changes:** none to the milestone 1 test. Its method is byte-for-byte the same as in `251d6ec`. One test was added, `promotionDuringTheBookingGapLeavesOneConfirmedBooking`, for suspect #6. It was committed red on the shipped code (`0a32fb5`: expected 1, was 2 CONFIRMED over 540..560) and is green now.
- **Stress runner, supporting evidence only:** 0/100, 0/100, 0/100, 0/2000 and 0/2000 bad rounds. Before the fix it was about 20/100.

**Is your milestone-1 test still testing anything?** After your fix, describe
what that test now exercises. Did `PausingStore` pause a call during the green run? How do
you know?

Yes. It drives the **same schedule** that was red: A is parked right after its
`hasOverlap` (call 1) answered "free", and B books the same slot through that pause.

What changed is what A does next. Its write is now `insertIfFree`, which re-checks under the
same hold of the lock as the add. So A sees B's booking and is rejected: `bk-1` CONFIRMED
(B), then `booking.rejected` (A), and one CONFIRMED booking in the end state.

**The gate fired in both green runs.** The test's first assertion is
`assertTrue(gated.awaitPaused(), ...)`, and the test passed. B also didn't wait (the class
ran in 0.1 s, with no 2 s timeout), because `PausingStore` parks A *after* `hasOverlap` has
returned, outside the store's lock.

What the test cannot show: it cannot park A *inside* `insertIfFree`, between the check and
the `add`. There is no store-call boundary there, which is the point of the fix. That part
rests on the lock, not on a test.

**What runs one at a time now that ran in parallel before.** Name concretely which callers
queue behind which. "Nothing, it is just correct now" does not count as an
answer.

Before the fix, each store call ran alone, but other calls could run between A's
`hasOverlap` and A's `insert`. Now `insertIfFree` holds the store's lock across the
overlap scan **and** the add. Who now waits for the whole check-and-insert pair instead of
slipping between its two halves:

- **Another `book()` for the same slot**, from any `BookingService` on this store (web front end or kiosk). Its pre-check or `insertIfFree` waits, then sees the first booking and is rejected. This is the pair that caused RES-4471.
- **`cancelBooking` → `cancelAndPromote`.** A promotion waits for an in-progress check-and-insert, or the insert waits for the promotion. Either way, the second one sees the first.
- **Every other store caller, for every room:** `book()` for unrelated rooms, `listBookings` (`bookingsForRoom`), and `joinWaitlist` (`insert`). One list and one lock means they already ran one at a time per call. Now each of them can wait for a scan *plus* an add instead of one or the other.
- **Extra cost:** every successful booking now takes the store's lock twice, once for the pre-check and once for `insertIfFree`. The longest hold is still one O(n) scan over all bookings, plus an add.

**What you left unfixed.** For anything your review flagged that you chose not
to fix, one line each on why.

- **`BookingStats` lost updates (#2):** documented as approximate, nothing reads the counters, and exact counting is not worth contention on every booking.
- **More than one process or replica:** the lock is per JVM, and the store is in memory per process. A shared database would need its own atomic enforcement, such as a constraint or a transaction with suitable isolation. Out of scope for this in-memory service.
- **`hasOverlap` and `insert` are still public on `BookingStore`:** `joinWaitlist` needs `insert`, and removing `hasOverlap` is an API change beyond this fix. A future caller could rebuild the race. The `insertIfFree` javadoc says to use it whenever an insert depends on a check.
- **An id is used up when a booking loses the race:** `nextId()` runs before `insertIfFree`, so the losing booking consumes an id. In the milestone 1 green run, A drew `bk-2` and was rejected. No contract promises contiguous ids, and the pre-check keeps this to real races, not every rejection.
- **The milestone 2 test stays `@Disabled`:** it is aimed at the rejected patch. RACE.md milestone 2 has the command to rerun it.
