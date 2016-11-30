# Problem

Sort a given a huge array of bounded, non-duplicate, positive integers. Extend
the problem to include negative integers. What are the different ways to reduce
memory complexity of the problem.

Notes for clarity:

* The reason we are calling the array bounded is for the reason of allocation of
memory in the first example.


## Updates since publishing:

* Fixed bug in boolean array size as pointed by [ploggingdev](https://news.ycombinator.com/user?id=ploggingdev) - thanks!
* Added notes on similarity with BucketSort
* Updated post title to indicate the domain to be bounded

# Solution

Most of the interview candidates that I have talked to about this problem come up
with the quick answer as `MergeSort` or divide-and-conquer. The cost of sorting
being `O(N * Log(N))` - which in this case is not the fastest sorting time.

The fastest time to sort an integer array is `O(N)`. Let me explain how.

* Construct a boolean array of length N
* For every integer `n` in array, mark the boolean at index `n` in array as `true`
* Once the array iteration is complete, just iterate over the boolean array again
to print all the indices that have the value set as `true`

```java
public void sortBoundedIntegers(int[] array) {
  /// the regular checks first
  if(array == null) {
    return;
  }

  if(array.length == 0) {
    return;
  }

  // build the boolean result array
  // Integer.MAX_INT makes sure that all integer values can be accommodated
  // this is very memory in-efficient and thus one should ideally use
  // a sparse bit-array for space savings
  boolean[] result = new boolean[Integer.MAX_INT];

  // iterate
  for(int index = 0; index < array.length; index++) {
    int num = array[index];
    result[num] = true;
  }

  // print the result
  for(int index = 0; index < result.length; index++) {
    if(result[index]) {
      System.out.println(index);
    }
  }
}
```

**Note:** The above solution using a boolean array is not the optimized solution
and is meant only for clarity of algorithm. The ideal solution would be to use a
`sparse bit-array`. Please go over [Issue #4](https://github.com/sangupta/ps/issues/4)
for some discussion.

## Additional constraints and optimizations available

* A `boolean` occupies one-byte of memory. Thus, switching to a bit-vector (also
called as bit-array) will reduce the memory consumption by a factor of 8. Check
code sample 2.

* In case the integers are also negative, another bit-array can be used to check
for negatives, and then both iterated one-after-another to produce result. Check
code sample 2.

* To further reduce the memory consumption, one can make use of sparsed-bit-arrays.
This can lead to huge drop in memory consumption if the integers are spaced apart
a lot. Check code sample 2. Refer to [brettwooldridge/SparseBitSet](https://github.com/brettwooldridge/SparseBitSet)
for one such implementation.

* In case the integer array contains duplicates, use a small `short` array than
the `boolean` array to hold the number of times an integer has been seen, thus
still sorting in `O(N)` time.

* If we use sparse bit-arrays on the above code example, then the part of the
problem becomes similar to [BucketSort](https://en.wikipedia.org/wiki/Bucket_sort)
where the buckets/bins are created to reduce memory space of the problem. However,
the individual bin-sorting is based on bits in the bit-array.

### Code Sample 2

```java
public void sortBoundedIntegers(int[] array) {
  // run regular checks

  final int len = array.length;

  // considering SparsedBitSet is an implementation available
  BitSet negative = new SparsedBitSet();
  BitSet positive = new SparsedBitSet();

  // sort
  for(int index = 0; index < len; index++) {
    int num = array[index];
    if(num < 0) {
      num = 0 - num; // convert to positive
      negative.setBit(num, true);
    } else {
      positive.setBit(num, true);
    }
  }

  // output
  int index = -1;
  do {
    index = negative.getNextSetBit(index);
    if(index < 0) {
      break;
    }

    System.out.println(0 - index);
  } while(true);

  index = -1;
  do {
    index = positive.getNextSetBit(index);
    if(index < 0) {
      break;
    }

    System.out.println(index);
  } while(true);
}
```

# Additional Reading

* [Bit-Arrays](https://en.wikipedia.org/wiki/Bit_array)
* [Hashing](https://en.wikipedia.org/wiki/Hash_function)
* [Bloom Filters](https://en.wikipedia.org/wiki/Bloom_filter)
* [BucketSort](https://en.wikipedia.org/wiki/Bucket_sort)
* [Python implementation of Countingsort, Bucketsort and Radixsort](https://github.com/MauriceGit/Advanced_Algorithms)
* [**Integer Sorting in 0(n sqrt (log log n)) Expected Time and Linear Space**](http://dl.acm.org/citation.cfm?id=652131), Yijie Han and Mikkel Thorup, FOCS '02 Proceedings of the 43rd Symposium on Foundations of Computer Science Pages 135-144
