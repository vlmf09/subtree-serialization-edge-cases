# Subtree of Another Tree: The Two-Function Recursion, the O(m+n) Serialization Trick, and the Edge Cases That Cost You the Interview

LeetCode 572 is labelled Easy, and an accepted solution fits in nine lines. Roughly half of all submissions still fail. On the Chinese version of the same problem, a bit over 266,000 accepted submissions sit against about 540,000 attempts.

The idea is not what's hard. The idea is: walk `root`, and at every node ask whether the tree hanging there is identical to `subRoot`. What trips people up is that "identical" is doing more work in that sentence than they expect — identical from that node down to the leaves, same structure, same values, nothing extra underneath. Most wrong answers come from comparing values without comparing shape.

## What "subtree of another tree" actually means here

A subtree of `root` consists of one node in `root` plus all of that node's descendants. The tree itself counts as a subtree of itself. So the question is whether some node of `root` is the root of a tree that matches `subRoot` exactly.

Two examples make the distinction concrete:


root = [3,4,5,1,2]      subRoot = [4,1,2]        → true
root = [3,4,5,1,2,null,null,null,null,0]
                        subRoot = [4,1,2]        → false


In the first case, the subtree rooted at node `4` in `root` is `4 → 1, 2`, which matches `subRoot` node for node.

In the second case, that same `4` is still there. What changed is that node `2` now has a left child `0`. `subRoot` has no `0`. The node values of `subRoot` all appear in `root`, but the shape doesn't match, so the answer is `false`. This is the single most common source of failed submissions.

Constraints worth remembering while you write the solution: `root` holds between 1 and 2,000 nodes, `subRoot` between 1 and 1,000, and node values sit in the range −10⁴ to 10⁴. Both trees are non-empty.

> Subtree is not the same problem as *substructure*. A substructure match (the one in 剑指 Offer 26 / "Subtree of Another Tree" variants circulating in Chinese solution threads) allows the smaller tree to match only part of a branch — the smaller tree can run out of nodes while the larger one keeps going. Here, the smaller tree has to run out at exactly the same moment. If you copied a solution from the wrong problem, you'll pass the first sample and fail hidden tests.

## The two-function recursion, and where the base cases actually bite

The standard solution uses two recursive functions: one to hunt for a candidate node, one to compare two trees.

python
class Solution:
    def isSubtree(self, root, subRoot):
        if not subRoot:
            return True
        if not root:
            return False
        if self.isSameTree(root, subRoot):
            return True
        return self.isSubtree(root.left, subRoot) or self.isSubtree(root.right, subRoot)

    def isSameTree(self, p, q):
        if not p or not q:
            return p is q           # both None → True; one None → False
        return (p.val == q.val
                and self.isSameTree(p.left, q.left)
                and self.isSameTree(p.right, q.right))


The outer function visits every node of `root`. At each one it calls the inner comparator, which walks both trees in lockstep until something disagrees. `isSameTree` is exactly problem 100, "Same Tree" — if that one isn't automatic for you yet, solve it first. 572 is 100 with a traversal wrapped around it.

The base cases are where careless versions go wrong:

- `isSameTree(None, None)` must return `True`. Both trees ended at the same place — that's a match.
- `isSameTree(None, node)` or the reverse must return `False`. One tree still has structure left.
- Order matters in `isSubtree`: check `isSameTree` at the current node *before* recursing into children, otherwise you're doing extra work for no reason.

The `p is q` shortcut is worth understanding rather than memorising: when either pointer is `None`, identity comparison gives you the correct answer for both cases in one expression. It's compact, but if a version of this ever appears in an interview, be ready to expand it into an explicit `if` — some interviewers read the terse form as "wrote something clever, hoping it works."

## Complexity: O(m × n) is the honest answer, so say it out loud

| Approach | Time | Space | When it's the right answer |
| --- | --- | --- | --- |
| DFS + `isSameTree` helper | O(m × n) worst case | O(h) recursion depth | Default choice in an interview; passes easily within 2,000 / 1,000 node limits |
| Serialize both trees, substring search | O(m + n) expected | O(m + n) | Follow-up when the interviewer asks for better than quadratic |
| Bottom-up subtree hashing | O(m + n) expected | O(m + n) | Same follow-up, if you're more comfortable with hashing than string matching |

Here `m` is the node count of `root` and `n` is the node count of `subRoot`.

The quadratic bound isn't pessimism, it's real. Build `root` as a single path where every node holds the same value as `subRoot`'s root, and make `subRoot` a path of length `n`. Every one of the `m` candidate nodes triggers a comparison that survives several levels before failing on shape. You end up with something on the order of `m × n` comparisons. Candidates who have posted tighter-bound questions about this on Stack Overflow usually get the same answer: the worst case is genuinely quadratic, and the reason the naive solution feels fast in practice is that most candidates die at the first value comparison, which is O(1).

For the interview, the useful sentence is not "it's O(m × n)" — it's "worst case O(m × n), but each comparison usually fails at the root value check, and with `root ≤ 2,000` nodes the quadratic path is still trivial." Then offer the linear approach before you're asked. Interviewers who know this problem expect you to know both.

## The O(m + n) follow-up: serialize with markers, then match strings

The trick is to turn both trees into strings so you can use substring search instead of nested recursion.

Encode a tree in preorder, writing a marker for every `None` child. Then `subRoot`'s encoding appears as a contiguous substring of `root`'s encoding exactly when `subRoot` is a subtree of `root` — the null markers are what make the match align to real node boundaries instead of slicing through the middle of a branch.

python
def encode(node):
    if node is None:
        return "N,"
    return f"{node.val}," + encode(node.left) + encode(node.right)

def is_subtree(root, subRoot):
    return encode(subRoot) in encode(root)


Two traps here, and both are the reason this approach gets marked wrong in interviews more often than the quadratic one:

1. **No separators.** Node values are multi-digit and can be negative. If you concatenate raw values, the tree `[1, 2]` and the tree `[12]` produce overlapping strings. Use a delimiter (the comma above) or wrap every value — for example `-{val}-` — so distinct values can't bleed into each other.
2. **No null markers.** A preorder encoding without `None` markers can't distinguish `[4,1,2]` from `[4,1,2, ...anything...]` continuing below. The markers are not decoration; they're the part that makes the method correct.

If the interviewer pushes on the worst case of the substring search itself, mention KMP or Z-algorithm. Python's `in` is fast in practice but not guaranteed linear on adversarial input.

The hashing alternative is worth knowing too — LeetCode tags 572 under Hash Function alongside Tree, DFS, Binary Tree and String Matching, which is a hint about what the tag authors consider legitimate. Compute a hash for every subtree of `root` bottom-up (parent hash derived from value plus children's hashes), store them in a set, and compare against `subRoot`'s hash. Add a node-count check alongside the hash, because two different trees can collide and you'd rather not explain a hash collision in a 45-minute screen.

## Edge cases that actually show up

**`subRoot` is `None`.** LeetCode won't test it, but the conventional answer is `True` — the empty tree is a subtree of everything. Pick a convention, state it, be consistent.

**`root` is a single node and `subRoot` is the same node.** Should return `True`. Your traversal has to allow the match at the very first node rather than starting from the children.

**Many duplicate values.** A root of 2,000 nodes where half of them share `subRoot`'s root value means half your candidates run the full comparison. This is the case that turns the "it's fast in practice" claim into a slow submission.

**Deep recursion in Python.** A 2,000-node path is a legal input, and Python's default recursion limit is 1,000. Locally you can hit `RecursionError` on a skewed test even when the logic is right — either convert to an explicit stack or raise the limit before you conclude your solution is wrong.

**Structure mismatch with matching values.** Example 2 above. Write it as a test case before you submit, not after.

## Where this problem sits in a prep plan — and how AlgoMonster treats it

572 is one problem from one pattern: traverse one tree, compare against another, recurse. The same skeleton answers "Same Tree" (100), "Linked List in Binary Tree" (1367), "Symmetric Tree" (101, with the recursion mirrored between left and right), and a pile of company-specific variants that swap the payload from a tree to a list or a string.

That grouping is the whole premise behind AlgoMonster, a prep platform built by ex-Google engineers and competitive programmers that organises material around roughly 48 reusable patterns instead of a flat problem list. Its free Lite Problems section carries a full write-up for 572 itself, with Java and C++ implementations and the same "a subtree is a node plus all of its descendants" definition, so you can see how the pattern is presented before paying for anything.

The paid tiers unlock the pattern lessons themselves: tree and recursion chapters, code templates in several languages, decision flowcharts for picking a pattern from a problem statement, an AI assistant that reads your submitted code, Speedrun for timed review, company-specific question banks, and the editorial library — the site currently advertises 3,000+ editorials and 500+ lessons across 6 courses, up from the "325 lessons and problems" it listed on an earlier version of the same page. Two extra courses, a CS Fundamentals course and a "Build an LLM" one, are bundled rather than sold separately; some partner courses, like the quant interview prep, are not.

What it does not have: any video. The whole thing is text and diagrams, which is either the reason it's efficient or the reason it doesn't work for you, depending on how you learn. It also won't simulate interview pressure — no timer, no one asking why your complexity is what it is. Users who review it consistently flag the same gaps: limited system design depth for senior loops and a problem bank in the hundreds rather than thousands.

### Current plans and prices

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Sample patterns and problems — enough to judge whether the text-and-diagram format suits you | $0 | — | [ Start with the free tier](https://algo.monster/subscribe?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems; AI assistant; Speedrun; company question banks; bonus courses | $99/year ($8.25/mo billed annually), listed against $198 | 12 months, renews | [ Get Pro Yearly](https://algo.monster/subscribe?fpr=li69) |
| Pro – Lifetime | Everything in Yearly plus lifetime updates and an advertised internal referral to a top tech company for qualified candidates | $189 one-time, listed against $378 | One payment | [ Get Pro Lifetime](https://algo.monster/subscribe?fpr=li69) |

A Pro Monthly option at $18.99/month has appeared on the checkout at various points. If you see it, it's the most expensive way to buy 12 months of access — $227.88 versus $99 for the yearly plan — so it only makes sense if you genuinely want a single month.

Promo codes: a partner page run by a tech YouTuber currently documents the code `SCHELLEY` as stacking an extra 10% on top of the running 50% sale, which would bring the yearly to roughly $89 and the lifetime to roughly $170. The page states it was last verified on its own September update, and partner codes do get retired, so apply it at checkout and check the total before paying. Other codes like `FAANG20` circulate in review blogs with less reliable verification.

Worth flagging: several third-party reviews still quote a $459 lifetime price and $119–$299 annual plans. Those numbers don't match what the checkout page shows. Trust the checkout.

The break-even on lifetime is straightforward. If you buy the yearly twice, you've spent $198 for two years; the lifetime tier is $189 once. If you're prepping for a single job search over the next two months and have no plan to switch jobs again, the yearly plan is the cheaper commitment. If you expect to re-interview in two or three years — which is what most engineers actually do — lifetime is the smaller number.

And be honest about what a pattern course will and won't do for you on 572 specifically. It will make the two-function structure instinctive and hand you the reasoning for the linear follow-up. It will not stop you from walking into the interview and forgetting why `isSameTree(None, None)` is `True`. That part is reps.

## A short drill for this exact pattern

Ten minutes of deliberate work here beats another twenty random tree problems:

1. Solve "Same Tree" (100) without looking anything up. This is the inner function of 572.
2. Solve 572. Write down why `subRoot = None` returns `True`.
3. Re-derive the serialization version from scratch with a delimiter of your choosing, then try to break it with values `1` and `12` in the same tree.
4. Take example 2 and construct a second false case of your own where every value matches but the shape doesn't.
5. Explain out loud, in under 30 seconds, why the brute-force version is O(m × n) and how you'd get to O(m + n).

If you can do all five, this pattern is done. Then go do "Linked List in Binary Tree" and check whether the skeleton transfers without you having to think about it — that's the actual test of whether you learned a pattern or memorised a solution.

If you'd rather have the pattern taught to you in order, with the complexity analysis written down next to the code instead of assembled from six solution threads, that's what the [👉 pattern-based curriculum on AlgoMonster](https://algo.monster?fpr=li69) is for. Start on the free tier with 572, and upgrade only if the format is one you'll actually keep reading.

## FAQ

**Is "Subtree of Another Tree" really Easy?** LeetCode labels it Easy and tracks it as such. The label refers to the approach, not the success rate — about half of submissions fail, mostly from shape-versus-value confusion and from copying a substructure solution.

**Do I need KMP to pass 572?** No. With `root ≤ 2,000` nodes, the quadratic recursive solution passes comfortably. KMP matters only in the follow-up conversation about the substring version, and only if the interviewer asks about its worst case.

**Subtree or substructure — which one does this problem want?** Subtree: the smaller tree must match all the way down, and both trees must end at the same time. Substructure allows the smaller tree to end early while the larger one keeps going, and the code looks almost identical, which is exactly why it's a common wrong answer.

**Why does my serialized-string solution fail?** Almost always one of two reasons — no delimiter between values (so `1` and `12` collide), or no `None` markers (so the match isn't anchored to node boundaries).

**Is AlgoMonster worth it for one tree problem?** No, and it doesn't pretend to be priced that way. Its value is the ordered pattern library across all 48 patterns; if your only gap is 572, the free write-up and a few practice problems will cover it.
