# 50 LeetCode Problems — DSA Interview Prep (Python)
> Easy & Medium | Patterns: Arrays, Strings, Hashing, Binary Search, Trees, Graphs
> Practice 2 problems/day. Read the explanation first, then code it yourself before looking at the answer.

---

## HOW TO USE THIS FILE

1. Read the **Problem** and **Example**
2. Read the **Key Insight** — this is the pattern to recognize
3. Try to code it yourself (set a 20-min timer)
4. Only then look at the **Solution**
5. Read the **Why It Works** section to lock in understanding

---

# SECTION 1 — ARRAYS & TWO POINTERS (Problems 1–10)

---

## Problem 1 — Two Sum (Easy)
**LeetCode #1**

### Problem
Given an array of integers `nums` and an integer `target`, return the indices of the two numbers that add up to `target`. You may assume exactly one solution exists.

### Example
```
Input:  nums = [2, 7, 11, 15], target = 9
Output: [0, 1]
Explanation: nums[0] + nums[1] = 2 + 7 = 9
```

### Key Insight
Brute force checks every pair = O(n²). Instead, for each number ask: "have I seen the complement (target - num) before?" Use a hashmap to answer this in O(1).

### Solution
```python
def twoSum(nums, target):
    seen = {}  # value -> index
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i
    return []
```

### Why It Works
- `seen` stores every number we've visited and its index
- For each new number, we check if its "partner" is already in the map
- One pass through the array = O(n) time, O(n) space

---

## Problem 2 — Best Time to Buy and Sell Stock (Easy)
**LeetCode #121**

### Problem
Given an array `prices` where `prices[i]` is the price on day `i`, find the maximum profit from one buy and one sell. You must buy before you sell.

### Example
```
Input:  prices = [7, 1, 5, 3, 6, 4]
Output: 5
Explanation: Buy on day 2 (price=1), sell on day 5 (price=6), profit=5
```

### Key Insight
Track the minimum price seen so far. At each day, the best profit you could make is `current_price - min_so_far`. Keep updating the max profit.

### Solution
```python
def maxProfit(prices):
    min_price = float('inf')
    max_profit = 0
    for price in prices:
        min_price = min(min_price, price)
        max_profit = max(max_profit, price - min_price)
    return max_profit
```

### Why It Works
- We never need to look back — the running minimum captures the best buy point up to today
- One pass = O(n) time, O(1) space

---

## Problem 3 — Contains Duplicate (Easy)
**LeetCode #217**

### Problem
Given an array `nums`, return `True` if any value appears at least twice.

### Example
```
Input:  nums = [1, 2, 3, 1]
Output: True
```

### Key Insight
A set holds only unique values. If the length of the set is less than the length of the array, a duplicate exists.

### Solution
```python
def containsDuplicate(nums):
    return len(set(nums)) != len(nums)

# Alternative — stops early on first duplicate
def containsDuplicate(nums):
    seen = set()
    for num in nums:
        if num in seen:
            return True
        seen.add(num)
    return False
```

### Why It Works
- Set lookup is O(1) average
- O(n) time, O(n) space

---

## Problem 4 — Maximum Subarray (Easy)
**LeetCode #53**

### Problem
Given an integer array, find the contiguous subarray with the largest sum and return its sum.

### Example
```
Input:  nums = [-2, 1, -3, 4, -1, 2, 1, -5, 4]
Output: 6
Explanation: [4, -1, 2, 1] has the largest sum = 6
```

### Key Insight
Kadane's Algorithm: at each position, decide whether to extend the current subarray or start fresh. If the running sum goes negative, reset it to 0 (starting fresh is better).

### Solution
```python
def maxSubArray(nums):
    max_sum = nums[0]
    current_sum = nums[0]
    for num in nums[1:]:
        current_sum = max(num, current_sum + num)
        max_sum = max(max_sum, current_sum)
    return max_sum
```

### Why It Works
- `max(num, current_sum + num)` means: "Is this number alone better than adding it to what I had?"
- If current_sum was negative, starting fresh from `num` is always better
- O(n) time, O(1) space

---

## Problem 5 — Move Zeroes (Easy)
**LeetCode #283**

### Problem
Given an array `nums`, move all 0s to the end while maintaining the relative order of non-zero elements. Do this **in-place**.

### Example
```
Input:  nums = [0, 1, 0, 3, 12]
Output: [1, 3, 12, 0, 0]
```

### Key Insight
Two-pointer approach. `write` pointer marks where to write the next non-zero. When we find a non-zero, place it at `write` and advance. Then fill the rest with zeros.

### Solution
```python
def moveZeroes(nums):
    write = 0
    for read in range(len(nums)):
        if nums[read] != 0:
            nums[write] = nums[read]
            write += 1
    # Fill remaining with zeros
    while write < len(nums):
        nums[write] = 0
        write += 1
```

### Why It Works
- `write` is always <= `read`, so we never overwrite something we haven't processed yet
- O(n) time, O(1) space in-place

---

## Problem 6 — 3Sum (Medium)
**LeetCode #15**

### Problem
Find all unique triplets in an array that sum to zero.

### Example
```
Input:  nums = [-1, 0, 1, 2, -1, -4]
Output: [[-1, -1, 2], [-1, 0, 1]]
```

### Key Insight
Sort the array first. Fix one number, then use two pointers on the rest to find pairs that sum to the negative of the fixed number. Skip duplicates to avoid repeated triplets.

### Solution
```python
def threeSum(nums):
    nums.sort()
    result = []
    for i in range(len(nums) - 2):
        if i > 0 and nums[i] == nums[i-1]:  # skip duplicate fixed number
            continue
        left, right = i + 1, len(nums) - 1
        while left < right:
            total = nums[i] + nums[left] + nums[right]
            if total == 0:
                result.append([nums[i], nums[left], nums[right]])
                while left < right and nums[left] == nums[left+1]: left += 1
                while left < right and nums[right] == nums[right-1]: right -= 1
                left += 1
                right -= 1
            elif total < 0:
                left += 1
            else:
                right -= 1
    return result
```

### Why It Works
- Sorting allows two-pointer technique
- Two pointers converge from both ends: sum too small → move left pointer right; sum too big → move right pointer left
- O(n²) time after O(n log n) sort

---

## Problem 7 — Product of Array Except Self (Medium)
**LeetCode #238**

### Problem
Return an array where `output[i]` is the product of all elements except `nums[i]`. Do it without division in O(n).

### Example
```
Input:  nums = [1, 2, 3, 4]
Output: [24, 12, 8, 6]
```

### Key Insight
For each index, the answer is (product of everything to the LEFT) × (product of everything to the RIGHT). Do two passes: left-to-right for prefix products, right-to-left for suffix products.

### Solution
```python
def productExceptSelf(nums):
    n = len(nums)
    result = [1] * n
    
    # Left pass: result[i] = product of all nums to the left of i
    prefix = 1
    for i in range(n):
        result[i] = prefix
        prefix *= nums[i]
    
    # Right pass: multiply by product of all nums to the right of i
    suffix = 1
    for i in range(n - 1, -1, -1):
        result[i] *= suffix
        suffix *= nums[i]
    
    return result
```

### Why It Works
- After left pass: result[i] = nums[0] * nums[1] * ... * nums[i-1]
- After right pass: result[i] *= nums[i+1] * ... * nums[n-1]
- Combined = product of everything except nums[i]
- O(n) time, O(1) extra space (output array doesn't count)

---

## Problem 8 — Find Minimum in Rotated Sorted Array (Medium)
**LeetCode #153**

### Problem
A sorted array was rotated at some pivot. Find the minimum element. Must be O(log n).

### Example
```
Input:  nums = [3, 4, 5, 1, 2]
Output: 1
```

### Key Insight
Binary search. If the middle element is greater than the rightmost element, the minimum is in the right half. Otherwise it's in the left half (including mid).

### Solution
```python
def findMin(nums):
    left, right = 0, len(nums) - 1
    while left < right:
        mid = (left + right) // 2
        if nums[mid] > nums[right]:
            left = mid + 1   # min is in right half
        else:
            right = mid      # min is in left half (mid could be the min)
    return nums[left]
```

### Why It Works
- The key observation: in a rotated sorted array, the minimum is at the "break point"
- If nums[mid] > nums[right], the break (minimum) is to the right of mid
- Otherwise the break is at mid or to its left
- O(log n) time

---

## Problem 9 — Container With Most Water (Medium)
**LeetCode #11**

### Problem
Given heights of vertical lines, find two lines that together with the x-axis form a container holding the most water.

### Example
```
Input:  height = [1, 8, 6, 2, 5, 4, 8, 3, 7]
Output: 49
```

### Key Insight
Two pointers at both ends. Water = min(left_height, right_height) × distance. To potentially increase water, move the shorter pointer inward (moving the taller one can only decrease area).

### Solution
```python
def maxArea(height):
    left, right = 0, len(height) - 1
    max_water = 0
    while left < right:
        water = min(height[left], height[right]) * (right - left)
        max_water = max(max_water, water)
        if height[left] < height[right]:
            left += 1
        else:
            right -= 1
    return max_water
```

### Why It Works
- We start with maximum width (both ends)
- If we move the taller pointer, width decreases AND height can only stay or decrease → never better
- Moving the shorter pointer is the only chance to find a taller barrier
- O(n) time, O(1) space

---

## Problem 10 — Search Insert Position (Easy)
**LeetCode #35**

### Problem
Given a sorted array and a target, return the index if found. If not, return the index where it would be inserted in order.

### Example
```
Input:  nums = [1, 3, 5, 6], target = 5  → Output: 2
Input:  nums = [1, 3, 5, 6], target = 2  → Output: 1
Input:  nums = [1, 3, 5, 6], target = 7  → Output: 4
```

### Key Insight
Classic binary search. The answer is where `left` ends up when the loop finishes — this is the correct insert position.

### Solution
```python
def searchInsert(nums, target):
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = (left + right) // 2
        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    return left  # insert position when not found
```

### Why It Works
- When the loop ends, `left > right`
- `left` always points to the first position where `target` could be placed
- O(log n) time, O(1) space

---

# SECTION 2 — STRINGS & HASHING (Problems 11–20)

---

## Problem 11 — Valid Anagram (Easy)
**LeetCode #242**

### Problem
Given two strings `s` and `t`, return `True` if `t` is an anagram of `s` (same characters, same counts).

### Example
```
Input:  s = "anagram", t = "nagaram"
Output: True

Input:  s = "rat", t = "car"
Output: False
```

### Key Insight
Count character frequencies. If both strings have identical frequency maps, they're anagrams.

### Solution
```python
from collections import Counter

def isAnagram(s, t):
    return Counter(s) == Counter(t)

# Manual approach (without Counter)
def isAnagram(s, t):
    if len(s) != len(t):
        return False
    count = {}
    for c in s:
        count[c] = count.get(c, 0) + 1
    for c in t:
        count[c] = count.get(c, 0) - 1
        if count[c] < 0:
            return False
    return True
```

### Why It Works
- Counter creates a frequency dictionary in O(n)
- Two Counter objects are equal only if all keys and values match
- O(n) time, O(1) space (max 26 letters)

---

## Problem 12 — Group Anagrams (Medium)
**LeetCode #49**

### Problem
Given a list of strings, group the anagrams together.

### Example
```
Input:  strs = ["eat","tea","tan","ate","nat","bat"]
Output: [["bat"],["nat","tan"],["ate","eat","tea"]]
```

### Key Insight
Anagrams always produce the same string when sorted. Use the sorted version as a hashmap key to group them.

### Solution
```python
from collections import defaultdict

def groupAnagrams(strs):
    groups = defaultdict(list)
    for s in strs:
        key = tuple(sorted(s))  # "eat" → ('a','e','t')
        groups[key].append(s)
    return list(groups.values())
```

### Why It Works
- Sorting each string gives the canonical form: all anagrams share the same sorted key
- `defaultdict(list)` automatically initializes empty lists
- O(n × k log k) time where k is max string length

---

## Problem 13 — Longest Substring Without Repeating Characters (Medium)
**LeetCode #3**

### Problem
Find the length of the longest substring without repeating characters.

### Example
```
Input:  s = "abcabcbb"
Output: 3  (substring "abc")

Input:  s = "pwwkew"
Output: 3  (substring "wke")
```

### Key Insight
Sliding window. Expand the right pointer. When a duplicate is found, shrink the window from the left until the duplicate is removed.

### Solution
```python
def lengthOfLongestSubstring(s):
    char_index = {}  # char -> its most recent index
    max_len = 0
    left = 0
    for right, char in enumerate(s):
        if char in char_index and char_index[char] >= left:
            left = char_index[char] + 1  # jump left past the duplicate
        char_index[char] = right
        max_len = max(max_len, right - left + 1)
    return max_len
```

### Why It Works
- We never need to re-examine characters — when duplicate found, jump `left` directly past it
- The condition `char_index[char] >= left` ensures the duplicate is inside our current window
- O(n) time, O(min(n, alphabet)) space

---

## Problem 14 — Valid Parentheses (Easy)
**LeetCode #20**

### Problem
Given a string of brackets `()[]{}`, determine if it is valid (each open bracket is closed in the correct order).

### Example
```
Input:  s = "()[]{}"   → Output: True
Input:  s = "([)]"     → Output: False
Input:  s = "{[]}"     → Output: True
```

### Key Insight
Stack. Push opening brackets. On closing bracket, check if it matches the top of the stack.

### Solution
```python
def isValid(s):
    stack = []
    matching = {')': '(', ']': '[', '}': '{'}
    for char in s:
        if char in '([{':
            stack.append(char)
        elif char in ')]}':
            if not stack or stack[-1] != matching[char]:
                return False
            stack.pop()
    return len(stack) == 0
```

### Why It Works
- A stack naturally tracks the "most recently opened" bracket
- `stack[-1]` is the most recent open bracket — it must be closed first (LIFO)
- Empty stack at the end means everything was matched
- O(n) time, O(n) space

---

## Problem 15 — Longest Common Prefix (Easy)
**LeetCode #14**

### Problem
Write a function to find the longest common prefix string among an array of strings.

### Example
```
Input:  strs = ["flower","flow","flight"]
Output: "fl"

Input:  strs = ["dog","racecar","car"]
Output: ""
```

### Key Insight
Use the first string as the reference. For each character in the first string, check if all other strings have the same character at that position.

### Solution
```python
def longestCommonPrefix(strs):
    if not strs:
        return ""
    prefix = strs[0]
    for s in strs[1:]:
        while not s.startswith(prefix):
            prefix = prefix[:-1]  # trim one char from end
            if not prefix:
                return ""
    return prefix
```

### Why It Works
- We shrink the candidate prefix until all strings start with it
- `startswith` is O(k) where k = prefix length
- O(n × m) time where m is average string length

---

## Problem 16 — Reverse Words in a String (Medium)
**LeetCode #151**

### Problem
Given a string `s`, reverse the order of the words. A word is a sequence of non-space characters. Remove extra spaces.

### Example
```
Input:  s = "  the sky  is blue  "
Output: "blue is sky the"
```

### Key Insight
Split on whitespace (handles multiple spaces), reverse the list, rejoin.

### Solution
```python
def reverseWords(s):
    words = s.split()      # split() handles multiple spaces automatically
    words.reverse()
    return ' '.join(words)

# One-liner
def reverseWords(s):
    return ' '.join(s.split()[::-1])
```

### Why It Works
- `split()` with no argument splits on any whitespace AND removes empty strings from multiple spaces
- Reversing the list and joining gives the result
- O(n) time

---

## Problem 17 — Find All Anagrams in a String (Medium)
**LeetCode #438**

### Problem
Given strings `s` and `p`, find all start indices of `p`'s anagrams in `s`.

### Example
```
Input:  s = "cbaebabacd", p = "abc"
Output: [0, 6]
Explanation: s[0:3]="cba" and s[6:9]="bac" are both anagrams of "abc"
```

### Key Insight
Sliding window of fixed size (len(p)). Compare character counts of the window vs `p`. Slide the window one step at a time, updating counts in O(1).

### Solution
```python
from collections import Counter

def findAnagrams(s, p):
    if len(p) > len(s):
        return []
    
    p_count = Counter(p)
    window = Counter(s[:len(p)])
    result = []
    
    if window == p_count:
        result.append(0)
    
    for i in range(len(p), len(s)):
        # Add new character to window
        window[s[i]] += 1
        # Remove oldest character from window
        old = s[i - len(p)]
        window[old] -= 1
        if window[old] == 0:
            del window[old]
        if window == p_count:
            result.append(i - len(p) + 1)
    
    return result
```

### Why It Works
- Fixed-size window slides across `s`
- Each step is O(1) — add one char, remove one char
- O(n) total time instead of O(n×k) for brute force

---

## Problem 18 — Roman to Integer (Easy)
**LeetCode #13**

### Problem
Convert a Roman numeral string to an integer.

### Example
```
Input:  s = "MCMXCIV"
Output: 1994
```

### Key Insight
If a smaller value appears before a larger one, subtract it. Otherwise add it. Walk right to left and check if current < next.

### Solution
```python
def romanToInt(s):
    values = {'I': 1, 'V': 5, 'X': 10, 'L': 50,
              'C': 100, 'D': 500, 'M': 1000}
    result = 0
    for i in range(len(s)):
        if i + 1 < len(s) and values[s[i]] < values[s[i+1]]:
            result -= values[s[i]]  # subtract (e.g. IV = -1 + 5 = 4)
        else:
            result += values[s[i]]
    return result
```

### Why It Works
- IV = I before V → subtract I (1), add V (5) = 4 ✓
- VI = V before I → add both = 6 ✓
- O(n) time, O(1) space

---

## Problem 19 — Count Vowel Substrings of a String (Medium)
**LeetCode #2062**

### Problem
Count substrings that contain only vowels (a,e,i,o,u) and contain all 5 vowels at least once.

### Example
```
Input:  word = "aeiouu"
Output: 2  ("aeiou" and "aeiouu")
```

### Key Insight
Sliding window with a frequency map. Expand right. When window has all 5 vowels, every extension is also valid. Shrink left when a consonant is found.

### Solution
```python
def countVowelSubstrings(word):
    vowels = set('aeiou')
    count = 0
    n = len(word)
    for i in range(n):
        if word[i] not in vowels:
            continue
        seen = set()
        for j in range(i, n):
            if word[j] not in vowels:
                break
            seen.add(word[j])
            if len(seen) == 5:
                count += 1
    return count
```

### Why It Works
- Outer loop sets the start of a valid vowel-only substring
- Inner loop extends until a consonant or end of string
- Only counts when all 5 vowels are present
- O(n²) — acceptable for this problem

---

## Problem 20 — Encode and Decode Strings (Medium)
**LeetCode #271**

### Problem
Design an algorithm to encode a list of strings into a single string, and decode it back.

### Example
```
Input:  ["lint","code","love","you"]
Encode: "4#lint4#code4#love3#you"
Decode: ["lint","code","love","you"]
```

### Key Insight
Store the length of each word followed by a delimiter before the word. Use the length to know exactly where each word ends — no ambiguity even if words contain the delimiter.

### Solution
```python
def encode(strs):
    return ''.join(f"{len(s)}#{s}" for s in strs)

def decode(s):
    result = []
    i = 0
    while i < len(s):
        j = s.index('#', i)        # find the # separator
        length = int(s[i:j])       # read the length
        result.append(s[j+1:j+1+length])  # extract the word
        i = j + 1 + length         # jump to next encoded word
    return result
```

### Why It Works
- `length#word` format is unambiguous — we always know where each word ends
- The `#` in a word content doesn't matter because we use the length to skip ahead
- O(n) encode and decode

---

# SECTION 3 — BINARY SEARCH (Problems 21–27)

---

## Problem 21 — Binary Search (Easy)
**LeetCode #704**

### Problem
Given a sorted array and target, return the index of target or -1 if not found.

### Example
```
Input:  nums = [-1,0,3,5,9,12], target = 9
Output: 4
```

### Key Insight
Classic binary search template. Three cases: found, go left, go right.

### Solution
```python
def search(nums, target):
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = (left + right) // 2
        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    return -1
```

### Why It Works
- Each iteration halves the search space
- `left <= right` (not `<`) ensures we check a single-element array
- O(log n) time, O(1) space

---

## Problem 22 — First Bad Version (Easy)
**LeetCode #278**

### Problem
`isBadVersion(n)` is given. Find the first bad version (all versions after first bad are also bad).

### Example
```
Input:  n = 5, bad = 4
Calls:  isBadVersion(3)→False, isBadVersion(5)→True, isBadVersion(4)→True
Output: 4
```

### Key Insight
Binary search. If mid is bad, the first bad might be mid or earlier. If mid is good, first bad is after mid.

### Solution
```python
def firstBadVersion(n):
    left, right = 1, n
    while left < right:          # Note: left < right (not <=)
        mid = (left + right) // 2
        if isBadVersion(mid):
            right = mid          # could be the answer, don't exclude mid
        else:
            left = mid + 1       # mid is good, answer is strictly after
    return left
```

### Why It Works
- `right = mid` (not `mid - 1`) because mid itself might be the first bad
- Loop ends when `left == right` — both point to the first bad version
- Minimizes calls to `isBadVersion`

---

## Problem 23 — Koko Eating Bananas (Medium)
**LeetCode #875**

### Problem
Koko eats `k` bananas/hour. Find the minimum `k` such that she finishes all piles in `h` hours.

### Example
```
Input:  piles = [3,6,7,11], h = 8
Output: 4
```

### Key Insight
Binary search on the answer (k). For a given k, compute hours needed = sum of ceil(pile/k). Search between 1 and max(piles).

### Solution
```python
import math

def minEatingSpeed(piles, h):
    left, right = 1, max(piles)
    result = right
    while left <= right:
        mid = (left + right) // 2
        hours = sum(math.ceil(pile / mid) for pile in piles)
        if hours <= h:
            result = mid       # valid speed, try slower
            right = mid - 1
        else:
            left = mid + 1     # too slow, need faster
    return result
```

### Why It Works
- The answer space is [1, max(piles)] — a sorted range we can binary search
- For any k: more bananas/hour → fewer hours needed (monotonic relationship)
- This allows binary search: if k works, try smaller; if not, try larger
- O(n log m) where m = max pile

---

## Problem 24 — Search a 2D Matrix (Medium)
**LeetCode #74**

### Problem
An m×n matrix where each row is sorted and the first element of each row is greater than the last of the previous row. Search for a target.

### Example
```
Input:  matrix = [[1,3,5,7],[10,11,16,20],[23,30,34,60]], target = 3
Output: True
```

### Key Insight
Treat the 2D matrix as a 1D sorted array of length m×n. Binary search using index math to convert position to row/col.

### Solution
```python
def searchMatrix(matrix, target):
    m, n = len(matrix), len(matrix[0])
    left, right = 0, m * n - 1
    while left <= right:
        mid = (left + right) // 2
        row, col = mid // n, mid % n   # convert flat index to 2D
        val = matrix[row][col]
        if val == target:
            return True
        elif val < target:
            left = mid + 1
        else:
            right = mid - 1
    return False
```

### Why It Works
- The matrix is essentially a sorted array "laid out" row by row
- `mid // n` gives the row, `mid % n` gives the column
- O(log(m×n)) time

---

## Problem 25 — Find Peak Element (Medium)
**LeetCode #162**

### Problem
Find any peak element (strictly greater than its neighbors). Array has no duplicates. O(log n) required.

### Example
```
Input:  nums = [1,2,3,1]
Output: 2  (index of value 3)
```

### Key Insight
If mid < mid+1, there must be a peak to the right (values are rising). Otherwise there's a peak at mid or to its left.

### Solution
```python
def findPeakElement(nums):
    left, right = 0, len(nums) - 1
    while left < right:
        mid = (left + right) // 2
        if nums[mid] < nums[mid + 1]:
            left = mid + 1    # rising slope, peak is to the right
        else:
            right = mid       # falling slope, peak is here or to the left
    return left
```

### Why It Works
- We follow the "uphill" direction — guaranteed to find a peak
- Works because boundaries are considered -infinity
- O(log n) time

---

## Problem 26 — Time Based Key-Value Store (Medium)
**LeetCode #981**

### Problem
Design a structure that stores (key, value, timestamp) and retrieves the value for a key at the largest timestamp <= given timestamp.

### Example
```
store.set("foo", "bar", 1)
store.set("foo", "bar2", 4)
store.get("foo", 4) → "bar2"
store.get("foo", 3) → "bar"
```

### Key Insight
Store a list of (timestamp, value) per key (sorted by time since set is called in increasing order). Binary search for the largest timestamp <= given.

### Solution
```python
from collections import defaultdict
import bisect

class TimeMap:
    def __init__(self):
        self.store = defaultdict(list)  # key -> [(timestamp, value)]

    def set(self, key, value, timestamp):
        self.store[key].append((timestamp, value))

    def get(self, key, timestamp):
        if key not in self.store:
            return ""
        entries = self.store[key]
        # Binary search for largest timestamp <= given
        left, right = 0, len(entries) - 1
        result = ""
        while left <= right:
            mid = (left + right) // 2
            if entries[mid][0] <= timestamp:
                result = entries[mid][1]
                left = mid + 1
            else:
                right = mid - 1
        return result
```

### Why It Works
- Timestamps are inserted in order → the list is always sorted
- Binary search finds the rightmost valid timestamp in O(log n)

---

## Problem 27 — Median of Two Sorted Arrays (Hard — for practice only)
**LeetCode #4**

### Problem
Find the median of two sorted arrays in O(log(m+n)).

*(This one is hard — include it for stretch practice. Don't panic if it takes time.)*

### Example
```
Input:  nums1 = [1,3], nums2 = [2]
Output: 2.0
```

### Key Insight
Binary search on the smaller array to find the correct partition point where left half of both arrays combined = right half.

### Solution
```python
def findMedianSortedArrays(nums1, nums2):
    if len(nums1) > len(nums2):
        nums1, nums2 = nums2, nums1  # ensure nums1 is shorter
    m, n = len(nums1), len(nums2)
    half = (m + n) // 2
    left, right = 0, m
    while True:
        i = (left + right) // 2   # partition in nums1
        j = half - i              # partition in nums2
        l1 = nums1[i-1] if i > 0 else float('-inf')
        r1 = nums1[i]   if i < m else float('inf')
        l2 = nums2[j-1] if j > 0 else float('-inf')
        r2 = nums2[j]   if j < n else float('inf')
        if l1 <= r2 and l2 <= r1:
            if (m + n) % 2:
                return min(r1, r2)
            return (max(l1, l2) + min(r1, r2)) / 2
        elif l1 > r2:
            right = i - 1
        else:
            left = i + 1
```

---

# SECTION 4 — TREES (Problems 28–40)

---

## Problem 28 — Invert Binary Tree (Easy)
**LeetCode #226** ← YOU GOT THIS IN YOUR INTERVIEW

### Problem
Invert a binary tree (mirror it left-to-right).

### Example
```
Input:       Output:
    4            4
   / \          / \
  2   7        7   2
 / \ / \      / \ / \
1  3 6  9    9  6 3  1
```

### Key Insight
Recursion. At each node, swap its left and right children, then recursively invert each subtree.

### Solution
```python
def invertTree(root):
    if not root:
        return None
    # Swap left and right
    root.left, root.right = root.right, root.left
    # Recurse on both sides
    invertTree(root.left)
    invertTree(root.right)
    return root
```

### Why It Works
- Base case: `None` node → return None
- At every node: swap children first, then recurse
- Every node in the tree gets its children swapped exactly once
- O(n) time (visit every node), O(h) space for recursion stack where h = height

---

## Problem 29 — Maximum Depth of Binary Tree (Easy)
**LeetCode #104**

### Problem
Find the maximum depth (number of nodes along the longest root-to-leaf path).

### Example
```
Input:   [3,9,20,null,null,15,7]
Output:  3
```

### Key Insight
Recursion. Depth at a node = 1 + max(depth of left subtree, depth of right subtree).

### Solution
```python
def maxDepth(root):
    if not root:
        return 0
    return 1 + max(maxDepth(root.left), maxDepth(root.right))
```

### Why It Works
- Empty tree has depth 0
- Each call asks: "what's the deeper side?" and adds 1 for the current level
- Elegant 1-liner that captures the recursive definition perfectly
- O(n) time

---

## Problem 30 — Same Tree (Easy)
**LeetCode #100**

### Problem
Check if two binary trees are structurally identical with the same node values.

### Example
```
Input:  p = [1,2,3], q = [1,2,3]
Output: True

Input:  p = [1,2], q = [1,null,2]
Output: False
```

### Key Insight
Recursion. Two trees are the same if: both are None, OR both have the same root value AND their left subtrees are the same AND their right subtrees are the same.

### Solution
```python
def isSameTree(p, q):
    if not p and not q:    # both None → same
        return True
    if not p or not q:     # one is None, other isn't → different
        return False
    if p.val != q.val:     # different values → different
        return False
    return isSameTree(p.left, q.left) and isSameTree(p.right, q.right)
```

### Why It Works
- All 4 cases handled clearly: both None, one None, different values, same values (recurse)
- O(n) time where n = smaller tree size

---

## Problem 31 — Binary Tree Level Order Traversal (Medium)
**LeetCode #102** ← YOU GOT THIS IN YOUR INTERVIEW (BFS)

### Problem
Return the level-order traversal of a binary tree as a list of lists (each list = one level).

### Example
```
Input:   [3,9,20,null,null,15,7]
Output:  [[3],[9,20],[15,7]]
```

### Key Insight
BFS using a queue. Process all nodes at the current level before moving to the next. Use the queue's current size to know how many nodes are on this level.

### Solution
```python
from collections import deque

def levelOrder(root):
    if not root:
        return []
    result = []
    queue = deque([root])
    while queue:
        level_size = len(queue)   # how many nodes are on this level
        level = []
        for _ in range(level_size):
            node = queue.popleft()
            level.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        result.append(level)
    return result
```

### Why It Works
- `deque` gives O(1) popleft (list.pop(0) is O(n))
- `level_size` snapshot at the start of each iteration = number of nodes at current level
- After processing all nodes at current level, queue contains exactly next level's nodes
- O(n) time, O(n) space (queue can hold entire last level)

---

## Problem 32 — Validate Binary Search Tree (Medium)
**LeetCode #98**

### Problem
Determine if a binary tree is a valid BST (left < root < right, recursively).

### Example
```
Input:   [2,1,3]   → True
Input:   [5,1,4,null,null,3,6]  → False (4 is right child of 5 but 4 < 5)
```

### Key Insight
Pass down valid range [min_val, max_val]. Every node must be strictly within its valid range. Left subtree's max = current node value. Right subtree's min = current node value.

### Solution
```python
def isValidBST(root, min_val=float('-inf'), max_val=float('inf')):
    if not root:
        return True
    if root.val <= min_val or root.val >= max_val:
        return False
    return (isValidBST(root.left, min_val, root.val) and
            isValidBST(root.right, root.val, max_val))
```

### Why It Works
- Root must be in (-inf, +inf)
- Left child must be in (min_val, root.val) — bounded above by parent
- Right child must be in (root.val, max_val) — bounded below by parent
- O(n) time

---

## Problem 33 — Lowest Common Ancestor of BST (Easy)
**LeetCode #235**

### Problem
Find the lowest common ancestor (LCA) of two nodes in a BST.

### Example
```
Input:  root=[6,2,8,0,4,7,9], p=2, q=8
Output: 6

Input:  root=[6,2,8,0,4,7,9], p=2, q=4
Output: 2
```

### Key Insight
In a BST, if both p and q are less than root → LCA is in left subtree. If both greater → right subtree. Otherwise root is the LCA (they're on different sides).

### Solution
```python
def lowestCommonAncestor(root, p, q):
    while root:
        if p.val < root.val and q.val < root.val:
            root = root.left    # both in left subtree
        elif p.val > root.val and q.val > root.val:
            root = root.right   # both in right subtree
        else:
            return root         # split point = LCA
    return None
```

### Why It Works
- BST property tells us exactly which direction to go
- When values are on different sides of root, root must be the LCA
- O(h) time where h = tree height, O(1) space (iterative)

---

## Problem 34 — Balanced Binary Tree (Easy)
**LeetCode #110**

### Problem
Determine if a binary tree is height-balanced (for every node, left and right subtrees differ in height by at most 1).

### Example
```
Input:  [3,9,20,null,null,15,7]  → True
Input:  [1,2,2,3,3,null,null,4,4]  → False
```

### Key Insight
Post-order DFS. Return the height from each subtree, but return -1 if unbalanced. Propagate -1 up the tree.

### Solution
```python
def isBalanced(root):
    def height(node):
        if not node:
            return 0
        left_h = height(node.left)
        right_h = height(node.right)
        if left_h == -1 or right_h == -1:       # subtree already unbalanced
            return -1
        if abs(left_h - right_h) > 1:           # this node is unbalanced
            return -1
        return 1 + max(left_h, right_h)
    
    return height(root) != -1
```

### Why It Works
- -1 is used as a sentinel for "unbalanced" — propagates all the way up
- One DFS pass computes heights and checks balance simultaneously
- O(n) time

---

## Problem 35 — Diameter of Binary Tree (Easy)
**LeetCode #543**

### Problem
Find the length of the longest path between any two nodes (the diameter). The path may not pass through the root.

### Example
```
Input:  [1,2,3,4,5]
Output: 3  (path: 4→2→1→3 or 5→2→1→3)
```

### Key Insight
At each node, the diameter through that node = height(left) + height(right). Track the global maximum while computing heights.

### Solution
```python
def diameterOfBinaryTree(root):
    max_diameter = [0]  # use list to allow mutation inside nested function
    
    def height(node):
        if not node:
            return 0
        left_h = height(node.left)
        right_h = height(node.right)
        max_diameter[0] = max(max_diameter[0], left_h + right_h)
        return 1 + max(left_h, right_h)
    
    height(root)
    return max_diameter[0]
```

### Why It Works
- `left_h + right_h` = path length through current node
- We update global max at every node, not just root
- O(n) time

---

## Problem 36 — Path Sum (Easy)
**LeetCode #112**

### Problem
Given a binary tree and a `targetSum`, return True if there exists a root-to-leaf path that sums to `targetSum`.

### Example
```
Input:  root=[5,4,8,11,null,13,4,7,2,null,null,null,1], targetSum=22
Output: True  (path: 5→4→11→2)
```

### Key Insight
DFS. Subtract node's value from target at each step. At a leaf, check if target reaches 0.

### Solution
```python
def hasPathSum(root, targetSum):
    if not root:
        return False
    targetSum -= root.val
    if not root.left and not root.right:  # leaf node
        return targetSum == 0
    return hasPathSum(root.left, targetSum) or hasPathSum(root.right, targetSum)
```

### Why It Works
- Subtract value at each node — if we reach a leaf with exactly 0 remaining, we found the path
- `or` short-circuits: if left subtree finds a path, don't even check right
- O(n) time

---

## Problem 37 — Symmetric Tree (Easy)
**LeetCode #101**

### Problem
Check if a binary tree is a mirror of itself (symmetric around its center).

### Example
```
Input:  [1,2,2,3,4,4,3]  → True
Input:  [1,2,2,null,3,null,3]  → False
```

### Key Insight
Two subtrees are mirrors if: both are None, OR both have the same value AND the outer children are mirrors AND the inner children are mirrors.

### Solution
```python
def isSymmetric(root):
    def isMirror(left, right):
        if not left and not right: return True
        if not left or not right: return False
        return (left.val == right.val and
                isMirror(left.left, right.right) and   # outer
                isMirror(left.right, right.left))       # inner
    
    return isMirror(root.left, root.right)
```

### Why It Works
- Compare the tree with its own mirror: left.left vs right.right (outer) and left.right vs right.left (inner)
- O(n) time

---

## Problem 38 — Binary Tree Right Side View (Medium)
**LeetCode #199**

### Problem
Imagine looking at the tree from the right. Return the values of nodes visible from the right side.

### Example
```
Input:  [1,2,3,null,5,null,4]
Output: [1,3,4]
```

### Key Insight
BFS level-order traversal. The rightmost node at each level is the one visible from the right.

### Solution
```python
from collections import deque

def rightSideView(root):
    if not root:
        return []
    result = []
    queue = deque([root])
    while queue:
        level_size = len(queue)
        for i in range(level_size):
            node = queue.popleft()
            if i == level_size - 1:     # last node of this level
                result.append(node.val)
            if node.left:  queue.append(node.left)
            if node.right: queue.append(node.right)
    return result
```

### Why It Works
- BFS guarantees left-to-right processing at each level
- The last node processed at each level (i == level_size - 1) is the rightmost
- O(n) time

---

## Problem 39 — Count Good Nodes in Binary Tree (Medium)
**LeetCode #1448**

### Problem
A node is "good" if the path from root to it has no node with a greater value. Count good nodes.

### Example
```
Input:  root=[3,1,4,3,null,1,5]
Output: 4
```

### Key Insight
DFS, tracking the maximum value seen so far on the path. A node is good if its value >= max on path to it.

### Solution
```python
def goodNodes(root):
    def dfs(node, max_so_far):
        if not node:
            return 0
        is_good = 1 if node.val >= max_so_far else 0
        new_max = max(max_so_far, node.val)
        return is_good + dfs(node.left, new_max) + dfs(node.right, new_max)
    
    return dfs(root, float('-inf'))
```

### Why It Works
- Carry max value seen along current path
- Root is always good (nothing above it is greater)
- O(n) time

---

## Problem 40 — Construct Binary Tree from Preorder and Inorder (Medium)
**LeetCode #105**

### Problem
Build a binary tree from preorder and inorder traversal arrays.

### Example
```
Input:  preorder=[3,9,20,15,7], inorder=[9,3,15,20,7]
Output: Tree rooted at 3
```

### Key Insight
In preorder, the first element is always the root. In inorder, everything left of the root value is the left subtree, everything right is the right subtree. Recurse.

### Solution
```python
def buildTree(preorder, inorder):
    if not preorder or not inorder:
        return None
    root_val = preorder[0]
    root = TreeNode(root_val)
    mid = inorder.index(root_val)   # split point in inorder
    root.left  = buildTree(preorder[1:mid+1], inorder[:mid])
    root.right = buildTree(preorder[mid+1:],  inorder[mid+1:])
    return root
```

### Why It Works
- preorder[0] = root
- `mid` elements before root in inorder = left subtree size
- Use `mid` to slice both arrays into left/right halves
- O(n²) simple version; O(n) with index map

---

# SECTION 5 — GRAPHS (Problems 41–50)

---

## Problem 41 — Number of Islands (Medium)
**LeetCode #200**

### Problem
Given a grid of '1's (land) and '0's (water), count the number of islands.

### Example
```
Input:
11110
11010
11000
00000
Output: 1
```

### Key Insight
DFS from every unvisited '1'. When you find land, flood-fill (mark all connected land as visited) — that's one island. Count how many flood-fills you start.

### Solution
```python
def numIslands(grid):
    if not grid:
        return 0
    rows, cols = len(grid), len(grid[0])
    count = 0
    
    def dfs(r, c):
        if r < 0 or r >= rows or c < 0 or c >= cols or grid[r][c] != '1':
            return
        grid[r][c] = '#'   # mark visited (in-place)
        dfs(r+1, c); dfs(r-1, c)
        dfs(r, c+1); dfs(r, c-1)
    
    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == '1':
                dfs(r, c)
                count += 1
    return count
```

### Why It Works
- Each cell is visited at most once (marked '#')
- One DFS call sinks an entire island
- Count = number of DFS calls started from unvisited '1' cells
- O(m×n) time and space

---

## Problem 42 — Clone Graph (Medium)
**LeetCode #133**

### Problem
Clone (deep copy) a connected undirected graph. Each node has `val` and `neighbors`.

### Example
```
Input:  adjList = [[2,4],[1,3],[2,4],[1,3]]
Output: deep copy of the same graph
```

### Key Insight
DFS with a hashmap (old_node → new_node). Before recursing on neighbors, create the clone and add it to the map to handle cycles.

### Solution
```python
def cloneGraph(node):
    if not node:
        return None
    cloned = {}  # old node -> new node
    
    def dfs(n):
        if n in cloned:
            return cloned[n]
        clone = Node(n.val)
        cloned[n] = clone          # store BEFORE recursing (handles cycles)
        for neighbor in n.neighbors:
            clone.neighbors.append(dfs(neighbor))
        return clone
    
    return dfs(node)
```

### Why It Works
- `cloned[n] = clone` before recursing prevents infinite loops in cyclic graphs
- When we revisit a node, we return the already-created clone
- O(V + E) time

---

## Problem 43 — Course Schedule (Medium)
**LeetCode #207**

### Problem
There are `numCourses` courses. `prerequisites[i] = [a, b]` means you must take b before a. Return True if you can finish all courses (no cycle in the dependency graph).

### Example
```
Input:  numCourses=2, prerequisites=[[1,0]]
Output: True

Input:  numCourses=2, prerequisites=[[1,0],[0,1]]
Output: False (cycle)
```

### Key Insight
DFS cycle detection. A cycle exists if we visit a node that is currently in the recursion stack. Use states: 0=unvisited, 1=in-progress, 2=done.

### Solution
```python
def canFinish(numCourses, prerequisites):
    adj = [[] for _ in range(numCourses)]
    for a, b in prerequisites:
        adj[b].append(a)
    
    state = [0] * numCourses   # 0=unvisited, 1=in-progress, 2=done
    
    def dfs(node):
        if state[node] == 1: return False   # cycle!
        if state[node] == 2: return True    # already processed
        state[node] = 1                     # mark in-progress
        for neighbor in adj[node]:
            if not dfs(neighbor):
                return False
        state[node] = 2                     # mark done
        return True
    
    for i in range(numCourses):
        if not dfs(i):
            return False
    return True
```

### Why It Works
- State 1 = "currently being explored" — if we hit it again, we found a cycle
- State 2 = "fully explored" — safe to skip
- O(V + E) time

---

## Problem 44 — Pacific Atlantic Water Flow (Medium)
**LeetCode #417**

### Problem
Water can flow from a cell to adjacent cells with equal or lower height. Find all cells from which water can reach BOTH the Pacific (top/left) and Atlantic (bottom/right) oceans.

### Example
```
Input:  heights = [[1,2,2,3,5],[3,2,3,4,4],[2,4,5,3,1],[6,7,1,4,5],[5,1,1,2,4]]
Output: [[0,4],[1,3],[1,4],[2,2],[3,0],[3,1],[4,0]]
```

### Key Insight
Reverse the problem: do BFS/DFS from the ocean borders inward. Find cells reachable from Pacific borders and cells reachable from Atlantic borders. Intersection = answer.

### Solution
```python
from collections import deque

def pacificAtlantic(heights):
    rows, cols = len(heights), len(heights[0])
    
    def bfs(starts):
        queue = deque(starts)
        visited = set(starts)
        while queue:
            r, c = queue.popleft()
            for dr, dc in [(1,0),(-1,0),(0,1),(0,-1)]:
                nr, nc = r+dr, c+dc
                if (0 <= nr < rows and 0 <= nc < cols and
                    (nr, nc) not in visited and
                    heights[nr][nc] >= heights[r][c]):  # can flow FROM (nr,nc) TO (r,c)
                    visited.add((nr, nc))
                    queue.append((nr, nc))
        return visited
    
    pacific_starts  = [(r, 0) for r in range(rows)] + [(0, c) for c in range(cols)]
    atlantic_starts = [(r, cols-1) for r in range(rows)] + [(rows-1, c) for c in range(cols)]
    
    pac = bfs(pacific_starts)
    atl = bfs(atlantic_starts)
    
    return [[r, c] for r, c in pac & atl]
```

### Why It Works
- Going "uphill inward" from ocean = all cells that could drain down to that ocean
- Intersection gives cells that can reach both
- O(m×n) time

---

## Problem 45 — Word Search (Medium)
**LeetCode #79**

### Problem
Given a 2D grid of characters and a word, return True if the word can be constructed from sequentially adjacent cells (no reuse).

### Example
```
Input:  board = [["A","B","C","E"],["S","F","C","S"],["A","D","E","E"]], word = "ABCCED"
Output: True
```

### Key Insight
DFS + backtracking from every cell. Mark cells visited during the search, unmark when backtracking.

### Solution
```python
def exist(board, word):
    rows, cols = len(board), len(board[0])
    
    def dfs(r, c, idx):
        if idx == len(word):
            return True
        if r < 0 or r >= rows or c < 0 or c >= cols or board[r][c] != word[idx]:
            return False
        temp = board[r][c]
        board[r][c] = '#'    # mark visited
        found = (dfs(r+1,c,idx+1) or dfs(r-1,c,idx+1) or
                 dfs(r,c+1,idx+1) or dfs(r,c-1,idx+1))
        board[r][c] = temp   # unmark (backtrack)
        return found
    
    for r in range(rows):
        for c in range(cols):
            if dfs(r, c, 0):
                return True
    return False
```

### Why It Works
- Marking '#' prevents reusing the same cell in the same path
- Restoring the cell on backtrack allows other paths to use it
- O(m × n × 4^L) worst case where L = word length

---

## Problem 46 — Surrounded Regions (Medium)
**LeetCode #130**

### Problem
Given a board of 'X' and 'O', capture all 'O' regions surrounded by 'X' (flip them to 'X'). 'O' on borders and connected to borders are NOT captured.

### Example
```
Input:
XXXX       XXXX
XOOX  →    XXXX
XXOX       XXOX
XOXX       XOXX
```

### Key Insight
Reverse thinking: mark all 'O's connected to borders as safe ('T'). Then flip all remaining 'O' to 'X', and 'T' back to 'O'.

### Solution
```python
def solve(board):
    rows, cols = len(board), len(board[0])
    
    def dfs(r, c):
        if r < 0 or r >= rows or c < 0 or c >= cols or board[r][c] != 'O':
            return
        board[r][c] = 'T'   # mark as safe
        dfs(r+1,c); dfs(r-1,c); dfs(r,c+1); dfs(r,c-1)
    
    # Mark border-connected 'O's as safe
    for r in range(rows):
        dfs(r, 0); dfs(r, cols-1)
    for c in range(cols):
        dfs(0, c); dfs(rows-1, c)
    
    # Flip: 'O'→'X' (captured), 'T'→'O' (safe)
    for r in range(rows):
        for c in range(cols):
            if board[r][c] == 'O': board[r][c] = 'X'
            elif board[r][c] == 'T': board[r][c] = 'O'
```

### Why It Works
- Any 'O' NOT connected to a border must be fully surrounded
- Marking border-connected ones first, then sweeping, handles this cleanly
- O(m×n) time

---

## Problem 47 — Rotting Oranges (Medium)
**LeetCode #994**

### Problem
In a grid, 0=empty, 1=fresh, 2=rotten. Each minute, rotten oranges rot adjacent fresh ones. Return minutes until no fresh remain, or -1 if impossible.

### Example
```
Input:  [[2,1,1],[1,1,0],[0,1,1]]
Output: 4
```

### Key Insight
Multi-source BFS. Start BFS from ALL rotten oranges simultaneously. Each BFS level = 1 minute. Count remaining fresh oranges after BFS.

### Solution
```python
from collections import deque

def orangesRotting(grid):
    rows, cols = len(grid), len(grid[0])
    queue = deque()
    fresh = 0
    
    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == 2:
                queue.append((r, c, 0))   # (row, col, time)
            elif grid[r][c] == 1:
                fresh += 1
    
    max_time = 0
    while queue:
        r, c, t = queue.popleft()
        for dr, dc in [(1,0),(-1,0),(0,1),(0,-1)]:
            nr, nc = r+dr, c+dc
            if 0 <= nr < rows and 0 <= nc < cols and grid[nr][nc] == 1:
                grid[nr][nc] = 2
                fresh -= 1
                max_time = max(max_time, t+1)
                queue.append((nr, nc, t+1))
    
    return max_time if fresh == 0 else -1
```

### Why It Works
- Multi-source BFS: spreading from multiple sources simultaneously models the problem perfectly
- `fresh` counter tracks if all oranges were reached
- O(m×n) time

---

## Problem 48 — Graph Valid Tree (Medium)
**LeetCode #261**

### Problem
Given `n` nodes and a list of edges, determine if they form a valid tree (connected + no cycles).

### Example
```
Input:  n=5, edges=[[0,1],[0,2],[0,3],[1,4]]
Output: True
```

### Key Insight
A valid tree has exactly n-1 edges AND is fully connected. Check both conditions.

### Solution
```python
def validTree(n, edges):
    if len(edges) != n - 1:     # quick check: tree must have n-1 edges
        return False
    
    adj = [[] for _ in range(n)]
    for a, b in edges:
        adj[a].append(b)
        adj[b].append(a)
    
    visited = set()
    
    def dfs(node):
        if node in visited:
            return
        visited.add(node)
        for neighbor in adj[node]:
            dfs(neighbor)
    
    dfs(0)
    return len(visited) == n    # all nodes reachable → connected
```

### Why It Works
- n-1 edges + fully connected → no cycles (mathematical property of trees)
- If connected and n-1 edges, it's a tree by definition
- O(V + E) time

---

## Problem 49 — Number of Connected Components in Undirected Graph (Medium)
**LeetCode #323**

### Problem
Given `n` nodes and a list of edges, return the number of connected components.

### Example
```
Input:  n=5, edges=[[0,1],[1,2],[3,4]]
Output: 2
```

### Key Insight
DFS/BFS from each unvisited node — each DFS call explores one complete component. Count the calls.

### Solution
```python
def countComponents(n, edges):
    adj = [[] for _ in range(n)]
    for a, b in edges:
        adj[a].append(b)
        adj[b].append(a)
    
    visited = set()
    count = 0
    
    def dfs(node):
        if node in visited:
            return
        visited.add(node)
        for neighbor in adj[node]:
            dfs(neighbor)
    
    for i in range(n):
        if i not in visited:
            dfs(i)
            count += 1
    
    return count
```

### Why It Works
- Each unvisited node starts a new component
- DFS marks the entire component as visited
- Count = number of DFS calls started
- O(V + E) time

---

## Problem 50 — Longest Consecutive Sequence (Medium)
**LeetCode #128**

### Problem
Given an unsorted array, find the length of the longest consecutive elements sequence. Must be O(n).

### Example
```
Input:  nums = [100,4,200,1,3,2]
Output: 4  (sequence: [1,2,3,4])
```

### Key Insight
Use a set. For each number, only start counting if it's the START of a sequence (num-1 not in set). Then count forward.

### Solution
```python
def longestConsecutive(nums):
    num_set = set(nums)
    max_len = 0
    
    for num in num_set:
        if num - 1 not in num_set:   # only start from the beginning of a sequence
            current = num
            length = 1
            while current + 1 in num_set:
                current += 1
                length += 1
            max_len = max(max_len, length)
    
    return max_len
```

### Why It Works
- Only start counting from sequence beginnings → each element counted at most once
- Set lookup is O(1) → total is O(n) even though there's a while loop inside a for loop
- O(n) time, O(n) space

---

# QUICK REFERENCE — PATTERN CHEAT SHEET

| Problem Type | Pattern | Key Idea |
|---|---|---|
| Find pair summing to X | HashMap | Store complement |
| Subarray max/min | Sliding window | Expand/shrink window |
| Sorted array search | Binary search | Eliminate half each step |
| Tree traversal | DFS recursion | base_case + recurse left + recurse right |
| Tree level-by-level | BFS + queue | snapshot level size at start |
| Graph connected | DFS/BFS + visited set | mark visited before recursing |
| Detect cycle | DFS with state (0/1/2) | in-progress = cycle |
| Multi-source spread | Multi-source BFS | Start all sources in queue |
| Character frequency | Counter / hashmap | Most string problems |
| No-repeat substring | Sliding window + set | Shrink on duplicate |

---

# DAILY SCHEDULE (2 problems/day)

| Day | Problems | Topic |
|---|---|---|
| 1 | 1, 2 | Arrays |
| 2 | 3, 4 | Arrays |
| 3 | 5, 6 | Arrays / Two Pointers |
| 4 | 7, 8 | Arrays / Binary Search |
| 5 | 9, 10 | Two Pointers / Binary Search |
| 6 | 11, 12 | Strings / Hashing |
| 7 | 13, 14 | Sliding Window / Stack |
| 8 | 15, 16 | Strings |
| 9 | 17, 18 | Sliding Window / Strings |
| 10 | 19, 20 | Strings |
| 11 | 21, 22 | Binary Search |
| 12 | 23, 24 | Binary Search |
| 13 | 25, 26 | Binary Search |
| 14 | 27 | Stretch problem |
| 15 | 28, 29 | Trees — CRITICAL |
| 16 | 30, 31 | Trees — CRITICAL (BFS) |
| 17 | 32, 33 | Trees (BST) |
| 18 | 34, 35 | Trees |
| 19 | 36, 37 | Trees |
| 20 | 38, 39 | Trees |
| 21 | 40 | Trees |
| 22 | 41, 42 | Graphs |
| 23 | 43, 44 | Graphs |
| 24 | 45, 46 | Graphs (Backtracking) |
| 25 | 47, 48 | Graphs (BFS) |
| 26 | 49, 50 | Graphs |
| 27–30 | Re-do ones you got wrong | Weak spots |
