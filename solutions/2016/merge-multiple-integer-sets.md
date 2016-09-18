# Problem

You are given multiple integer sets and you need to merge them into a single set in the fastest possible
way.

# Solution

Suppose we have 3 arrays of integer sets and we need to merge them. The fastest solution would be possible
in time of `O(n1 + n2 + n3)` where `n1`, `n2`, `n3` are the lengths of the three sets respectively. The solution
lies in using a bit-vector (also called as bit-set or a bit-array) to represent elements using the index
and then iterating over them.

* Construct a bit-array
* Iterate over the first array and for each integer set the corresponding bit to true
* Repeat the above process for remaining arrays
* Any duplicate element will result in setting `true` an already set bit
* The resultant bit-array is the merge of all the arrays

```java
public void mergeArrays(int[] array1, int[] array2, int[] array3) {
  BitSet bitSet = new BitSet();

  // start merging
  for(int num : array1) {
    bitSet.setBit(num, true);
  }
  for(int num : array2) {
    bitSet.setBit(num, true);
  }
  for(int num : array3) {
    bitSet.setBit(num, true);
  }

  // output
  int index = -1;
  do {
    index = bitSet.getNextSetBit(index);
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
