# One-Page Planner View: Nestlé Order Fulfillment Optimization

## The Problem, in Plain Terms

When a customer order can't be fully filled from its usual distribution
center, someone has to decide: ship it incomplete, or send part of it from
a different center? Right now, this decision either happens order-by-order
(handling each one as it comes) or defaults to "just ship what's
available." Neither approach looks at the full picture — all orders,
competing for the same limited stock, at the same time.

## What We Built

We built and compared three ways of making this decision:

1. **Do nothing special** (default): every order stays at its home center,
   even if that center is short on stock.
2. **Quick fixes** (greedy): when a center runs short, look for another
   center with spare stock and reroute — one order at a time, first come,
   first served.
3. **See the whole picture** (optimizer): consider all orders and all
   centers together, and find the assignment that earns the most profit
   overall.

## What We Found

| Approach | Extra Profit vs. "Do Nothing" | Orders Fully Filled |
|---|---|---|
| Do nothing | — (baseline) | 95.8% |
| Quick fixes | +$5.2M | 98.3% |
| **See the whole picture** | **+$8.7M** | 97.2% |

The full-picture approach earns **$8.7M more** than doing nothing, and
**$3.5M more** than the quick-fix approach — by making smarter trade-offs
between which orders to fully serve, which to partially serve, and where
shipping costs make sense.

**Note:** the full-picture approach has a slightly *lower* fill rate than
quick fixes (97.2% vs 98.3%). That's not a mistake — it means the optimizer
sometimes chooses *not* to chase every last unit of demand, because the
shipping cost to do so would outweigh the penalty for leaving it unfilled.
It's optimizing for profit, not just for filling orders.

## What This Costs in Time

- The "quick fix" approach is instant.
- The "full picture" approach (guaranteed best answer) took about 7 minutes
  to run on our full order volume — a real but manageable cost for a
  meaningfully better result.
- We also tested a faster, "good enough" version (inspired by quantum
  computing techniques) that reached 91-97% of the best possible profit in
  well under a minute, and scaled predictably as order volume grew — a
  promising option if speed becomes more important than squeezing out the
  last few percent of profit.

## Bottom Line for Decision-Makers

Looking at the whole order book together, instead of one order at a time,
is worth real money — millions of dollars at this scale — and the
trade-off in solve time is small relative to the gain. As order volumes
grow, faster "good enough" methods offer a practical middle ground that
keeps most of the benefit without the full computational cost.