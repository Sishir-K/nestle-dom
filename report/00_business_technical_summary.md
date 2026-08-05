# Business and Technical Summary

## What is Distributed Order Management?

When a company like Nestlé gets a customer order, that order is usually
assigned to one "default" distribution center (DC) — normally whichever one
is closest or historically handles that customer. Most of the time this
works fine. But sometimes that default DC doesn't have enough stock of
everything the order needs. When that happens, the company has a choice:
ship the order incomplete from the default DC (and eat a penalty for the
unmet demand), or reroute part or all of the order to a different DC that
does have the stock — even though shipping from farther away usually costs
more.

Distributed Order Management (DOM) is the general name for this
decision-making problem: given many orders, many DCs, limited inventory,
limited dock capacity, and real shipping costs, figure out the best way to
assign orders to DCs so the company keeps as much revenue as possible while
keeping penalty and shipping costs down.

## Why is this a combinatorial optimization problem?

The tricky part isn't deciding what to do with *one* order — that's easy,
you'd just pick whichever DC has the stock and is cheapest to ship from.
The hard part is that **every order is competing with every other order**
for the same limited inventory and dock capacity. If Order A gets assigned
to DC 5083 first, it might use up the last of a SKU that Order B also
needed from that same DC. So the "best" assignment for the whole batch of
orders isn't just the best individual choice repeated — it's a genuinely
combinatorial problem, where the number of possible ways to assign orders
to DCs grows extremely fast as more orders and DCs are added. For our
dataset alone (1,109 orders, 12 candidate DCs), the number of possible
combinations is astronomically larger than anything you could check one by
one.

## Why do advanced optimization methods matter here?

Because of that combinatorial explosion, "obvious" or greedy approaches
(just handle each order as it comes, one at a time) can get stuck making
locally sensible choices that turn out to be bad for the whole batch. We
actually demonstrated this directly: our simple greedy baseline, which
reassigns orders to alternate DCs one at a time, landed about 4.2% below
the true optimal profit — a real, measurable gap caused entirely by not
considering all orders together. Advanced optimization methods (like Mixed
Integer Linear Programming, or newer quantum-inspired approaches) are built
specifically to search this much larger space of combinations properly,
instead of settling for whatever the first reasonable-looking choice
happens to be.

## Trade-offs between solution quality and runtime

There's a real cost to getting the "best possible" answer: it takes longer.
In our project, solving the *exact* optimal assignment (using a Mixed
Integer Linear Program, or MILP) took about 7 minutes for the full
1,109-order dataset, and that time grew faster than the number of orders
did — going from 300 to 1,109 orders (about 3.7x more orders) made the
solve time about 8x longer, not just 3.7x longer. That's a real practical
concern: if Nestlé had 10,000 orders a day instead of 1,109, an exact
solver might become too slow to use in daily operations.

This is exactly why heuristic and quantum-inspired methods matter, even
though they don't guarantee the perfect answer. In our own testing, a
quantum-inspired solver (simulated annealing) reached about 91-95% of the
MILP's exact optimal profit at smaller scales (up to 100 orders), where it
also solved faster than the MILP. At full scale, this advantage did not
hold: a real bug in how the solver's constraints combine caused the
quantum-inspired method's accuracy to drop and its solve time to grow
slower than the MILP's. We traced this to the same underlying issue as a
bug we found and partly fixed earlier in the project, and a targeted
fix — splitting the problem by date — recovered some, but not all, of the
lost performance, landing at about 87% of the MILP's optimal profit. This
is an honest, useful finding in its own right: it shows that
quantum-inspired methods aren't simply "faster but slightly worse" at
every scale — their practical usefulness depends heavily on how the
problem is decomposed, and that trade-off needs to be evaluated at the
scale you actually intend to run at, not extrapolated from a smaller test.

## Exact methods vs. heuristics

**Exact methods** (like MILP) are guaranteed to find the true best answer,
and can prove it mathematically — when our MILP solver said "Optimal," we
know for certain there is no better assignment possible. The cost is
runtime, especially as the problem grows.

**Heuristic methods** (like our greedy baseline, or quantum-inspired
solvers like simulated annealing and Tabu search) don't come with that
guarantee — they find a *good* answer, often quickly, but there's no proof
it's the best possible one. In our project, we used an exact method (MILP)
specifically so we'd have a trustworthy benchmark, then compared heuristic
and quantum-inspired methods against it to see how close they could get.
This let us measure the real gap, instead of just assuming a heuristic
method's answer was good.

## Hardware vs. simulator constraints

For the quantum-inspired part of this project, we did not use real quantum
computing hardware. Instead, we used classical, quantum-*inspired* methods
(simulated annealing and Tabu search) that borrow ideas from how quantum
annealers work, but run entirely on ordinary computer hardware. Real
quantum hardware access (like D-Wave's cloud-hosted solvers) turned out to
be restricted for free-tier accounts during this project, so we built and
solved everything locally instead. This is a legitimate and common approach
for exploring these methods — it's how most people get started with
quantum-inspired optimization before ever touching real quantum
hardware — but it does mean our results reflect what's achievable on
classical simulators, not necessarily what a real quantum annealer or
gate-based quantum computer would produce.