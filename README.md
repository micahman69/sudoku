# Solving Every Sudoku Puzzle


SOLVE SUDOKU THING!!!!

## Sudoku Notation and Preliminary Notions

First we have to agree on some notation. A Sudoku puzzle is a grid of 81
squares; the majority of enthusiasts label the columns 1-9, the rows A-I,
and call a collection of nine squares (column, row, or box) a unit and the
squares that share a unit the peers. A puzzle leaves some squares blank and
fills others with digits, and the whole idea is:

  A puzzle is solved if the squares in each unit are filled with a
  permutation of the digits 1 to 9.

## Notes about the CL port

The main goal here is to have fun in CL, of course, and then to learn more
of it along the way. So that port is not as straighforward as one would
think, I've been trying to make good use of some CL features.

The first consequence concerns data structures used, as those lists and hash
tables in python are better replaced with two dimensional arrays in CL, and
abstracted away in a class. Probably. At least that's what I did here.

So I first picked bit-vectors (that is, array of bits) to represent the set
of values that are still possible to assign in each square of the grid, and
then was told that the usual way in CL is to use the advanced function that
act on an integer directly in its 2-complement bit representation.

And of course that integer-as-bitfields implementation performs way better!

### logcount, logbitp and friends

Using numbers as bitfields is quite simple with functions such as `logcount`
and `logbitp`, as you can see here:

```lisp
(defun count-remaining-possible-values (possible-values)
  "How many possible values are left in there?"
  ;; we could raise an empty-values condition if we get 0...
  (logcount possible-values))

(defun first-set-value (possible-values)
  "Return the index of the first set value in POSSIBLE-VALUES."
  (+ 1 (floor (log possible-values 2))))

(defun only-possible-value-is? (possible-values value)
  "Return a generalized boolean which is true when the only value found in
   POSSIBLE-VALUES is VALUE"
  (and (logbitp (- value 1) possible-values)
       (= 1 (logcount possible-values))))

(defun list-all-possible-values (possible-values)
  "Return a list of all possible values to explore"
  (loop for i from 1 to 9
     when (logbitp (- i 1) possible-values)
     collect i))

(defun value-is-set? (possible-values value)
  "Return a generalized boolean which is true when given VALUE is possible
   in POSSIBLE-VALUES"
  (logbitp (- value 1) possible-values))

(defun unset-possible-value (possible-values value)
  "return an integer representing POSSIBLE-VALUES with VALUE unset"
  (logxor possible-values (ash 1 (- value 1))))
```

## Performances

That's somewhat useless and generally is responsible for loads of
visibility, so let's have some nice numbers here. Note that I'm using the
Clozure Common Lisp implementation (CCL for short) here.

### Original python version

I had to change a single line in the Norvig's python code, given that I had
to hack the sodoku.txt in order to get the easy50.txt file content, and
obviously that's not matching what Norvig has been working with:

```diff
-    solve_all(from_file("easy50.txt", '========'), "easy", None)
+    solve_all(from_file("easy50.txt"), "easy", None)
```

Now for the results:  

    dim ~/dev/CL/sudoku python sudoku.dim.py 
    All tests pass.
    Solved 50 of 50 easy puzzles (avg 0.01 secs (151 Hz), max 0.01 secs).
    Solved 95 of 95 hard puzzles (avg 0.02 secs (42 Hz), max 0.12 secs).
    Solved 11 of 11 hardest puzzles (avg 0.01 secs (115 Hz), max 0.01 secs).

### CCL, bit-vectors

That's my implementation of choice when developping, but it's known for not
being very fast, let's see about that:

    CL-USER> (sudoku:solve-example-grids)
    Solved 50 of 50 easy puzzles (avg 0.005 sec (185.1 Hz), max 0.021 secs).
    Solved 95 of 95 hard puzzles (avg 0.579 sec (1.728 Hz), max 8.845 secs).
    Solved 11 of 11 hardest puzzles (avg 0.018 sec (55.12 Hz), max 0.102 secs).
    NIL

### CCL, fixnums

So, when using logcount, logbitp and friends, we have:

    CL-USER> (time (sudoku:solve-example-grids))
    Solved 50 of 50 easy puzzles (avg 0.003 sec (394.3 Hz), max 0.008 secs).
    Solved 95 of 95 hard puzzles (avg 0.003 sec (333.8 Hz), max  0.01 secs).
    Solved 11 of 11 hardest puzzles (avg 0.002 sec (422.8 Hz), max 0.004 secs).
    
    (SUDOKU:SOLVE-EXAMPLE-GRIDS)
    took 441,831 microseconds (0.441831 seconds) to run.
          35,660 microseconds (0.035660 seconds, 8.07%) of which was spent in GC.
    During that period, and with 2 available CPU cores,
         418,886 microseconds (0.418886 seconds) were spent in user mode
          14,287 microseconds (0.014287 seconds) were spent in system mode
     41,960,896 bytes of memory allocated.
    NIL
  
### SBCL, bit-vectors

And the results with sbcl which is known to be faster than ccl:

    * (sudoku:solve-example-grids)
    Solved 50 of 50 easy puzzles (avg .0029 sec (347.2 Hz), max 0.023 secs).
    Solved 95 of 95 hard puzzles (avg .2143 sec (4.666 Hz), max 3.139 secs).
    Solved 11 of 11 hardest puzzles (avg .0069 sec (144.7 Hz), max 0.036 secs).
    NIL

Not good. Yet.

### SBCL, fixnums

So, let's see if now there's still that big a difference between CCL and
SBCL:

    * (time (sudoku:solve-example-grids))
    Solved 50 of 50 easy puzzles (avg .0021 sec (471.7 Hz), max 0.015 secs).
    Solved 95 of 95 hard puzzles (avg .0022 sec (446.0 Hz), max 0.008 secs).
    Solved 11 of 11 hardest puzzles (avg .0018 sec (550.0 Hz), max 0.003 secs).
    Evaluation took:
      0.341 seconds of real time
      0.327922 seconds of total run time (0.324250 user, 0.003672 system)
      [ Run times consist of 0.011 seconds GC time, and 0.317 seconds non-GC time. ]
      96.19% CPU
      1,039,299,781 processor cycles
      27,718,320 bytes consed
      
    NIL

That's fast, now!

## Conclusion

The common lisp port has been easy enough to implement, and we get a nice 3x
to 10x speedup for having done so. The interactive development facilities
offered in Common Lisp (SLIME in Emacs) are amazing, so that it's been easy
to test every function alongside writing it.
