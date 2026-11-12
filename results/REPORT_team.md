# huskyutils: the open issues

Fixes for issues #1, #2, #4, #5, #6, #8, and #10 were verified and are ready to review, no issues still need a person, and issues #3, #7, and #9 were closed without a code change.

| Issue | Triage | What happened |
|---|---|---|
| #1 Slugs have double hyphens and a hyphen at the end | bug | fix verified, 1 verifier round(s) |
| #2 Shared payment loses a cent | bug | fix verified, 2 verifier round(s) |
| #3 word_count says "don't stop" is 2 words | not_a_bug | closed, no code change |
| #4 Delivery date skips over the weekend wrong | bug | fix verified, 1 verifier round(s) |
| #5 Delivery windows overlap after merging | bug | fix verified, 1 verifier round(s) |
| #6 A 1 kg package is charged $9 shipping | bug | fix verified, 1 verifier round(s) |
| #7 Free shipping for orders over $50 | enhancement | closed, no code change |
| #8 Median on the sales dashboard is wrong for an even number of days | bug | fix verified, 1 verifier round(s) |
| #9 5 kg package should be in the heavy tier | not_a_bug | closed, no code change |
| #10 Product card description runs past the edge of the card | bug | fix verified, 1 verifier round(s) |

## #1: Slugs have double hyphens and a hyphen at the end

**Triage:** bug. The function slugify produces multiple consecutive hyphens and trailing hyphens when encountering multiple non-alphanumeric characters, which contradicts the docstring stating that runs of other characters become a single hyphen and slugs never end with a hyphen.

**Fix** (fix verified): Fixed slugify to collapse consecutive hyphens and strip leading/trailing hyphens as specified in the docstring.

```diff
--- a/huskyutils/text.py
+++ b/huskyutils/text.py
@@ -1,4 +1,5 @@
 """Text helpers for product pages."""
+import re
 
 
 def slugify(title):
@@ -16,7 +17,9 @@
     out = []
     for ch in title.lower():
         out.append(ch if ch.isalnum() else "-")
-    return "".join(out)
+    slug = "".join(out)
+    slug = re.sub(r"-+", "-", slug)
+    return slug.strip("-")
 
 
 def word_count(text):
```

**Verifier:** Ran tests in original and fixed directories using `python -m unittest discover -s tests -v`. In original/, test_docstring_examples and test_issue_cases failed, correctly reproducing the issue. In fixed/, all 27 tests passed successfully (including existing tests and our new test_issue.py covering issue cases and docstring examples).

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.text import slugify


class TestIssueSlugify(unittest.TestCase):
    def test_docstring_examples(self):
        self.assertEqual(slugify("Wireless Headphones"), "wireless-headphones")
        self.assertEqual(slugify("USB-C Dock (2nd gen)"), "usb-c-dock-2nd-gen")

    def test_issue_cases(self):
        self.assertEqual(slugify("Hello, World! T-shirt"), "hello-world-t-shirt")
        self.assertEqual(slugify("Webcam (HD)"), "webcam-hd")
```

</details>

## #2: Shared payment loses a cent

**Triage:** bug. The function `split_amount` simply integer-divides `cents` by `people`, missing the docstring requirement that shares differ by at most one cent and add up to exactly `cents`, losing remainders.

**Fix** (fix verified): Compute absolute value split and then negate shares for negative amounts so that the first shares get the extra absolute cent as required by the docstring.

```diff
--- a/huskyutils/money.py
+++ b/huskyutils/money.py
@@ -12,5 +12,12 @@
         split_amount(1000, 3) == [334, 333, 333]
         split_amount(-1000, 3) == [-334, -333, -333]
     """
-    share = cents // people
-    return [share] * people
+    abs_cents = abs(cents)
+    base = abs_cents // people
+    rem = abs_cents % people
+    shares = [base] * people
+    for i in range(rem):
+        shares[i] += 1
+    if cents < 0:
+        shares = [-s for s in shares]
+    return shares
```

**Verifier:** Ran unittest discover in original/ and fixed/ with newly written tests/test_issue.py. In original/, test_split_amount_issue_2 failed as expected with AssertionError: Lists differ: [333, 333, 333] != [334, 333, 333]. In fixed/, all 26 tests passed successfully (OK).

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.money import split_amount


class TestIssue(unittest.TestCase):
    def test_split_amount_issue_2(self):
        self.assertEqual(split_amount(900, 3), [300, 300, 300])
        self.assertEqual(split_amount(1000, 3), [334, 333, 333])
        self.assertEqual(split_amount(-1000, 3), [-334, -333, -333])
```

</details>

## #3: word_count says "don't stop" is 2 words

**Triage:** not_a_bug. The docstring explicitly specifies that punctuation stays attached to the word, so "don't" is considered one word. The code correctly implements this behavior, making the issue not a bug.

## #4: Delivery date skips over the weekend wrong

**Triage:** bug. The function adds calendar days rather than business days before checking weekends, causing incorrect results for days > 1. The code clearly does not match its docstring specification.

**Fix** (fix verified): Fixed add_business_days to add business days one by one instead of adding calendar days first.

```diff
--- a/huskyutils/dates.py
+++ b/huskyutils/dates.py
@@ -13,9 +13,9 @@
         add_business_days(date(2026, 11, 6), 1) == date(2026, 11, 9)                             # Monday
         add_business_days(date(2026, 11, 6), 1, holidays={date(2026, 11, 9)}) == date(2026, 11, 10)
     """
-    if days == 0:
-        return start
-    result = start + timedelta(days=days)
-    while result.weekday() >= 5 or result in holidays:
+    result = start
+    for _ in range(days):
         result += timedelta(days=1)
+        while result.weekday() >= 5 or result in holidays:
+            result += timedelta(days=1)
     return result
```

**Verifier:** Ran tests in original/ and fixed/ using python -m unittest discover -s tests -v. In original/, test_issue_thursday_plus_3 failed because add_business_days(date(2026, 11, 5), 3) returned date(2026, 11, 9) instead of date(2026, 11, 10). In fixed/, all 27 tests passed successfully.

<details><summary>The verifier's test</summary>

```python
import unittest
from datetime import date

from huskyutils.dates import add_business_days


class TestIssue4(unittest.TestCase):
    def test_docstring_examples(self):
        self.assertEqual(add_business_days(date(2026, 11, 6), 1), date(2026, 11, 9))
        self.assertEqual(add_business_days(date(2026, 11, 6), 1, holidays={date(2026, 11, 9)}), date(2026, 11, 10))

    def test_issue_thursday_plus_3(self):
        self.assertEqual(add_business_days(date(2026, 11, 5), 3), date(2026, 11, 10))
```

</details>

## #5: Delivery windows overlap after merging

**Triage:** bug. The docstring states that intervals can be in any order, but `merge_intervals` assumes input is sorted because it does not sort `intervals` before processing, causing out-of-order overlapping intervals to be missed.

**Fix** (fix verified): Sorted the intervals by start time before processing so that out-of-order overlapping intervals are correctly merged.

```diff
--- a/huskyutils/intervals.py
+++ b/huskyutils/intervals.py
@@ -13,8 +13,11 @@
         merge_intervals([[1, 3], [2, 6], [8, 10]]) == [[1, 6], [8, 10]]
         merge_intervals([[1, 4], [4, 5]]) == [[1, 5]]
     """
+    if not intervals:
+        return []
+    sorted_intervals = sorted(intervals, key=lambda x: x[0])
     merged = []
-    for start, end in intervals:
+    for start, end in sorted_intervals:
         if merged and start <= merged[-1][1]:
             merged[-1][1] = max(merged[-1][1], end)
         else:
```

**Verifier:** Ran `python -m unittest discover -s tests -v` in both `/home/daytona/original` and `/home/daytona/fixed`. The test `test_docstring_examples_and_issue` failed in `original` (reproducing Issue #5 where `merge_intervals([[8, 10], [1, 3], [2, 6]])` returned `[[8, 10]]` instead of `[[1, 6], [8, 10]]`), and all tests passed successfully in `fixed`.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.intervals import merge_intervals


class TestIssueIntervals(unittest.TestCase):
    def test_docstring_examples_and_issue(self):
        self.assertEqual(merge_intervals([[1, 3], [2, 6], [8, 10]]), [[1, 6], [8, 10]])
        self.assertEqual(merge_intervals([[1, 4], [4, 5]]), [[1, 5]])
        self.assertEqual(merge_intervals([[8, 10], [1, 3], [2, 6]]), [[1, 6], [8, 10]])
```

</details>

## #6: A 1 kg package is charged $9 shipping

**Triage:** bug. The docstring states that packages up to and including 1 kg cost $5.00. However, the code checks `if weight_kg < 1:`, meaning a weight of exactly 1 kg falls through to the $9.00 tier, which contradicts the specification.

**Fix** (fix verified): Changed weight_kg < 1 to weight_kg <= 1 so that packages up to and including 1 kg are charged $5.00 as per the docstring.

```diff
--- a/huskyutils/shipping.py
+++ b/huskyutils/shipping.py
@@ -12,7 +12,7 @@
         shipping_cost(0.4) == 500
         shipping_cost(5.2) == 1100
     """
-    if weight_kg < 1:
+    if weight_kg <= 1:
         return 500
     if weight_kg <= 5:
         return 900
```

**Verifier:** Ran `python -m unittest discover -s tests -v` in both original and fixed directories. In original/ repository, the new test `test_issue_6_exact_1kg` failed with `AssertionError: 900 != 500`. In fixed/ repository, all 27 tests passed successfully.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.shipping import shipping_cost


class TestIssueShipping(unittest.TestCase):
    def test_docstring_examples(self):
        self.assertEqual(shipping_cost(0.4), 500)
        self.assertEqual(shipping_cost(5.2), 1100)

    def test_issue_6_exact_1kg(self):
        self.assertEqual(shipping_cost(1), 500)
```

</details>

## #7: Free shipping for orders over $50

**Triage:** enhancement. The shipping_cost function calculates prices based on weight, while the request asks for a new feature for order-total-based free shipping, which is not mentioned in the docstring.

## #8: Median on the sales dashboard is wrong for an even number of days

**Triage:** bug. The median function for an even list returns the upper middle element instead of the average of the two middle values, which violates its docstring specification.

**Fix** (fix verified): Fixed median calculation for lists with an even number of values to return the average of the two middle values instead of the upper middle element.

```diff
--- a/huskyutils/stats.py
+++ b/huskyutils/stats.py
@@ -11,4 +11,8 @@
         median([4, 1, 3, 2]) == 2.5
     """
     ordered = sorted(values)
-    return ordered[len(ordered) // 2]
+    n = len(ordered)
+    if n % 2 == 1:
+        return ordered[n // 2]
+    else:
+        return (ordered[n // 2 - 1] + ordered[n // 2]) / 2
```

**Verifier:** Ran tests in original/ and fixed/ using `python -m unittest discover -s tests -v`. In original/, test_median_even and test_docstring_examples failed with AssertionError (median returned 30 and 3 instead of 25 and 2.5). In fixed/, all 27 tests passed successfully.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.stats import median


class TestIssue(unittest.TestCase):
    def test_median_even(self):
        self.assertEqual(median([10, 20, 30, 40]), 25)

    def test_docstring_examples(self):
        self.assertEqual(median([3, 1, 2]), 2)
        self.assertEqual(median([4, 1, 3, 2]), 2.5)
```

</details>

## #9: 5 kg package should be in the heavy tier

**Triage:** not_a_bug. The docstring states that packages over 5 kg cost $9.00 plus $2.00 for each started kilogram over 5, and packages up to and including 5 kg cost $9.00. A 5 kg package correctly costs $9.00 according to the specification.

## #10: Product card description runs past the edge of the card

**Triage:** bug. The function truncate() slices text at limit and appends '...', causing the result to exceed the specified limit when truncated. The docstring specifies the result is at most limit characters long.

**Fix** (fix verified): Implemented word-aware truncation so that the resulting truncated text does not exceed `limit` characters, adhering strictly to the docstring specification.

```diff
--- a/huskyutils/cards.py
+++ b/huskyutils/cards.py
@@ -15,4 +15,24 @@
     """
     if len(text) <= limit:
         return text
-    return text[:limit] + "..."
+
+    words = text.split()
+    current = ""
+    for word in words:
+        if current:
+            candidate = current + " " + word + "..."
+        else:
+            candidate = word + "..."
+
+        if len(candidate) <= limit:
+            if current:
+                current = current + " " + word
+            else:
+                current = word
+        else:
+            break
+
+    if current:
+        return current + "..."
+
+    return text[:limit - 3] + "..."
```

**Verifier:** Ran tests in original/ where test_issue failed with AssertionError as expected (reproducing issue #10), and ran tests in fixed/ where all tests passed successfully.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.cards import truncate


class TestIssue10Truncate(unittest.TestCase):
    def test_docstring_examples(self):
        self.assertEqual(truncate("Wireless headphones", 30), "Wireless headphones")
        self.assertEqual(truncate("Wireless noise-cancelling headphones with case", 30), "Wireless noise-cancelling...")

    def test_truncate_length(self):
        s = truncate("Wireless noise-cancelling headphones with case", 30)
        self.assertEqual(s, "Wireless noise-cancelling...")
        self.assertLessEqual(len(s), 30)
```

</details>