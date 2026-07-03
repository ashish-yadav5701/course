
# DSA Solutions: String and Array Problems - Detailed Questions & Interview Answers

---

## Array Problems

### 1. Two Sum

**Problem:**
Given an array of integers `nums` and an integer `target`, return the indices of the two numbers that add up to `target`. You may assume that each input has exactly one solution, and you may not use the same element twice. Return the answer in any order.

**Example:**
```

Input: nums = [2,7,11,15], target = 9
Output: [0,1]
Explanation: nums[0] + nums[1] = 2 + 7 = 9

```

**Interview Explanation:**
"To solve this efficiently, I'll use a HashMap to store each number and its index as I iterate. For each element, I calculate the complement (target - current). If the complement exists in the map, we've found the pair. Otherwise, I add the current number and its index to the map. This gives O(n) time and O(n) space."

**Solution:**
```java
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> map = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (map.containsKey(complement)) {
            return new int[]{map.get(complement), i};
        }
        map.put(nums[i], i);
    }
    return new int[]{-1, -1}; // should not reach
}
```

Complexity: O(n) time, O(n) space.

Follow-up: If the array is sorted, we can use two pointers with O(1) space.

---

2. Best Time to Buy and Sell Stock

Problem:
You are given an array prices where prices[i] is the price of a given stock on day i. You want to maximize your profit by choosing a single day to buy one stock and a different day in the future to sell that stock. Return the maximum profit you can achieve. If no profit is possible, return 0.

Example:

```
Input: prices = [7,1,5,3,6,4]
Output: 5
Explanation: Buy on day 2 (price = 1) and sell on day 5 (price = 6), profit = 5.
```

Interview Explanation:
"We track the minimum price seen so far. As we iterate, if we see a lower price, we update the minimum. Otherwise, we calculate the profit if we sell at current price and update the maximum profit. This is a single-pass solution with constant space."

Solution:

```java
public int maxProfit(int[] prices) {
    int minPrice = Integer.MAX_VALUE;
    int maxProfit = 0;
    for (int price : prices) {
        if (price < minPrice) {
            minPrice = price;
        } else {
            maxProfit = Math.max(maxProfit, price - minPrice);
        }
    }
    return maxProfit;
}
```

Complexity: O(n) time, O(1) space.

Follow-up: If multiple transactions are allowed, sum all positive differences between consecutive days.

---

3. Maximum Subarray (Kadane’s Algorithm)

Problem:
Given an integer array nums, find the contiguous subarray (containing at least one number) which has the largest sum and return its sum.

Example:

```
Input: nums = [-2,1,-3,4,-1,2,1,-5,4]
Output: 6
Explanation: [4,-1,2,1] has the largest sum = 6.
```

Interview Explanation:
"I use Kadane's algorithm which maintains a running sum of the best subarray ending at the current position. At each element, we decide whether to extend the current subarray or start a new one. We also keep track of the global maximum seen so far."

Solution:

```java
public int maxSubArray(int[] nums) {
    int currentMax = nums[0];
    int globalMax = nums[0];
    for (int i = 1; i < nums.length; i++) {
        currentMax = Math.max(nums[i], currentMax + nums[i]);
        globalMax = Math.max(globalMax, currentMax);
    }
    return globalMax;
}
```

Complexity: O(n) time, O(1) space.

Follow-up: To return the subarray itself, track start and end indices when updating globalMax.

---

4. Move Zeroes

Problem:
Given an array nums, write a function to move all 0s to the end while maintaining the relative order of the non-zero elements. You must do this in-place without making a copy of the array.

Example:

```
Input: nums = [0,1,0,3,12]
Output: [1,3,12,0,0]
```

Interview Explanation:
"I use a two-pointer approach where one pointer (nonZeroIndex) tracks where the next non-zero element should be placed. I iterate through the array, and whenever I encounter a non-zero, I place it at the nonZeroIndex and increment it. After the loop, I fill the remaining positions with zeros."

Solution:

```java
public void moveZeroes(int[] nums) {
    int nonZeroIndex = 0;
    for (int i = 0; i < nums.length; i++) {
        if (nums[i] != 0) {
            nums[nonZeroIndex++] = nums[i];
        }
    }
    while (nonZeroIndex < nums.length) {
        nums[nonZeroIndex++] = 0;
    }
}
```

Complexity: O(n) time, O(1) space.

Follow-up: We can also swap non-zero elements with zeros to maintain order without the second pass.

---

5. Find the Duplicate Number

Problem:
Given an array nums containing n + 1 integers where each integer is between 1 and n (inclusive), prove that at least one duplicate number must exist. Assume there is only one duplicate number, find it. You must not modify the array and use only constant extra space.

Example:

```
Input: nums = [1,3,4,2,2]
Output: 2
```

Interview Explanation:
"This is a classic cycle detection problem. Because the numbers are in range [1, n] and the array size is n+1, we can treat the array as a linked list where nums[i] is the next index. A duplicate creates a cycle. Using Floyd's Tortoise and Hare algorithm, we find the intersection point, then the start of the cycle, which is the duplicate number."

Solution:

```java
public int findDuplicate(int[] nums) {
    int slow = nums[0];
    int fast = nums[0];
    
    // Find intersection
    do {
        slow = nums[slow];
        fast = nums[nums[fast]];
    } while (slow != fast);
    
    // Find start of cycle (duplicate)
    slow = nums[0];
    while (slow != fast) {
        slow = nums[slow];
        fast = nums[fast];
    }
    return slow;
}
```

Complexity: O(n) time, O(1) space.

Follow-up: Binary search with counting approach is also valid but O(n log n).

---

6. Container With Most Water

Problem:
You are given an integer array height of length n. There are n vertical lines drawn such that the two endpoints of the i-th line are (i,0) and (i, height[i]). Find two lines that together with the x-axis form a container that holds the most water. Return the maximum amount of water a container can store.

Example:

```
Input: height = [1,8,6,2,5,4,8,3,7]
Output: 49
Explanation: The max area is between lines at indices 1 and 8: height = min(8,7)=7, width=7, area=49.
```

Interview Explanation:
"I use two pointers starting at both ends. The area is determined by the shorter line, so to maximize area, I move the pointer with the smaller height inward. I compute the area at each step and keep track of the maximum."

Solution:

```java
public int maxArea(int[] height) {
    int left = 0, right = height.length - 1;
    int maxArea = 0;
    while (left < right) {
        int minHeight = Math.min(height[left], height[right]);
        maxArea = Math.max(maxArea, minHeight * (right - left));
        if (height[left] < height[right]) left++;
        else right--;
    }
    return maxArea;
}
```

Complexity: O(n) time, O(1) space.

Follow-up: We can skip lines that are shorter than the current minimum height to optimize further.

---

7. Product of Array Except Self

Problem:
Given an integer array nums, return an array answer such that answer[i] is equal to the product of all the elements of nums except nums[i]. You must write an algorithm that runs in O(n) time and without using the division operation.

Example:

```
Input: nums = [1,2,3,4]
Output: [24,12,8,6]
```

Interview Explanation:
"Without division, I compute prefix and suffix products. In the first pass, I store the product of all elements to the left of each position in the result array. In the second pass, I multiply each position by the product of all elements to its right, maintaining a running right product variable."

Solution:

```java
public int[] productExceptSelf(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    
    // Prefix products
    int leftProduct = 1;
    for (int i = 0; i < n; i++) {
        result[i] = leftProduct;
        leftProduct *= nums[i];
    }
    
    // Suffix products
    int rightProduct = 1;
    for (int i = n - 1; i >= 0; i--) {
        result[i] *= rightProduct;
        rightProduct *= nums[i];
    }
    return result;
}
```

Complexity: O(n) time, O(1) extra space (besides output).

Follow-up: If zeros are present, we can handle by counting zeros.

---

8. Merge Sorted Array

Problem:
You are given two integer arrays nums1 and nums2, sorted in non-decreasing order, and two integers m and n representing the number of elements in nums1 and nums2 respectively. Merge nums2 into nums1 so that the resulting array is sorted in-place. nums1 has a length of m + n where the first m elements are valid and the rest are zeros.

Example:

```
Input: nums1 = [1,2,3,0,0,0], m = 3, nums2 = [2,5,6], n = 3
Output: [1,2,2,3,5,6]
```

Interview Explanation:
"I merge from the back to avoid overwriting elements in nums1. I start from the last valid element of both arrays and the last position of the combined array. I compare the largest elements and place the larger at the end. If any elements remain in nums2, I copy them over."

Solution:

```java
public void merge(int[] nums1, int m, int[] nums2, int n) {
    int p1 = m - 1, p2 = n - 1, p = m + n - 1;
    while (p1 >= 0 && p2 >= 0) {
        nums1[p--] = (nums1[p1] > nums2[p2]) ? nums1[p1--] : nums2[p2--];
    }
    while (p2 >= 0) {
        nums1[p--] = nums2[p2--];
    }
}
```

Complexity: O(m + n) time, O(1) space.

Follow-up: If nums1 doesn't have enough space, we'd need extra array.

---

9. 3Sum

Problem:
Given an integer array nums, return all the triplets [nums[i], nums[j], nums[k]] such that i != j, i != k, j != k, and nums[i] + nums[j] + nums[k] == 0. The solution set must not contain duplicate triplets.

Example:

```
Input: nums = [-1,0,1,2,-1,-4]
Output: [[-1,-1,2],[-1,0,1]]
```

Interview Explanation:
"I sort the array to enable the two-pointer technique. For each element, I fix it as the first element and use two pointers to find pairs that sum to its negative. To avoid duplicates, I skip over duplicate values for the first element and also skip duplicates when I find a valid triplet."

Solution:

```java
public List<List<Integer>> threeSum(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    Arrays.sort(nums);
    for (int i = 0; i < nums.length - 2; i++) {
        if (i > 0 && nums[i] == nums[i-1]) continue;
        int left = i+1, right = nums.length-1;
        while (left < right) {
            int sum = nums[i] + nums[left] + nums[right];
            if (sum == 0) {
                result.add(Arrays.asList(nums[i], nums[left], nums[right]));
                while (left < right && nums[left] == nums[left+1]) left++;
                while (left < right && nums[right] == nums[right-1]) right--;
                left++; right--;
            } else if (sum < 0) {
                left++;
            } else {
                right--;
            }
        }
    }
    return result;
}
```

Complexity: O(n²) time, O(1) extra space (excluding output).

Follow-up: For 4Sum, we'd add an outer loop and use the same two-pointer approach.

---

10. Next Permutation

Problem:
Implement the next permutation, which rearranges numbers into the lexicographically next greater permutation. If such arrangement is not possible, it must rearrange to the lowest possible order (i.e., sorted in ascending order). The replacement must be in-place and use only constant extra memory.

Example:

```
Input: nums = [1,2,3]
Output: [1,3,2]
Input: nums = [3,2,1]
Output: [1,2,3]
```

Interview Explanation:
"The algorithm has three steps: First, find the first decreasing element from the right (pivot). Second, find the element just greater than the pivot from the right. Third, swap them and reverse the suffix after the pivot. This gives the next permutation. If no pivot is found, the array is in descending order, so we reverse the whole array."

Solution:

```java
public void nextPermutation(int[] nums) {
    int i = nums.length - 2;
    while (i >= 0 && nums[i] >= nums[i+1]) i--;
    if (i >= 0) {
        int j = nums.length - 1;
        while (nums[j] <= nums[i]) j--;
        swap(nums, i, j);
    }
    reverse(nums, i+1, nums.length - 1);
}
private void swap(int[] nums, int i, int j) {
    int temp = nums[i]; nums[i] = nums[j]; nums[j] = temp;
}
private void reverse(int[] nums, int start, int end) {
    while (start < end) swap(nums, start++, end--);
}
```

Complexity: O(n) time, O(1) space.

Follow-up: If input is the largest permutation, reverse to smallest.

---

String Problems

11. Reverse String

Problem:
Write a function that reverses a string. The input string is given as a character array s. You must do this by modifying the input array in-place with O(1) extra memory.

Example:

```
Input: s = ["h","e","l","l","o"]
Output: ["o","l","l","e","h"]
```

Interview Explanation:
"I use two pointers, one at the start and one at the end, and swap characters until they meet."

Solution:

```java
public void reverseString(char[] s) {
    int left = 0, right = s.length - 1;
    while (left < right) {
        char temp = s[left];
        s[left] = s[right];
        s[right] = temp;
        left++; right--;
    }
}
```

Complexity: O(n) time, O(1) space.

Follow-up: Recursive approach is possible but uses O(n) stack space.

---

12. Valid Palindrome

Problem:
A phrase is a palindrome if, after converting all uppercase letters to lowercase and removing all non-alphanumeric characters, it reads the same forward and backward. Given a string s, return true if it is a palindrome, or false otherwise.

Example:

```
Input: s = "A man, a plan, a canal: Panama"
Output: true
```

Interview Explanation:
"I use two pointers and skip non-alphanumeric characters. I compare characters after converting to lowercase."

Solution:

```java
public boolean isPalindrome(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        while (left < right && !Character.isLetterOrDigit(s.charAt(left))) left++;
        while (left < right && !Character.isLetterOrDigit(s.charAt(right))) right--;
        if (Character.toLowerCase(s.charAt(left)) != Character.toLowerCase(s.charAt(right))) {
            return false;
        }
        left++; right--;
    }
    return true;
}
```

Complexity: O(n) time, O(1) space.

Follow-up: If we want to consider spaces and punctuation, we'd not skip them.

---

13. Valid Anagram

Problem:
Given two strings s and t, return true if t is an anagram of s, and false otherwise. An anagram is a word or phrase formed by rearranging the letters.

Example:

```
Input: s = "anagram", t = "nagaram"
Output: true
```

Interview Explanation:
"I use a frequency array of size 26 for lowercase letters. I increment for each character in s, then decrement for t. If any count becomes negative, they are not anagrams."

Solution:

```java
public boolean isAnagram(String s, String t) {
    if (s.length() != t.length()) return false;
    int[] count = new int[26];
    for (char c : s.toCharArray()) count[c-'a']++;
    for (char c : t.toCharArray()) {
        count[c-'a']--;
        if (count[c-'a'] < 0) return false;
    }
    return true;
}
```

Complexity: O(n) time, O(1) space.

Follow-up: For Unicode, use HashMap.

---

14. Longest Substring Without Repeating Characters

Problem:
Given a string s, find the length of the longest substring without repeating characters.

Example:

```
Input: s = "abcabcbb"
Output: 3
Explanation: The answer is "abc", with length 3.
```

Interview Explanation:
"I use the sliding window technique with a HashSet. As I expand the right pointer, if the current character is already in the set, I remove characters from the left until the duplicate is gone. I update the maximum window length."

Solution:

```java
public int lengthOfLongestSubstring(String s) {
    Set<Character> set = new HashSet<>();
    int left = 0, max = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        while (set.contains(c)) {
            set.remove(s.charAt(left++));
        }
        set.add(c);
        max = Math.max(max, right - left + 1);
    }
    return max;
}
```

Complexity: O(n) time, O(min(n, 26)) space.

Follow-up: Optimize with HashMap to jump left directly.

---

15. Group Anagrams

Problem:
Given an array of strings strs, group the anagrams together. You can return the answer in any order. An anagram is a word formed by rearranging the letters.

Example:

```
Input: strs = ["eat","tea","tan","ate","nat","bat"]
Output: [["bat"],["nat","tan"],["ate","eat","tea"]]
```

Interview Explanation:
"The key is that anagrams have the same sorted character sequence. I sort each string's characters and use that as a key in a HashMap. The value is a list of strings that share that key."

Solution:

```java
public List<List<String>> groupAnagrams(String[] strs) {
    Map<String, List<String>> map = new HashMap<>();
    for (String str : strs) {
        char[] chars = str.toCharArray();
        Arrays.sort(chars);
        String sorted = new String(chars);
        map.computeIfAbsent(sorted, k -> new ArrayList<>()).add(str);
    }
    return new ArrayList<>(map.values());
}
```

Complexity: O(n * k log k) time, O(n * k) space, where k is max string length.

Follow-up: Use character count as key to avoid sorting.

---

16. Longest Palindromic Substring

Problem:
Given a string s, return the longest palindromic substring in s.

Example:

```
Input: s = "babad"
Output: "bab"
Explanation: "aba" is also a valid answer.
```

Interview Explanation:
"I use expand-around-center. There are 2n-1 centers (each character and between each pair). For each center, I expand while characters match and track the longest length and start index."

Solution:

```java
public String longestPalindrome(String s) {
    int start = 0, maxLen = 1;
    for (int i = 0; i < s.length(); i++) {
        int len1 = expand(s, i, i);
        int len2 = expand(s, i, i+1);
        int len = Math.max(len1, len2);
        if (len > maxLen) {
            maxLen = len;
            start = i - (len-1)/2;
        }
    }
    return s.substring(start, start + maxLen);
}
private int expand(String s, int left, int right) {
    while (left >= 0 && right < s.length() && s.charAt(left) == s.charAt(right)) {
        left--; right++;
    }
    return right - left - 1;
}
```

Complexity: O(n²) time, O(1) space.

Follow-up: Manacher's algorithm gives O(n) time.

---

17. String to Integer (atoi)

Problem:
Implement the myAtoi(string s) function, which converts a string to a 32-bit signed integer. The algorithm should handle leading whitespace, optional '+'/'-' sign, digits, and clamp to [-2^31, 2^31 - 1]. Ignore any characters after the valid number.

Example:

```
Input: s = "   -42"
Output: -42
Input: s = "4193 with words"
Output: 4193
Input: s = "words and 987"
Output: 0
```

Interview Explanation:
"I iterate through the string, skipping whitespace, handling sign, then converting digits. I detect overflow before multiplying by checking against Integer.MAX_VALUE/10 and the digit."

Solution:

```java
public int myAtoi(String s) {
    int i = 0, n = s.length(), sign = 1, result = 0;
    while (i < n && s.charAt(i) == ' ') i++;
    if (i < n && (s.charAt(i) == '+' || s.charAt(i) == '-')) {
        sign = (s.charAt(i) == '-') ? -1 : 1;
        i++;
    }
    while (i < n && Character.isDigit(s.charAt(i))) {
        int digit = s.charAt(i) - '0';
        if (result > Integer.MAX_VALUE/10 || (result == Integer.MAX_VALUE/10 && digit > 7)) {
            return (sign == 1) ? Integer.MAX_VALUE : Integer.MIN_VALUE;
        }
        result = result * 10 + digit;
        i++;
    }
    return sign * result;
}
```

Complexity: O(n) time, O(1) space.

Follow-up: Using long simplifies overflow check but uses extra space.

---

18. Implement strStr() / Find the Index of the First Occurrence

Problem:
Given two strings haystack and needle, return the index of the first occurrence of needle in haystack, or -1 if needle is not part of haystack. If needle is empty, return 0.

Example:

```
Input: haystack = "sadbutsad", needle = "sad"
Output: 0
```

Interview Explanation:
"I use a simple sliding window. For each possible start index where the remaining length is sufficient, I check if the substring matches the needle. If a mismatch occurs, I break and move to the next start."

Solution:

```java
public int strStr(String haystack, String needle) {
    if (needle.isEmpty()) return 0;
    int n = haystack.length(), m = needle.length();
    for (int i = 0; i <= n - m; i++) {
        int j = 0;
        while (j < m && haystack.charAt(i+j) == needle.charAt(j)) j++;
        if (j == m) return i;
    }
    return -1;
}
```

Complexity: O(n*m) worst case, O(n+m) average with KMP.

Follow-up: KMP or Boyer-Moore for better performance.

---

19. Valid Parentheses

Problem:
Given a string s containing just the characters '(', ')', '{', '}', '[' and ']', determine if the input string is valid. An input string is valid if brackets are closed in the correct order.

Example:

```
Input: s = "()[]{}"
Output: true
Input: s = "([)]"
Output: false
```

Interview Explanation:
"I use a stack. For opening brackets, I push. For closing, I check if the stack is empty or the top doesn't match, then it's invalid. At the end, the stack must be empty."

Solution:

```java
public boolean isValid(String s) {
    Stack<Character> stack = new Stack<>();
    Map<Character, Character> map = Map.of(')', '(', ']', '[', '}', '{');
    for (char c : s.toCharArray()) {
        if (map.containsKey(c)) {
            if (stack.isEmpty() || stack.pop() != map.get(c)) return false;
        } else {
            stack.push(c);
        }
    }
    return stack.isEmpty();
}
```

Complexity: O(n) time, O(n) space.

Follow-up: Use ArrayDeque for better performance.

---

20. String Compression

Problem:
Given an array of characters chars, compress it in-place using the following algorithm: Begin with an empty string s. For each group of consecutive repeating characters, append the character followed by the group's length if the length is greater than 1. Return the new length of the array. You must do this in-place.

Example:

```
Input: chars = ["a","a","b","b","c","c","c"]
Output: Return 6, and the first 6 chars of input should be ["a","2","b","2","c","3"]
```

Interview Explanation:
"I use two pointers: a read pointer to traverse the array and a write pointer to build the compressed result. For each group of identical characters, I count them, write the character, and if count > 1, write the digits of the count."

Solution:

```java
public int compress(char[] chars) {
    int write = 0, read = 0;
    while (read < chars.length) {
        char current = chars[read];
        int count = 0;
        while (read < chars.length && chars[read] == current) {
            count++; read++;
        }
        chars[write++] = current;
        if (count > 1) {
            for (char c : String.valueOf(count).toCharArray()) {
                chars[write++] = c;
            }
        }
    }
    return write;
}
```

Complexity: O(n) time, O(1) space.

Follow-up: If the compressed string is longer, return original length.

