# Discussion Topic

How understanding the process of finding duplicates in an integer array leads to
a good understanding of Bloom filters.

# Notes

One of the very basic and fundamental question that one codes for in any project
is to answer the following question:

**Does a given element/object exist in a data-store?**

The above question can be read in context of integer arrays or for objects such as
checking if a user exists in our system for successful authentication, if a given
store item exists (in quantity) for online retail purchase, and so on. As this
code piece usually becomes hot (called a lot many times during the usage of the
system), the faster we can answer the question above, the faster our overall system
would be.

## Contains check or finding duplicates in integer array

Let's pick up an integer array with N integers. The integers are bounded by the
theory of computing to be between a given range. We can create a `bit-array`
(also called as bitsets and bit-vectors) of length N. For every integer encountered
within the array, set the bit at the index corresponding to value of integer to
`true`. Once the array iteration and `bit-array` population is complete, we can
answer the question of `contains` in `O(1)` by just checking if the bit at the
index corresponding to integer being checked is `true` or `false`.

As integers are bounded, the above works in constant space and time. For example
if we have a total of 1 billion numbers, the total space required is **128MB of
RAM**. If we start using sparsed-bit-arrays, this space can be reduced further.

## Strings instead of integers

Now let's consider that instead of integers we had strings to be checked for
presence. The only piece unsolved in this challenge now remains on how to map
strings to integers. If we can devise a way to map strings to integers on a `1:1`
basis, our problem is solved.

We know that hashing a string can give us an integer (`int` or `long` as in Java).
If we hash a given string using a really good hash function (that distributes
strings equally over the entire range space) then we can answer the contains check
with the same probability as the probability of hash-collision of the hash
function chosen.

Thus, our problem of answering contains is solved but with an error probability
introduced in. Our methodology can now answer the contains check with the
following two conditions:

* The algorithm can tell if an element is **not present** with **100%** confidence
* If the algorithm says that an element is present, there is a slight chance of error

The above data-structure that satisfies the above two rules, and allows us to
configure the error probability in lieu of storage space is called a **BLOOM
FILTER**.

## Hash functions and storage space

As discussed above, the error probability in a bloom filter is as good as the
collision probability of the hash function used. Most of the awesome hash functions
like MD5, Murmur2, Murmur3 etc generate an value greater than the integers `32-bit`
storage space. Like Murmur3 generates a `128-bit` value.

If we were to still use the full integer space for storage, we can divide the
above `128-bit` value into `4 32-bit` values. Thus, we can keep 4-different bit-sets
to answer the contains check matching the probability of Murmur3 hash function.
However, this leads to an increase in storage space to `1 GB of RAM` for 1-billion
elements. If we use the same bit-set to mark indexes for the `4 32-bit` values above, the
probability of error increases from that of hash-collision function to approx 4
times (in lay-man computational terms).

This leads us to a choice on the accuracy that we need against the storage space
that we need.

## Additional Reading

* [Bit-Vectors](https://en.wikipedia.org/wiki/Bit_array)
* [Hashing](https://en.wikipedia.org/wiki/Hash_function)
* [Bloom Filters](https://en.wikipedia.org/wiki/Bloom_filter)
