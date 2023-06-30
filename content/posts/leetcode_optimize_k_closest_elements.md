---
title: "Non-algorithm optimization of K closest elements Java solution"
type: "post"
date: 2023-03-21T07:07:17+05:30
draft: false
tags: [leetcode, java, performance]
---

I am a weird creature in my friend circle for using Java to do DSA problems. (Everyone else uses C++). I am not that good anyway in Leetcode, so at least I will learn some more Java by doing this.

Here's the puzzle I was attempting to solve: Given a sorted array and an integer `x`, find k elements closest to `x` in the array ([Leetcode #658](https://leetcode.com/problems/find-k-closest-elements/)).

The solution is very straightforward: find the lower bound or upper bound of the element in the array using built-in binary search, then expand the range using 2 indexes.

This was my initial solution:

```java
class Solution {
    public List<Integer> findClosestElements(int[] arr, int k, int x) {
        int pos = Arrays.binarySearch(arr, x);
        int left = -1, right = -1;

		if (pos >= 0) {
            k--;
            left = pos-1;
            right = pos+1;
        } else {
            int insPos = -(pos + 1);
            left = insPos-1;
            right = insPos;
        }

        while (k != 0) {
            if (left < 0) {
                right++;
            } else if (right >= arr.length) {
                left--;
            } else {
                if (Math.abs(arr[left] - x) <= Math.abs(arr[right] - x)) {
                    left--;
                } else {
                    right++;
                }
            }
            k--;
        }
        var list = new ArrayList<Integer>();
        for (int i = left+1; i < right; i++) {
            list.add(arr[i]);
        }
        return list;
    }
}
```

Pretty straightforward, right? This beats 60% Java solutions in runtime, (and 70% in memory). These numbers aren't actually reliable at this point, when your solution is very similar to others' solutions. But these indicate that you can optimize the solution further [^fn1].

So here's my second attempt. I pre-allocated the result list, and also recalculated left / right values only when the index changed. (I don't think the latter has much impact, except the noise it adds).

```java
class Solution {
    public List<Integer> findClosestElements(int[] arr, int k, int x) {
        int pos = Arrays.binarySearch(arr, x);
        int left = -1, right = -1;
        if (pos >= 0) {
            k--;
            left = pos-1;
            right = pos+1;
        } else {
            int insPos = -(pos + 1);
            left = insPos-1;
            right = insPos;
        }
        int len = arr.length;
        int absLeft = 0, absRight = 0;
        if (left >= 0) absLeft = Math.abs(arr[left] - x);
        if (right < len) absRight = Math.abs(arr[right] - x);
        while (k != 0) {
            if (left < 0) {
                right++;
                if (right < len) absRight = Math.abs(arr[right] - x);
            } else if (right >= arr.length) {
                left--;
                if (left >= 0) absLeft = Math.abs(arr[left] - x);
            } else {
                if (absLeft <= absRight) {
                    left--;
                    if (left >= 0) absLeft = Math.abs(arr[left] - x);
                } else {
                    right++;
                    if (right < len) absRight = Math.abs(arr[right] - x);
                }
            }
            k--;
        }
        var list = new ArrayList<Integer>(right - left - 1);
        for (int i = left+1; i < right; i++) {
            list.add(arr[i]);
        }
        return list;
    }
}
```

Runtime: 84 %ile, memory: 81 %ile. This was obvious optimization, can we do better?

At this point you'd have observed the function returns a `List<Integer>`, whose out-of-the-box implementation (`ArrayList`), turns out to be grossly inefficient than a contiguous list of simple value types. At this point I wondered if a `List<Integer>` backed by `int[]` can be created with a one-liner or two.

But before that, I tried another optimization. Instead of using `List<Integer>::add`, I created a `Integer[]` (still reference type), added values to this array, and lastly returned a array-backed list using `Arrays.asList`.

```java
class Solution {
    public List<Integer> findClosestElements(int[] arr, int k, int x) {
        int pos = Arrays.binarySearch(arr, x);
        int left = -1, right = -1;
        if (pos >= 0) {
            k--;
            left = pos-1;
            right = pos+1;
        } else {
            int insPos = -(pos + 1);
            left = insPos-1;
            right = insPos;
        }
        int len = arr.length;
        int absLeft = 0, absRight = 0;
        if (left >= 0) absLeft = Math.abs(arr[left] - x);
        if (right < len) absRight = Math.abs(arr[right] - x);
        while (k != 0) {
            if (left < 0) {
                right++;
                if (right < len) absRight = Math.abs(arr[right] - x);
            } else if (right >= arr.length) {
                left--;
                if (left >= 0) absLeft = Math.abs(arr[left] - x);
            } else {
                if (absLeft <= absRight) {
                    left--;
                    if (left >= 0) absLeft = Math.abs(arr[left] - x);
                } else {
                    right++;
                    if (right < len) absRight = Math.abs(arr[right] - x);
                }
            }
            k--;
        }
        var list = new Integer[right - left - 1];
        int s = left+1;
        for (int i = left+1; i < right; i++) {
            list[i-s] = arr[i];
        }
        return Arrays.asList(list);
    }
}
```

That gets us to consistently 99%ile of runtime, and little less than 90%ile memory.

Can we do better?

I tried inlining `Math.abs` calls - no use. Probably JVM does that already.

Okay I give up. I look at top 99.97 %ile at the submission runtime bar chart.

This is the _"sample 0ms solution"_ (Although I guess there's only one sample). However leetcode doesn't show who wrote this.

```java
import java.util.*;

class Solution {
    public List<Integer> findClosestElements(int[] arr, int k, int x) {
        int left = 0;
        int right = arr.length-k;
        while(left<right) {
            int mid = (left+right) >> 1;
            if(x-arr[mid]>arr[mid+k]-x) left = mid+1;
            else right = mid;
        }
        return new MyList(arr, left, left+k);
    }

    class MyList extends AbstractList<Integer> {
        private int[] arr;
        private int left;
        private int right;

        public MyList(int[] arr, int left, int right) {
            this.arr = arr;
            this.left = left;
            this.right = right;
        }

        @Override
        public Integer get(int index) {
            return arr[left+index];
        }

        @Override
        public int size() {
            return right-left;
        }
    }
}
```

Here's the trick. `MyList` doesn't cause any allocations, because it wraps the source array only, and does minimal overrides to expose a `List<Integer>` interface.

So I learned about a new type in collections framework: `AbstractList`. The documentation says this type can be used to minimally implement a list backed by a random access source.

---

[^fn1]: Back when I was doing DSA puzzles in C++, I used to change `std::string` in function signatures to `std::string&`, and see runtime reduce drastically, since a significant portion of time was probably spent copying the string to solution method.

