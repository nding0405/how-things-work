# Read-Copy-Update (RCU): The Art of Swapping Menus Without Angering Customers

## What is RCU?

RCU stands for Read-Copy-Update. It is a synchronization trick for data that gets read constantly but changed rarely. Think of it as the kernel's answer to this question: "How do you renovate a room while people are still standing in it?"

Here is the analogy. Imagine a restaurant that wants to update its menu. The naive approach: lock the doors, snatch every menu out of every customer's hands, swap in new ones, reopen. Customers hate this. This is what a lock does.

RCU is the chill approach:

1. Customers (readers) grab the current menu and browse it.
2. The chef (updater) prints a fresh menu in the back.
3. The chef puts the new menu on the stand. New customers get the new one. Old customers keep reading the old one, blissfully unaware.
4. The chef waits until every customer holding an old menu has left the building. Only then do the old menus hit the shredder.

That waiting step has a name: the **grace period**. It is the secret sauce of RCU.

One note: RCU is not a spinlock. It does nothing to stop two chefs from fighting over the printer. If multiple updaters modify the same structure, they still need a separate lock among themselves. RCU only makes life easy for readers.

## Who is responsible for what?

RCU is a contract, not magic. It protects an object's lifetime only if everyone plays their part.

**A reader must:**

1. Enter an RCU read-side critical section before grabbing the pointer. (Walk into the restaurant before picking up a menu.)
2. Obtain the pointer through an operation such as `rcu_dereference()`. (Take the menu from the stand, not from the trash.)
3. Stop using the pointer before leaving the critical section. (Put the menu down before you walk out. No takeaway menus.)

**An updater must:**

1. Remove the old object from the shared structure. (Take the old menu off the stand.)
2. Ensure that later readers cannot obtain it.
3. Defer reclamation with `kfree_rcu()`, `call_rcu()`, or a similar operation. (Do not shred menus while someone is still ordering from one.)

A pointer obtained under RCU is only safe inside that critical section. If you want it longer, you will need a separate lifetime guarantee, like a reference count. That is the takeaway-menu exception: you may leave with a menu, but only if you formally check it out first.

The [Linux RCU documentation](https://docs.kernel.org/RCU/whatisRCU.html) spells out this same division of responsibility, in slightly less delicious terms. 🍕

## Example

The following is conceptual pseudocode:

```c
reader()
{
    rcu_read_lock();

    p = rcu_dereference(current);
    if (p != NULL)
        use(p);

    rcu_read_unlock();
}

writer(new)
{
    lock(writer_lock);

    old = current;
    rcu_assign_pointer(current, new);

    unlock(writer_lock);

    if (old != NULL)
        kfree_rcu(old, rcu);
}
```

The updater publishes `new` immediately. It cannot free `old` immediately because an earlier reader may still be using it.

## Why not wait until there are no readers?

The simplest rule would be:

$$
\left(\exists R.\,\mathrm{Active}(R,\tau)\right)
\Longrightarrow
\neg\mathrm{Free}(o,\tau)
$$

In words:

> Do not free anything while any RCU reader exists.

This is safe, but too conservative. Readers may continuously overlap:

```text
R1: [----------)
R2:       [----------)
R3:             [----------)
R4:                   [----------)
```

Every reader eventually finishes, but there may never be a moment with zero readers.

RCU therefore waits only for readers that existed before a chosen boundary. This boundary is the start of a grace period.

## The grace-period rule

Let $R$ denote one dynamic, outermost RCU critical section. If locks are nested, the critical section begins when the nesting depth changes from $0$ to $1$, and ends when it changes from $1$ to $0$.

Let:

$$
s_R=\mathrm{start}(R)
$$

$$
e_R=\mathrm{end}(R)
$$

$$
s_n=\mathrm{start}(G_n)
$$

$$
e_n=\mathrm{end}(G_n)
$$

Here, $\lt$ means event order in the execution trace.

The required property is:

$$
\boxed{
\forall n\in\mathbb N,\forall R,\quad
s_R\lt s_n\Longrightarrow e_R\lt e_n
}
$$

In words:

> Every RCU critical section that began before $G_n$ must end before $G_n$ ends.

A reader that begins after $s_n$ does not need to delay $G_n$. Linux may conservatively wait for some such readers, but this only makes the grace period longer.

This is the [RCU grace-period guarantee](https://docs.kernel.org/RCU/Design/Requirements/Requirements.html).

## A simplified operational model

We now model normal preemptible RCU.

For grace period $G_n$, define:

- $W_n$: CPUs that still owe a quiescent-state report.
- $L$: active readers that have been preempted and recorded.
- $B_n\subseteq L$: recorded readers that block $G_n$.

The rules are:

1. At `StartGP(n)`, place every participating CPU that still needs to report into $W_n$.
2. At `StartGP(n)`, every reader already in $L$ becomes a blocker in $B_n$.
3. When a task is switched out inside an RCU critical section, add it to $L$, unless it is already listed.
4. If its CPU is still in $W_n$, add the reader to $B_n$ before removing the CPU from $W_n$.
5. If its CPU has already left $W_n$, the reader need not block $G_n$. It remains listed and may block the next GP.
6. Resuming a reader does not remove it from $L$ or $B_n$.
7. The outermost `rcu_read_unlock()` removes the reader from $L$ and $B_n$.
8. `EndGP(n)` is enabled only when:

$$
W_n=\varnothing
\land
B_n=\varnothing
$$

The important transfer is:

$$
\text{CPU obligation}
\longrightarrow
\text{task obligation}
$$

The task obligation must be created before the CPU obligation is cleared.

Linux does not literally store these mathematical sets. TREE_RCU uses hierarchical CPU masks, an ordered blocked-task list, and a `gp_tasks` boundary. These implement the same abstract obligations. See [TREE_RCU blocked-task management](https://docs.kernel.org/RCU/Design/Data-Structures/Data-Structures.html#blocked-task-management).

## Why is this correct?

We prove it by induction over machine transitions.

We do not use induction over grace-period numbers. The statement for $G_{n-1}$ does not prove that responsibility is transferred correctly during $G_n$. That is a property of individual state transitions.

### Lemma 1: every active reader is represented

Define:

$$
\mathrm{Active}(R,\tau)
\iff
s_R\le\tau\lt e_R
$$

At every time $\tau$:

$$
\boxed{
\mathrm{Active}(R,\tau)
\Longrightarrow
\mathrm{Listed}(R,\tau)
\lor
\exists c.\,\mathrm{Running}(R,c,\tau)
}
\tag{1}
$$

This uses OR, not XOR. A reader may resume while its blocked-reader entry still exists. It is then both running and listed.

Proof by induction over transitions:

1. Initially, no reader is active.
2. A reader can enter RCU only while running on a CPU.
3. Before an active reader is switched out, it is added to $L$.
4. When it resumes, its list entry remains.
5. At the outermost unlock, it becomes inactive and its entry is removed.
6. Other transitions do not affect the statement.

Therefore, (1) always holds.

### Lemma 2: every old active reader has an obligation

Define:

$$
\mathrm{Covered}_n(R,\tau)
\iff
R\in B_n(\tau)
\lor
\exists c.\left(
\mathrm{Running}(R,c,\tau)
\land
c\in W_n(\tau)
\right)
$$

We want to prove that, during $G_n$:

$$
\boxed{
s_R\lt s_n
\land
\mathrm{Active}(R,\tau)
\Longrightarrow
\mathrm{Covered}_n(R,\tau)
}
\tag{2}
$$

Immediately after `StartGP(n)`, Lemma 1 gives two cases:

1. $R$ is listed. GP start places it in $B_n$.
2. $R$ is running on CPU $c$. That CPU receives an obligation in $W_n$.

Therefore, every pre-existing active reader is covered at GP start.

Now assume (2) holds before the next transition.

- If $R$ keeps running, its CPU remains in $W_n$.
- If $R$ is switched out, it enters $B_n$ before its CPU leaves $W_n$.
- If $R$ resumes, its entry remains in $B_n$.
- If $R$ exits its critical section, it is no longer active.
- A new reader starts after $s_n$, so the premise does not apply.
- A CPU may leave $W_n$ only after its responsibility has ended or moved into $B_n$.

Thus every transition preserves (2).

A race between GP start and preemption has only two possible orders:

| Order | Result |
| --- | --- |
| Preemption happens first | $R$ enters $L$, then GP start places it in $B_n$ |
| GP start happens first | CPU $c$ enters $W_n$, then preemption transfers the obligation to $B_n$ |

There is no state in which both obligations are absent.

### The safety theorem

Assume, for contradiction, that:

$$
s_R\lt s_n
$$

but:

$$
e_R\ge e_n
$$

Then $R$ is still active immediately before `EndGP(n)`.

By Lemma 2, either:

$$
R\in B_n
$$

or:

$$
\exists c.\ c\in W_n
$$

Therefore:

$$
B_n\neq\varnothing
\lor
W_n\neq\varnothing
$$

But `EndGP(n)` requires:

$$
B_n=\varnothing
\land
W_n=\varnothing
$$

This is a contradiction. Therefore:

$$
s_R\lt s_n\Longrightarrow e_R\lt e_n
$$

Since $n$ was arbitrary:

$$
\boxed{
\forall n\in\mathbb N,\forall R,\quad
s_R\lt s_n\Longrightarrow e_R\lt e_n
}
$$

The operational mechanism therefore satisfies the grace-period rule.

## From grace periods to object safety

`kfree_rcu()` is not literally the start of a grace period. It queues deferred reclamation. The required grace period may start later, and the actual free may happen later still. The [kernel API documentation](https://docs.kernel.org/core-api/kernel-api.html#kfree-rcu) describes `kfree_rcu()` as freeing an object after a grace period.

Let:

- $u_o$: the old object is unpublished;
- $q_o$: `kfree_rcu(o)` is called;
- $G(o)$: a full grace period used for this request;
- $f_o$: the object is actually freed.

Then:

$$
u_o\le q_o\lt s(G(o))\lt e(G(o))\lt f_o
$$

If a valid reader $R$ can still hold $o$, it must have obtained the pointer before $o$ was unpublished. Therefore:

$$
s_R\lt u_o\lt s(G(o))
$$

By the grace-period theorem:

$$
e_R\lt e(G(o))
$$

Hence:

$$
e_R\lt e(G(o))\lt f_o
$$

So the object is not freed while any valid old reader may still use it.

This proof depends on the caller obligations. If the updater leaves the object published, or a reader uses the pointer after `rcu_read_unlock()`, the theorem no longer applies.

## Operational semantics of `kfree_rcu()`

Let:

- $S$ be the number of grace periods that have started;
- $C$ be the number of grace periods that have completed;
- $Q$ map pending objects to grace-period tickets;
- $F$ be the set of freed objects.

Calling `kfree_rcu(o)` records the current GP counter and requests a future grace period:

$$
\frac{
\mathrm{Unpublished}(o)
\land
o\notin\mathrm{dom}(Q)
\land
o\notin F
}{
\langle S,C,Q,F\rangle
\xrightarrow{\mathrm{kfree\_rcu}(o)}
\langle S,C,Q[o\mapsto S],F\rangle
}
$$

The call returns immediately. It does not free $o$.

The reclaim transition is:

$$
\frac{
Q(o)=k
\land
C\gt k
}{
\langle S,C,Q,F\rangle
\xrightarrow{\mathrm{reclaim}(o)}
\langle S,C,Q\setminus\{o\},F\cup\{o\}\rangle
}
$$

The condition $C\gt k$ means that at least one full grace period that started after the `kfree_rcu()` call has completed.

Linux may batch callbacks and run the reclaim transition later. Delaying reclamation affects performance, but not safety.
