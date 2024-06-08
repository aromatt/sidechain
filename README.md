# sidechain

Split and merge Unix pipelines for filtering and data manipulation.

## "Sidechain?"

The name and concept are borrowed from an audio production technique in which an
effect on one audio signal is controlled by another audio signal. For example, you
might run a bass guitar through a compressor triggered by a kick drum.

Similarly, in a Unix pipeline, we can use `sidechain` to control our critical path
using a secondary data stream.

## Filter Mode
Imagine you have TSV data like this:
```txt
bob	{"foo":1}
alice	{"foo":0}
```
You need to print a list of all users that have `"foo" > 0`. How would you do it?

It's easy enough to do `cut -f2 | jq 'select(.foo > 0)'`, but this is no good because
`cut` removes the user names.

`sidechain` makes this easy:

```bash
cat input.tsv \
  | sidechain filter 'cut -f2 | jq "select(.foo > 0)"' \
  | cut -f1
```

### A more realistic example
You have millions of files names like `20231101_1.2.3.0-24.csv.gz`. The `1.2.3.0-24`
part represents a CIDR (`1.2.3.0/24`). You're interested in a particular set of
CIDRs, but they might be child/parent CIDRs of those in the filenames.

You have a tool that can efficiently filter a list of input CIDRs given a set of
query CIDRs, but you'd need to remove the date prefixes and `.csv` suffixes in order
to use it. This is a problem; you won't be able to reconstruct the full filenames
without the date prefixes.

```bash
ls /path/to/files/ | sidechain filter 'sed <extract CIDR> | filter-cidrs'
```

### Improving performance with `-t`
When you specify `-t <char>`, you agree to make your filter print exactly one line
per input line. Print `<char>` to indicate that the input line passed the filter.

Here's the first example again using `-t`:

```bash
cat input.tsv \
  | sidechain filter -t1 'cut -f2 | jq "if .foo > 0 then 1 else 0 end"' \
  | cut -f1
```

This provides a valuable hint to `sidechain`: no extra work is required to correctly
pair your filter's output lines with its input lines.

## Map Mode
Suppose you have a file containing lines of JSON with an `"ip"` field, but some of
the "IPs" are actually URLs. You want to clean up this data. You have a tool called
`extract-ips` which can perform the IP extraction
([cidrq](https://github.com/aromatt/cidrq) can actually do this!).

```bash
cat input.json | sidechain map -I% 'jq .ip | extract-ips' jq '.ip = "%"'
```

Note: this does NOT invoke `jq` for every input line. Each use of `jq` in the above
command is invoked only once, processing all of your input before exiting.

### Using `$[]` for process substitution
For a cleaner, more-intuitive incantation, you can use `$[]` to wrap your
value-generating command:

```bash
cat input.json | sidechain map jq '.ip = "$[jq .ip | extract-ips]"'
```

### Mapping from a file
Alternatively, you can provide a file containing the values that you want to insert:
```bash
cat input.json | sidechain map -I% -f ips.txt jq '.ip = "%"'
```
