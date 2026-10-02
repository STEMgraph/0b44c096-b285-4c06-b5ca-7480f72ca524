<!---
{
  "id": "0b44c096-b285-4c06-b5ca-7480f72ca524",
  "teaches": "GNU Make: Dependencies, Incremental Builds and .PHONY Targets",
  "depends_on": [
    "e9a90c71-845e-4e2c-a6af-1128a7921d93"
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2026-10-01",
  "keywords": [
    "make",
    "dependencies",
    "incremental build",
    ".PHONY",
    "clean",
    "make -n",
    "make --debug",
    "make -j"
  ]
}
--->

# GNU Make: Dependencies, Incremental Builds and `.PHONY` Targets

> In this exercise you will deliberately leave a header out of the dependency list and watch `make` produce a stale program, fix it, add `clean` and `all` targets, learn why they must be declared `.PHONY`, and use the debug options of `make` to see its decisions. No `sudo` is required.

## Introduction

`make` is only as correct as the dependency graph you give it. It never reads your C code. If `main.c` includes `vec2.h` but the `Makefile` does not say so, a change to the header will not trigger a rebuild, and the old `main_program` keeps running with the old struct layout. Everything *looks* fine until it does not.

Besides file targets, a `Makefile` usually has a few **convenience targets** that do not produce a file: `clean` removes build products, `all` builds everything, `install` copies files somewhere. These are called **phony** targets. They have to be marked with the special target `.PHONY`, otherwise `make` treats them like file names, and a file called `clean` in the directory would make `make clean` do nothing ("`clean` is up to date").

### 1.1) Further Readings and Other Sources

* [GNU Make Manual: Phony Targets](https://www.gnu.org/software/make/manual/html_node/Phony-Targets.html)
* [GNU Make Manual: Options Summary (`-n`, `-d`, `--debug`, `-j`)](https://www.gnu.org/software/make/manual/html_node/Options-Summary.html)
* [GNU Make Manual: Automatic Prerequisites](https://www.gnu.org/software/make/manual/html_node/Automatic-Prerequisites.html)

## Tasks

### Task 1: The Missing Header Dependency

Start in `~/make-intro` with the `Makefile` from the previous exercise. Build once, then change the header so that the struct gets a third component:

```bash
make
sed -i 's/double x, y;/double x, y, z;/' vec2.h
make
./main_program
```

`make` reports that everything is up to date. The program still runs. Now change `main.c` so it initialises `z` as well:

```c
vec2 a = { 3, 4, 0 }, b = { 1, 2, 0 };
```

```bash
make
./main_program
```

Only `main.c` was recompiled. `vec2.o` still uses the two-component struct. Depending on your compiler the output is wrong, garbage, or by luck correct. Explain why the two object files disagree about the size of `vec2`.

### Task 2: Add the Header to the Dependencies

```make
main_program: main.c vec2.h libvec2.a
	gcc -Wall -Wextra -O2 main.c -o main_program -L. -lvec2 -lm

libvec2.a: vec2.o
	ar rcs libvec2.a vec2.o

vec2.o: vec2.c vec2.h
	gcc -Wall -Wextra -O2 -c vec2.c -o vec2.o
```

```bash
make
./main_program
touch vec2.h
make
```

Now a change to the header rebuilds both sides. Undo the `z` experiment (remove the third component in `vec2.h` and `main.c`) and rebuild.

### Task 3: A `clean` Target

Add at the end of the `Makefile`:

```make
clean:
	rm -f vec2.o libvec2.a main_program
```

```bash
make clean
ls
make
```

### Task 4: Break `clean` With a File

```bash
touch clean
make clean
```

Read the message. `make` found a file called `clean`, it has no dependencies, so it is up to date. Fix this by declaring the target phony:

```make
.PHONY: clean
```

```bash
make clean
rm clean
```

### Task 5: An `all` Target as Default

`make` without arguments builds the **first** target in the file. Make that an explicit `all`:

```make
all: main_program

.PHONY: all clean
```

Put the `all` rule at the top of the file. Run `make`, then `make clean`, then `make all`. Why is it good practice for the default target to be phony?

### Task 6: Watch `make` Think

```bash
make clean
make --debug=b
make --debug=b
touch vec2.c
make -n
make -d 2>&1 | wc -l
```

Read the `--debug=b` output of the first and the second run side by side. Find the lines "Must remake target" and "is up to date". The `-d` output is long; find the line in it that explains why `vec2.o` is considered out of date.

### Task 7: Parallel Builds

Add a second, independent library so there is something to parallelise:

```c
// file: util.c
int UTIL_twice(int x) { return 2 * x; }
```

Add the rules `libutil.a: util.o` and `util.o: util.c`, add `libutil.a` to the dependencies and `-lutil` to the link line of `main_program`. Then:

```bash
make clean
make -j2 --debug=b 2>&1 | grep -i 'remake\|considering' | head
```

Which two recipes can run at the same time, and which one must wait? Draw the dependency graph on paper and mark the critical path.

## Questions

- In Task 1, `make` said "up to date" after the header changed. Was `make` wrong? Who was?
- Why does `.PHONY` not only fix the `touch clean` problem but also make `make clean` slightly faster?
- What does `make -n` print for a phony target that has already been "built" in this run?
- Why is `all` conventionally the first target, and what happens if you put `clean` first by accident?
- `make -j` runs recipes in parallel. What property of the dependency graph makes that safe? What happens if two recipes write the same file?
- Large projects do not list every header by hand. Look up `gcc -MMD` and explain in two sentences how it solves the problem from Task 1.

## Advice

The failure in Task 1 is silent, and that is what makes it dangerous. Whenever a program behaves strangely after a header change, the first thing to try is `make clean && make`. If that fixes it, your dependency list is incomplete.

`.PHONY` costs nothing and prevents a whole class of mysterious "nothing happens" problems. Declare every non-file target phony, always.

`make -n` and `make --debug=b` are the two tools you will use most when you inherit someone else's `Makefile`. Use them before you edit anything.
