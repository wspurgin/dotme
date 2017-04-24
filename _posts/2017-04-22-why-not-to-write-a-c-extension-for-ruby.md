---
layout: post
title:  "Why not to write a C extensiion for Ruby"
date:   2017-04-22 22:22:22 -0500
categories: ruby
---

C extensions are certainly cool, and in many instances they *are* faster than
same approach Ruby approach. However, they're a pain to maintain, resources for
writing good C code that still use proper C-Ruby paradigms are sparse, and (most
of all) you're probably trying to hit a fly with a hammer.

🐝…🔨

In reality, writing a C extension for Ruby should be more of a last resort than
a goto. In fact, as [Aaron Patterson][tenderlove] has shown us, there are
situations were [writing it in Ruby produces better results than C][msgpack-135]
(and [avoids SegFaults][every-thing-in-c] to boot).

To give you an anecdotal example, recently, I was attempting to do fuzzy
string matching using the names of Entities. My goal was to select the Entity
closet to the name I was given. This is an old problem with lots of solutions,
so I settled on using a tried and tested method called the edit distance (or
[Levenshtein Distance][levenshtein] as it's formally known). There was already a
"[gem for that][]" so I didn't think I'd even have to bother with anything
complicated. However, there was a slight hiccup. The names of the Entities I was
dealing with were occasionally in other languages than English (i.e. in
Unicode). This is not an issue for Ruby because it encodes everything in UTF-8
(which covers all of Unicode and is backwards compatible with ASCII). The
downside was that the gem I used (and every derivative I found) all implemented
the Levenshtein algorithm(s) in a C extension, and C's standard string
operations are [historically biased][unicode-in-c] towards ASCII based strings.
For example, a `char` type is always one byte (8 bits), the number of bits
required to represent any ASCII character. Unicode characters are multi-byte
character strings. This meant that trying to compare the distance between any
string and a Unicode string produce bad results (and in some cases bad C-level
errors). Go ahead, try it:

```ruby
# After gem install levenshtein
require "levenshtein"

uni_str       = "🐝🔨"
other_uni_str = "🔨🐝"

Levenshtein.distance_fast(uni_str, other_uni_str, nil)
#=> ArgumentError: malloc: possible integer overflow (-6299234123000012815*4)
```

That's because when getting the length of the strings, C does some magic
calculations so that it doesn't have to actually iterate over the string to
determine its length. However, these magic calculations were based on ASCII
single byte character strings not UTF-8 1-or-4 byte character strings.

Thus I got it into my head to write a Levenshtein Distance gem that supported
UTF-8 encoding **in native C**. I spent several hours managing memory,
researching Unicode support in C, and how to bundle that all into a C extension
in a Ruby gem.

In the midst of all this, my friend [zorab47](https://github.com/zorab47) walks
in and asks:

> zorab: What are you doing?
>
> me: Writing a C extension for Ruby.
>
> zorab: Why?
>
> me: For fuzzy matching the names of those Entities.
>
> zorab: No, I mean why are you writing a C extension?
>
> me: Because it'd be faster than doing it in Ruby?
>
> zorab: … Hold on

In a few moments, zorab comes back with the results of two Google search results
that rendered my several hours of toil nil. As you might have guessed, these
Entities were a model in a Ruby on Rails application, and represented a table
within a PostgreSQL database. The two things zorab came back with were the
PostgreSQL extensions [fuzzystrmatch][] and [pgtrgm][]. I ended up using pgtrgm
to handle the comparison between Entities' names, and it was

1. Much faster
2. Not in my code base
3. More scalable (in my situation)
4. **Not** a C extension

The moral of this story is: why are you writing a C extension? They have there
uses, but it's quite probable that there are easier ways to do what you want to
do that don't involve you having to write and maintain C. Anything from an
algorithm change to an architecture change (i.e. where in your stack you're
applying your algorithm).

Learning some of the ins-and-outs of C extensions was still interesting, so if
you still really want to write a C extension, I'll be posting soon with some
tips and resources I've found through my own experience of writing one.

[tenderlove]:   https://twitter.com/tenderlove
[msgpack-135]:  https://github.com/msgpack/msgpack-ruby/pull/135
[every-thing-in-c]:   https://twitter.com/tenderlove/status/818492252964040705
[levenshtein]:  https://en.wikipedia.org/wiki/Levenshtein_distance
[gem for that]: https://rubygems.org/gems/levenshtein
[unicode-in-c]: http://www.cprogramming.com/tutorial/unicode.html
[fuzzystrmatch]: http://devdocs.io/postgresql~9.4/fuzzystrmatch
[pgtrgm]:   http://devdocs.io/postgresql~9.4/pgtrgm
