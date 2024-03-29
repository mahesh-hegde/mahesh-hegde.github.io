---
title: "Leetcode #995: Minimum number of K bit flips"
type: "post"
date: 2023-03-20T11:03:56+05:30
draft: false
tags: [Leetcode, Algorithms]
---

This is my solution and logic for [Problem 995](https://leetcode.com/problems/minimum-number-of-k-consecutive-bit-flips/) on Leetcode. This is actually copy of solution I posted on discussion forums.

I find it a little hard to get the initial intuition & optimization from `O(nk)` to `O(n)`, and as usual, other solution posts are very terse. So I thought it is worth to explain a little more.

## Problem statement
> You are given a binary array nums and an integer `k`.

> A `k-bit flip` is choosing a subarray of length k from nums and simultaneously changing every 0 in the subarray to 1, and every 1 in the subarray to 0.

> Return the minimum number of k-bit flips required so that there is no 0 in the array. If it is not possible, return -1.

## Intuition

Observation 1: bit flips are commutative. It doesn't matter in which order you flip the subarrays, the end result should be same.

So there must be a valid sequence of bit flips in an increasing order through the array.

We must find this increasing order of flips.

Observation 2: when we do a pass to find these flips, if our sliding window of size `k` has a `0` in beginning, it must be flipped. Because we are going in increasing order and it's our last chance to flip that integer.

Observation 3: If the bit at the beginning of sliding window is 1, we must __not__ flip that bit, because we can't flip that ever again.

From (2) and (3), whether we flip a certain sliding window depends only on the first bit.

# Approach
Straightforward `O(nk)` approach: keep a sliding window of size `k`, if it's first bit is 0, flip all subsequent bits in that window.

This actually passes 110/113 test cases!

To not actually have to flip the bits for every single window, we can lazily compute the state of each cell when we encounter it. I do it by marking some points as "turning points" (my own term).

In particular, when I encounter a window I want to flip, I'd rather invert a boolean state `flip`, and mark the end of this window (actually one past the end), as turning point. (I use queue to remember turning points, but you can use any kind of linear DS.)

When I see a turning point at a future position, I pop that turning point from queue, flip the current state and continue the usual logic.

# Complexity

__Time complexity__: `O(nk)`, `O(n)` respectively

__Space complexity__: There can be at most `k` turning points in queue at a time, (haven't calculated more precisely). so `O(k)`.

# Code

## Unoptimized `O(n*k)`

This passed 110/113 test cases, if I remember correctly.

```java
class Solution {
    public int minKBitFlips(int[] nums, int k) {
        int ans = 0;
        for (int i = 0; i < nums.length - k + 1; i++) {
            int bit = nums[i];
            if (bit == 0) {
                // mark [i, i+k) as flipped
                for (int j = 0; j < k; j++) {
                    nums[i+j] = 1 - nums[i+j];
                }
                ans++;
            }
        }
        for (int i = nums.length - k + 1; i < nums.length; i++) {
            if (nums[i] == 0) return -1;
        }
        return ans;
    }
}
```

## Optimized `O(n)`

```java
class Solution {
    public int minKBitFlips(int[] nums, int k) {
        int ans = 0;
        boolean flip = false;
        Queue<Integer> turns = new ArrayDeque<Integer>();
        for (int i = 0; i < nums.length - k + 1; i++) {
            int bit = nums[i];

            if (!turns.isEmpty() && i == turns.peek()) {
                flip = !flip;
                turns.remove();
            }

            if (flip) {
                bit = 1 - bit;
            }

            if (bit == 0) {
                turns.add(i+k);
                flip = !flip;
                ans++;
            }
        }

        for (int i = nums.length - k + 1; i < nums.length; i++) {
            int bit = nums[i];

            if (!turns.isEmpty() && i == turns.peek()) {
                flip = !flip;
                turns.remove();
            }

            if (flip) bit = 1 - bit;
            if (bit == 0) return -1;
        }

        return ans;
    }
}
```

