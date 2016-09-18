# Problem

You are given multiple integer sets and you need to find duplicates in them, ie. you need to find the intersection
of all the sets.

# Solution

Suppose we have 3 arrays of integer sets and we need to merge them. The fastest solution would be possible
in time of `O(n1 + n2 + n3)` where `n1`, `n2`, `n3` are the lengths of the three sets respectively. The solution
lies in using two bit-vectors (also called as bit-set or a bit-array) to represent intersection between any two
sets and then using the resultant to intersect with the next one, and so on.

* Construct a running bit-set and populate it with the first array
* Using a second bit-set intersect the first bit-set with second array
* Now change-over the second to the first bit-set
* And repeat the process above with the third array, fourth array and so on

```java
public void findDuplicates(int[] array1, int[] array2, int[] array3) {
  BitSet first = new BitSet();
  BitSet result = new BitSet();

  // populate the initial one
  for(int num : array1) {
    first.setBit(num, true);
  }

  // intersect with second
  for(int num : array2) {
    if(first.isBitSet(num)) {
      result.setBit(num);
    }
  }

  // change-over
  first = result;
  result = new BitSet();

  // intersect with third
  for(int num : array3) {
    if(first.isBitSet(num)) {
      result.setBit(num);
    }
  }

  // output
  int index = -1;
  do {
    index = result.getNextSetBit(index);
    if(index < 0) {
      break;
    }

    System.out.println(index);
  } while(true);
}
```

The solution above can be extended to as many arrays as are provided in the problem definition. The time to
sort will still remain `O(N)` where `N` is the sum of total number of elements across all provided arrays.

## Optimizations available

* One can make use of sparsed-bit-arrays to reduce memory consumption. Refer to [brettwooldridge/SparseBitSet]
(https://github.com/brettwooldridge/SparseBitSet) for one such implementation.

* If the arrays are really, really huge - an implementation that uses file-based persistence of a bit-array
can be used. Refer to [one such implementation](https://github.com/sangupta/jerry-core/blob/master/src/main/java/com/sangupta/jerry/ds/bitarray/MMapFileBackedBitArray.java)
available in the [jerry-core](https://github.com/sangupta/jerry-core) project.
