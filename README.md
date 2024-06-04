# sidechain

Filter lines of text based on a parallel subprocess, preserving the original lines.

## "Sidechain?"

The name and concept are borrowed from an audio production technique in which an
effect on one audio signal is controlled by another audio signal. For example, you
might run a bass guitar through a compressor triggered by a kick drum.

## Example
Imagine you have TSV data like this:
```
bob	{"foo":1}
alice	{"foo":0}
```
You need to find all users that have `"foo" > 0`. How would you do it?

It's easy enough to do `cut -f2 | jq 'select(.foo > 0)'`, but this won't work because
`cut` chops off the user names.

Sidechain makes this easy:
```
cat data.tsv | sidechain 'cut -f2 | jq \'select(.foo > 0)\' | cut -f1
```

### More examples

* You want to find all lines in a file that contain valid IPs. You have a tool that
  can validate IPs, but it provides very few options for specifying where the IPs are
  in each line. And worse: *some* of the IPs have a `:<port>` suffix.
