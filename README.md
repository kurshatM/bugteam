# DADA3: A bug-fixing team

**Deployed app:** https://bug-fixing-team-9tbq.onrender.com

## Write-up

### Setup

Model was gemini-3.5-flash-lite, free tier, sandbox daytona, with one fresh sandbox per verification. One run of each system, I skipped the optional second run since it didn't fit my quota. No changes from the bundle.

### Results

From section 2 of notebook 03 (hidden tests, one run each):

```
team          resolved 9/10, triage 10/10, bugs fixed 6/7, false claims 1, 129 calls, 2,568,722 tokens, 683 s
baseline      resolved 7/10, triage 9/10, bugs fixed 5/7, false claims 3, 57 calls, 742,887 tokens, 215 s
```

### Did the team do better?

Yes. The team resolved 9/10 issues, 2 more issues resolved than the baseline's 7. Baseline also made three false claims but the team made only one. The team had a 10/10 triage as well. The difference is two issues/20 points of the percentage resolved. With a run of each system, I can't measure how much a single system varies between runs, so I'm not 100% on whether 20 points is larger than just luck. 

Something that the tests missed: both systems failed in the same way in slugify. Its test left out `slugify("Crème Brûlée Mug") == "creme-brulee-mug"`. The fix kept the accented letters, and that led to the tests the verifier wrote to be passed, but the hidden tests rejected it because of the accented letters. So the verifier here skipped the one docstring example that would've caught the bug. Same with the baseline.

### The handoff

There was one handback in the run, on #2 "shared payment loses a cent" (split_amount). 

The first fix used `divmod` and gave the extra cents to the first shares which is correct for positive amounts. For a negative amount, it gave us [-333, -333, -334] for split_amount(-1000, 3). Also, the verifier's reason quoted the docstring "the first shares get the extra cent. A negative amount (a refund) is split the same way as the positive amount, with every share negative", and its example split_amount(-1000, 3) == [-334, -333, -333]. And it was right, the verifier tested what the docstring said and the first fix shows that. The next fix addressed the reason directly though, it split the absolute value and negated every share for a negative amount. The verifier accepted it on round 2 and the hidden tests pass.

### What did it cost?

```
team: 129 calls, 2,568,722 tokens, 683 s
baseline: 57 calls, 742,887 tokens, 215 s
```

Around 3.5x the tokens, which is a very large amount for two more resolved issues and two less false claims. But it seems like a lot only because of our sample size. In real application use, it is absolutely worth it if this run is how it would be at scale, since two more here is 20% better, and 20% is a good margin of improvement. So I believe it is worth it. Also, making the verifier include every docstring exmaple in its test would have been a cheaper improvement.
