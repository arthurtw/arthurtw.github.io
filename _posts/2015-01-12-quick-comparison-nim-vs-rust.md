---
title: A Quick Comparison of Nim vs. Rust
layout: post
---

> EDIT (Jan-13): Use Nim’s `-d:release` instead of `--opt:speed` flag. Use Rust’s `regex!` macro, and `collections::HashMap` instead of `BTreeMap`.
>
> EDIT (Jan-15): Add Rust results with regular expression `r"[a-zA-Z0-9_]+"`.
>
> EDIT (Jan-17): Refine Rust code for Zachary Dremann’s pull requests. Edit Rust’s strengths list.

[Rust](http://www.rust-lang.org/) and [Nim](http://nim-lang.org/) are the two new programming languages I have been following for a while. Shortly after my previous [blog post](http://arthurtw.github.io/2014/12/21/rust-anti-sloppy-programming-language.html) about Rust, [Nim 0.10.2](http://nim-lang.org/news.html#Z2014-12-29-version-0-10-2-released) was out. This led me to take a closer look at Nim, and, naturally, compare it with Rust.

In this blog post, I’m going to share my two simple examples written in Nim and Rust, the rough execution time comparison, and my subjective impressions of coding in both.

## Example 1: wordcount

This example exercises file I/O, regular expression, hash table (hash map), and command arguments parsing. As the name suggests, it calculates the word counts from the input files or `stdin`. Here is the usage:

    Usage: wordcount [OPTIONS] [FILES]

    Options:
        -o:NAME             set output file name
        -i --ignore-case    ignore case
        -h --help           print this help menu

If we pipe the above usage to the program with `-i`, the output would be:

    2       case
    1       file
    1       files
    1       h
    2       help
    ...

### Nim version

The Nim version is straightforward. It uses [tables](http://nim-lang.org/tables.html#CountTable).`CountTable` to count words, [parseopt2](http://nim-lang.org/parseopt2.html).`getopt` to parse command arguments, and [sequtils](http://nim-lang.org/sequtils.html).`mapIt` for functional mapping operation. For regular expression, I chose [pegs](http://nim-lang.org/pegs.html) which the Nim documentation recommends over [re](http://nim-lang.org/re.html).

The `{.raises: [IOError].}` compile pragma at line 3 ensures the `doWork` proc only raises `IOError` exception. For this, I have to put `input.findAll(peg"\w+")` inside a `try` statement at line 21 to contain the exceptions it may theoretically raise.

Here is the code snippet from [wordcount.nim](https://github.com/arthurtw/nim-examples/blob/master/wordcount.nim):

{% highlight nim linenos %}
proc doWork(inFilenames: seq[string] = nil,
            outFilename: string = nil,
            ignoreCase: bool = false) {.raises: [IOError].} =
  # Open files
  var
    infiles: seq[File] = @[stdin]
    outfile: File = stdout
  if inFilenames != nil and inFilenames.len > 0:
    infiles = inFilenames.mapIt(File, (proc (filename: string): File =
      if not open(result, filename):
        raise newException(IOError, "Failed to open file: " & filename)
    )(it))
  if outFilename != nil and outFilename.len > 0 and not open(outfile, outFilename, fmWrite):
    raise newException(IOError, "Failed to open file: " & outFilename)

  # Parse words
  var counts = initCountTable[string]()
  for infile in infiles:
    for line in infile.lines:
      let input = if ignoreCase: line.tolower() else: line
      let words = try: input.findAll(peg"\w+") except: @[]
      for word in words:
        counts.inc(word)

  # Write counts
  var words = toSeq(counts.keys)
  sort(words, cmp)
  for word in words:
    outfile.writeln(counts[word], '\t', word)
{% endhighlight %}

### Rust version

To acquaint myself with Rust, I implemented a simple `BTreeMap` struct akin to `collections::BTreeMap`, but I ended up with using Rust’s `collections::HashMap` for fair comparison with Nim. (The code is still left there for your reference.) The [getopts](http://doc.rust-lang.org/getopts/getopts/index.html) crate is used to parse command arguments to my `Config` struct. Other parts should be straightforward.

Here is the code snippet from my [Rust wordcount](https://github.com/arthurtw/rust-examples/tree/master/wordcount) project:

{% highlight rust linenos %}
fn do_work(cfg: &config::Config) -> IoResult<()> {
    // Open input and output files
    let mut readers = vec![];
    if cfg.input.is_empty() {
        readers.push(BufferedReader::new(Box::new(io::stdin()) as Box<Reader>));
    } else {
        for name in cfg.input.iter() {
            let file = try!(File::open(&Path::new(name.as_slice())));
            readers.push(BufferedReader::new(Box::new(file) as Box<Reader>));
        }
    }
    let mut writer = match cfg.output {
        Some(ref name) => {
            let file = try!(File::create(&Path::new(name.as_slice())));
            Box::new(BufferedWriter::new(file)) as Box<Writer>
        }
        None => { Box::new(io::stdout()) as Box<Writer> }
    };

    // Parse words
    let mut map = collections::HashMap::<String, u32>::new();
    // let mut map = btree_map::BTreeMap::<String, u32>::new();
    let re = regex!(r"\w+");
    // let re = Regex::new(r"\w+").unwrap();
    // let re = regex!(r"[a-zA-Z0-9_]+");
    // let re = Regex::new(r"[a-zA-Z0-9_]+").unwrap();
    for reader in readers.iter_mut() {
        for line in reader.lines() {
            for caps in re.captures_iter(line.unwrap().as_slice()) {
                if let Some(cap) = caps.at(0) {
                    let word = match cfg.ignore_case {
                        true  => cap.to_ascii_lowercase(),
                        false => cap.to_string(),
                    };
                    match map.entry(word) {
                        Occupied(mut view) => { *view.get_mut() += 1; }
                        Vacant(view) => { view.insert(1); }
                    }
                }
            }
        }
    }

    // Write counts
    let mut words: Vec<&String> = map.keys().collect();
    words.sort();
    for word in words.iter() {
        if let Some(count) = map.get(*word) {
            try!(writeln!(writer, "{}\t{}", count, word));
        }
    }
    Ok(())
}
{% endhighlight %}

If you build the project by yourself, you may want to uncomment line 2 of [main.rs](https://github.com/arthurtw/rust-examples/blob/master/wordcount/src/main.rs) to disable the superflous warnings resulting from the “unstable” state of Rust 1.0 alpha libraries.

Zachary Dremann’s [pull request](https://github.com/arthurtw/rust-examples/pull/1/files) suggested using `find_iter`. I keep using `captures_iter` for consistency with the Nim version, but did refine my code a bit.

### Execution time comparison

I compiled the code with Nim’s `-d:release` and Rust’s `--release` flags. The sample 5MB input file was collected from Nim compiler’s C source files:

    $ cat c_code/3_3/*.c > /tmp/input.txt
    $ wc /tmp/input.txt
      217898  593776 5503592 /tmp/input.txt

The command to run the Nim version is like this: (Rust’s is similar)

    $ time ./wordcount -i -o:result.txt input.txt

Here is the result on my Mac mini with 2.3 GHz Intel Core i7 and 8 GB RAM: (1x = 0.88 seconds)

|           | Rust | regex! \w  | Regex \w  | regex! […] | Regex […] | Nim       |
----------- | ---- | ---------- | --------- | ---------- | --------- | --------- |
release, -i |      | **1x**     | **1.30x** | **0.44x**  | **1.14x** | **0.75x** |
release     |      | **1.07x**  | **1.33x** | **0.50x**  | **1.24x** | **0.73x** |
debug, -i   |      | 12.65x     | 20.14x    | 8.77x      | 19.42x    | 3.51x     |
debug       |      | 12.41x     | 20.09x    | 8.84x      | 19.33x    | 3.25x     |

Some notes:

1. Rust `regex!` runs faster than `Regex`, and `r"[a-zA-Z0-9_]+"` faster than `r"\w+"`. All 4 combinations were tested. (Uncomment line 21-23 above to try them.)
1. The “debug” version is just for your reference.
1. Nim only ran 1-2% slower with `--boundChecks:on`, so I didn’t include its result in this example.


## Example 2: Conway’s Game of Life

This example runs [Conway’s Game of Life](http://en.wikipedia.org/wiki/Conway%27s_Game_of_Life) on console with a fix-sized map and pattern. (To change the size or pattern, please edit the source code.) It uses [ANSI CSI code](http://en.wikipedia.org/wiki/ANSI_escape_code#CSI_codes) to redraw the screen.

The output screen looks like:

    . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . .
    . . . . . . . . . . . . . . . . . . . . . . . . (). (). . . . . . . . . . .
    . . . . . . . . . . . . . . . . . . . . . . (). . . (). . . . . . . . . . .
    . . . . . . . . . . . . . . (). . . . . . . (). . . . . . . . . . . . ()().
    . . . . . . . . . . . . . ()()()(). . . . (). . . . (). . . . . . . . ()().
    . ()(). . . . . . . . . ()(). (). (). . . . (). . . . . . . . . . . . . . .
    . ()(). . . . . . . . ()()(). (). . (). . . (). . . (). . . . . . . . . . .
    . . . . . . . . . . . . ()(). (). (). . . . . . (). (). . . . . . . . . . .
    . . . . . . . . . . . . . ()()()(). . . . . . . . . . . . . . . . . . . . .
    . . . . . . . . . . . . . . (). . . . . . . . . . . . . . . . . . . . . . .
    . . . . . . . . . . . . . . . . . . . . . . . . . (). . . . . . . . . . . .
    . . . . . . . . . . . . . . . . . . . . . . . (). (). . . . . . . . . . . .
    . . . . . . . . . . . . . . . . . . . . . . . . ()(). . . . . . . . . . . .
    . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . .
    . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . .
    . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . .
    . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . .
    . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . (). . . . .
    . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . (). . . .
    . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . ()()(). . . .
    . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . .
    n = 300   Press ENTER to exit

The program uses another thread to read `stdin`, and aborts the game when any line is read.

### Nim version

Here is the code snippet from my [Nim conway](https://github.com/arthurtw/nim-examples/tree/master/conway) project:

{% highlight nim linenos %}
type
  Cell = bool
  ConwayMap* = array[0.. <mapHeight, array[0.. <mapWidth, Cell]]

proc init*(map: var ConwayMap, pattern: openarray[string]) =
  ## Initialise the map.
  let
    ix = min(mapWidth, max(@pattern.mapIt(int, it.len)))
    iy = min(mapHeight, pattern.len)
    dx = int((mapWidth - ix) / 2)
    dy = int((mapHeight - iy) / 2)
  for y in 0.. <iy:
    for x in 0.. <ix:
      if x < pattern[y].len and pattern[y][x] notin Whitespace:
        map[y + dy][x + dx] = true

proc print*(map: ConwayMap) =
  ## Display the map.
  ansi.csi(AnsiOp.Clear)
  ansi.csi(AnsiOp.CursorPos, 1, 1)
  for row in map:
    for cell in row:
      let s = if cell: "()" else: ". "
      stdout.write(s)
    stdout.write("\n")

proc next*(map: var ConwayMap) =
  ## Iterate to next state.
  let oldmap = map
  for i in 0.. <mapHeight:
    for j in 0.. <mapWidth:
      var nlive = 0
      for i2 in max(i-1, 0)..min(i+1, mapHeight-1):
        for j2 in max(j-1, 0)..min(j+1, mapWidth-1):
          if oldmap[i2][j2] and (i2 != i or j2 != j): inc nlive
      if map[i][j]: map[i][j] = nlive >= 2 and nlive <= 3
      else: map[i][j] = nlive == 3
{% endhighlight %}

### Rust version

Here is the code snippet from my [Rust conway](https://github.com/arthurtw/rust-examples/tree/master/conway) project:

{% highlight rust linenos %}
type Cell = bool;

#[derive(Copy)]
pub struct Conway {
    map: [[Cell; MAP_WIDTH]; MAP_HEIGHT],
}

impl Conway {
    pub fn new() -> Conway {
        Conway {
            map: [[false; MAP_WIDTH]; MAP_HEIGHT],
        }
    }

    pub fn init(&mut self, pattern: &[&str]) {
        let h = pattern.len();
        let h0 = (MAP_HEIGHT - h) / 2;
        for i in 0..(h) {
            let row = pattern[i];
            let w = row.len();
            let w0 = (MAP_WIDTH - w) / 2;
            for j in 0..(w) {
                self.map[i + h0][j + w0] = row.char_at(j) == '1';
            }
        }
    }

    /// Iterate to next state. Return false if the state remains unchanged.
    pub fn next(&mut self) -> bool {
        let mut newmap = [[false; MAP_WIDTH]; MAP_HEIGHT];
        for i in 0..(MAP_HEIGHT) {
            for j in 0..(MAP_WIDTH) {
                let mut nlive = 0;
                for i2 in cmp::max(i-1, 0)..cmp::min(i+2, MAP_HEIGHT) {
                    for j2 in cmp::max(j-1, 0)..cmp::min(j+2, MAP_WIDTH) {
                        if self.map[i2][j2] && (i2 != i || j2 != j) {
                            nlive += 1;
                        }
                    }
                }
                newmap[i][j] = match (self.map[i][j], nlive) {
                    (true, 2) | (true, 3) => true,
                    (true, _) => false,
                    (false, 3) => true,
                    (false, _) => false,
                };
            }
        }
        // let changed = self.map != newmap;
        let changed = true;
        self.map = newmap;
        changed
    }
}

impl fmt::String for Conway {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        for row in self.map.iter() {
            for cell in row.iter() {
                try!(write!(f, "{}", if *cell { "()" } else { ". " }));
            }
            try!(write!(f, "\n"));
        }
        Ok(())
    }
}
{% endhighlight %}

In line 49, I intended to determine if the new map has changed, but it turns out a simple comparison `self.map != newmap` won’t work for array size > 32, unless you implement `PartialEq` trait.

Note that using unsafe `libc::exit` in my [main.rs](https://github.com/arthurtw/rust-examples/blob/master/conway/src/main.rs#L20) is very non-idiomatic in Rust. Zachary Dremann’s [pull request](https://github.com/arthurtw/rust-examples/pull/2/files) elegantly avoided the glaring `libc::exit` with `select!` and a non-blocking timer receiver. You may want to take a look.

### Execution time comparison

To measure the execution time, some code changes are needed:

1. Comment out the sleep function in [conway.nim](https://github.com/arthurtw/nim-examples/blob/master/conway/conway.nim#L31) and [main.rs](https://github.com/arthurtw/rust-examples/blob/master/conway/src/main.rs#L34).
1. Change the loop count from 300 to 30000.
1. As the map redraw is time-consuming, the measurement is taken both (1) with and (2) without map print (i.e. comment out the map print lines in [conway.nim](https://github.com/arthurtw/nim-examples/blob/master/conway/conway.nim#L29) and [main.rs](https://github.com/arthurtw/rust-examples/blob/master/conway/src/main.rs#L31,L32)).

Here is the result with Nim’s `-d:release` and Rust’s `--release` flags:

|                     | Rust | Nim  | Nim/bc:on   | n=30000 |
--------------------- | ---- | ---- | ----- | ------- |
(1) with map print    | **1x** | **1.75x** | **1.87x** | 1x=3.33s |
(2) without map print | **1x** | **1.15x** | **1.72x** | 1x=0.78s |

(Since Rust does bounds checking, to be fair, I added a column “Nim/bc:on” for Nim binaries compiled with an additional `--boundChecks:on` flag.)

## Nim vs. Rust

Although Nim and Rust are both compiled languages aiming for good performance, they are two very different languages. Their limited similarities I can think up include:

- compiled and statically typed
- aiming for good performance (either could run faster depending on the underlying implementations, and further optimizations are probable)
- composition over inheritance (this seems a trend of new languages?)
- easy C binding
- popular language goodies: generics, closure, functional features, type inference, macro, statement as expression…

But their differences are more interesting.

### Philosophy: freedom vs. discipline

Coding in Nim often gives me an illusion of scripting languages. It really blurs the line. Nim tries to remove the coding friction as much as possible, and writing in Nim is a great joy.

However, there is a flip side when leaning toward the writer’s freedom too much: the explicitness, clarity or even maintainability could be hampered. Let me give a small example: in Nim, `import` will bring in all module symbols to your namespace. A symbol of a module *can* be qualified with `module.symbol` syntax, or you can use `from module import nil` to force qualification, but really, who will bother doing this? And this seems not “idiomatic Nim” anyway. The result is, you often cannot tell which symbol comes from which module when reading others’ (or *your*) code. (Fortunately, there won’t be naming collision bugs because Nim compiler forces you to qualify in such cases.)

Other examples include: UFCS that lets you use `len(x)`, `len x`, `x.len()` or `x.len` freely, whichever you like; underline- and case-insensitive, so `mapWidth`, `mapwidth` and `map_width` all map to the same name (I’m glad they enabled the “partial case sensitivity” rule in 0.10.2, therefore `Foo` and `foo` at least differ); use of uninitialised values is OK; etc. In theory, you can follow strict style guidelines to mitigate the issue, but you tend to be more relaxed when coding in Nim.

On the other hand, Rust honors discipline. Its compiler is very rigid. Everything must be crystal clear. You have to get a lot of things right beforehand. Ambiguity is not an attribute of Rust code... Things like these are usually good for long-lived projects and maintainability, but coding in Rust may feel restrictive and force you to take care of some details you’re not interested of. You are also tempted into memory or performance optimisation even it’s not your priority. You tend to be more disciplined when coding in Rust.

Both have their pros and cons. As a coder, I enjoy coding in Nim more; as a maintainer, I would rather maintain products written in Rust.

### Visual style: Python-ish vs. C++-ish

Like Python, Nim uses indentation to delimit blocks and has less sigils. Rust is more like C++. Those `{}`, `::`, `<>` and `&` should be familiar to C++ programmers, plus some new ones like `'a`.

Occasionally, Nim can be too literal. For example, I think Rust’s `match` syntax:

{% highlight rust %}
match key.cmp(&node.key) {
    Less    => return insert(&mut node.left, key, value),
    Greater => return insert(&mut node.right, key, value),
    Equal   => node.value = value,
}
{% endhighlight %}

looks clearer than Nim’s `case` statement:
{% highlight nim %}
case key
of "help", "h": echo usageString
of "ignore-case", "i": ignoreCase = true
of "o": outFilename = val
else: discard
{% endhighlight %}

But overall, Nim code is less visually noisy. Rust lifetime parameters are particularly cluttering IMO and, unfortunately, unlikely to change since it has passed 1.0 Alpha.

### Memory management: GC vs. manual safety

Though Nim allows unsafe (untraced) memory management and provides soft realtime support with more predictable GC behaviour, it’s still a GC language, having all the [pros and cons](http://en.wikipedia.org/wiki/Garbage_collection_%28computer_science%29#Advantages) of GC. Nim’s object assignment is value copy. If you can live with GC, Nim memory management should be effortless to you.

Rust provides limited GC support, but more often than not, you’ll rely on Rust’s ownership system for memory management. The abstraction is so thin that you are basically managing the memory on your own. As a Rust programmer, you need to fully comprehend its memory management model (ownership, borrow and lifetimes) before you can code in Rust effectively, which is often the first barrier inflicting Rust newcomers.

On the other hand, it’s Rust’s most unique strength: safe memory management without GC. Rust solved this tough challenge beautifully. This along with resource safety, concurrent data safety and the elimination of null pointers makes Rust an extremely safe, reliable, low (or even no) runtime-overhead programming language.

Depending on your requirements, Nim’s GC can be good enough, or Rust is your only sensible choice.

### Other differences

Nim’s strengths:

- Productivity: within the same unit of time, you can crank out more features in Nim
- Ease of learning
- Scripting in a compiled language, good for prototyping, interactive exploration, batch processing etc.
- Language features:
  - method overloading
  - new operator definition
  - named arguments and default values
  - powerful compile-time coding

Rust’s strengths:

- A true systems programming language: embeddable, GC-free and bare metal
- Safety, correctness, runtime reliability
- Strong core team and vibrant community
- Language features:
  - pattern matching (excellent!)
  - enum variants (though Nim’s object variants are good, too)
  - `let mut` instead of `var` (small thing but matters)
  - powerful destructuring syntax

Error handling: Nim opts for the common exception mechanism while Rust uses a flexible `Result` type (and `panic!`). I have no strong preference here, but consider it an important difference worth mentioning.

### Release 1.0 is on the way

Both [Nim](http://forum.nim-lang.org/t/698) and [Rust](http://blog.rust-lang.org/2014/12/12/1.0-Timeline.html) target 1.0 release at the first half of this year. That’s very exciting! Rust has gained a lot of attention and Nim is becoming better-known, too. Their flavors are very different, but both are great new programming languages. Rust shines when safety, performance or reliability matters. Nim is nimble, expressive, blending the strengths of scripting languages and compiled languages well. They should be great additions to your language selection.

Hope this article has given you a glimpse on them.
