---
title: "Positive measure forces long progressions"
date: 2026-09-30
description: "Any set with positive Lebesgue measure contains arbitrarily long arithmetic progressions"
tags: [math]
---

# Positive measure forces long progressions

Wanted to share a really cool problem I saw recently on my analysis midterm:

> An arithmetic progression on $$\mathbb{R}$$ of length $$k$$ is a sequence of real numbers of the form $$a, a + h, \ldots, a + (k - 1)h$$ for some $$a, h \in \mathbb{R}$$ and $$h > 0$$. Show that any Lebesgue measurable subset of $$\mathbb{R}$$ of positive and finite Lebesgue measure contains arbitrarily long arithmetic progressions on $$\mathbb{R}$$.

# Solution 

Being Lebesgue measurable is a statement about utility, let's convert to a statement about geometry / structure, which feels more useful. Let $$E$$ be our set and $$\lambda$$ denote Lebesgue measure. Using inner regularity[^regularity], for every $$\varepsilon > 0$$ we can find some compact $$K \subseteq E$$ such that $$\lambda(E) - \lambda(K) < \varepsilon$$, in particular one with $$\lambda(K) > 0$$. Since any compact set is also Lebesgue measurable[^compact], it is necessary and sufficient to consider compact subsets instead of Lebesgue measurable subsets.

To effectively leverage compactness, we should use the language of sets[^intersection]:

$$
K \text{ contains a length-}N \text{ arithmetic progression} \iff \exists\, h > 0 : \bigcap_{j=0}^{N-1} (K + jh) \neq \emptyset
$$

Intuitively, if we can somehow bound the measure of the container all $$N$$ sets lie in, this could force their intersection to be non-empty.

![N translated copies of K inside a container, overlapping in a shaded intersection](/images/notes/analysis-prob/fig1_intersection.png){: .kv-fig .narrow}

*Fig 1. Illustration of intuition only; in practice, $$K$$ doesn't need to contain any intervals. For example, [the fat Cantor set](https://en.wikipedia.org/wiki/Smith%E2%80%93Volterra%E2%80%93Cantor_set).*
{: .kv-cap}

This motivates using outer regularity, which gives us a bounding open set $$U$$ arbitrarily close in measure, i.e. for every $$\varepsilon > 0$$, there exists an open $$U \supseteq K$$ with $$\lambda(U) - \lambda(K) < \varepsilon$$.

![Inner regularity finds a compact K inside E; outer regularity finds an open U around K](/images/notes/analysis-prob/fig2_regularity.png){: .kv-fig}

*Fig 2. Two uses of inner / outer regularity*
{: .kv-cap}

In order for any to fit, there must exist some $$d > 0$$ such that $$K$$ and $$K + d$$ both lie in $$U$$. We may proceed with the separation lemma (since $$K$$ is compact and $$U^c$$ is closed), which not only guarantees $$d$$ exists, but also that for all $$\lvert d' \rvert < d$$, $$K + d'$$ also lies in $$U$$[^separation]. But we haven't covered this theorem yet, and we can arrive at the same conclusion with a more elementary method.

Let $$B(x, r) = (x - r, x + r)$$ be the open ball centered at $$x$$ with radius $$r$$. Consider two different open covers of $$K$$:

1. $$B(x_i, r_i)$$ for all $$x_i \in K$$
2. $$B(x_i, 2r_i)$$ for all $$x_i \in K$$

where each $$r_i$$ is chosen so that $$B(x_i, 2r_i) \subseteq U$$ (and hence also $$B(x_i, r_i) \subseteq U$$). Note that such an $$r_i > 0$$ exists for every $$x_i$$ because $$U$$ is open.

- Now, let's consider cover 1. From the definition of compactness, there must exist finitely many points $$x_{k_1}, x_{k_2}, \ldots, x_{k_n} \in K$$ such that $$\bigcup_{i=1}^{n} B(x_{k_i}, r_{k_i})$$ covers $$K$$[^indexing]. Then, since each open set in cover 2 is larger than the corresponding one in cover 1, we can use the balls centered at the same $$n$$ points as the finite cover for cover 2. Note that if we got our list of points from cover 2 and tried to use it on cover 1, it might not be sufficient.

- Consider any $$x \in K$$. Because it is covered by cover 1, it is distance at most $$r_{k_i}$$ from some $$x_{k_i}$$. Because ball $$k_i$$ is also in cover 2, $$B(x, r_{k_i}) \subseteq B(x_{k_i}, 2r_{k_i}) \subseteq U$$. Thus, we may move up to $$r_{k_i}$$ away from $$x$$ and still remain in $$U$$. If we let $$r = \min(r_{k_1}, r_{k_2}, \ldots, r_{k_n})$$, then $$r$$ is a "safe distance" for any $$x \in K$$: $$x + d \in U$$ for all $$\lvert d \rvert < r$$.

This feels strange, that it is regardless of our choice of $$U$$ — it can be arbitrarily tight as long as it contains $$K$$ - the point is there is always a gap still.

![Cover 1 (green) and cover 2 (blue) balls around points of K inside U, reduced to a finite subcover](/images/notes/analysis-prob/fig3_covers.png){: .kv-fig}

*Fig 3. Construction using compactness*
{: .kv-cap}

To conclude, using this safe distance, we can fit $$K + h, K + 2h, \ldots, K + (N-1)h$$ by choosing $$h$$ appropriately small as a function of $$N$$. In particular, let $$r = \inf\{\lvert x - y \rvert : x \in K,\ y \in U^c\} > 0$$ by the separation lemma, or $$r = \min(r_{k_1}, r_{k_2}, \ldots, r_{k_n})$$ by the argument above. Then, for any choice of $$N$$, letting $$h = r / N$$ ensures all $$N$$ sets fit in $$U$$.

Now we can formalize our thought from fig 1:

$$
\begin{aligned}
\lambda\big(K \cap (K + h)\big) &\ge 2\lambda(K) - \lambda(U) \\
\lambda\big(K \cap (K + h) \cap (K + 2h)\big) &\ge 2\lambda(K) - \lambda(U) + \lambda(K) - \lambda(U) = 3\lambda(K) - 2\lambda(U) \\
&\;\;\vdots \\
\lambda\Big(\bigcap_{j=0}^{N-1} (K + jh)\Big) &\ge N\lambda(K) - (N-1)\lambda(U)
\end{aligned}
$$

[^translation] [^direct]

We can let $$\lambda(U) < \frac{N}{N-1}\lambda(K)$$. Then, $$\lambda\big(\bigcap_{j=0}^{N-1} (K + jh)\big) > 0$$ so the intersection is non-empty, thus a length-$$N$$ arithmetic sequence exists. In fact, our construction not only guarantees existence of a single arithmetic sequence. For each of the uncountably many $$h \in \left(0, \frac{r}{N-1}\right)$$, there are uncountably many arithmetic sequences with this $$h$$[^uncountable]! It can also easily be extended to $$\mathbb{R}^n$$, other patterns of sequences, and infinite measure[^infinite].

In summary, for any $$N$$, we can find a sufficiently tight $$U$$ so that $$N$$ sets each measuring $$\lambda(K)$$ must have non-empty intersection, then we can ensure all of them fit by choosing a sufficiently small $$h$$ based on $$N$$ and the safe distance between $$K$$ and $$U^c$$. This is essentially an extension of the construction in the proof of [Steinhaus's theorem](https://en.wikipedia.org/wiki/Steinhaus_theorem). Analysis is more than just rigor and formalism, there are actually some really cool and new ways of thinking that lie in wait!


[^regularity]: Outer regularity follows from the definition of the outer measure used to construct the Lebesgue measure: the infimum of $$\sum_i \lvert I_i \rvert$$ over all countable covers of the set by open intervals $$I_i$$. Then to get inner regularity, restrict to a sufficiently large bounded interval and apply outer regularity to the complement.

[^compact]: In $$\mathbb{R}$$, compact is equivalent to closed and bounded. Thus its complement is open and lies in the Borel $$\sigma$$-algebra. Since the Lebesgue $$\sigma$$-algebra is a completion of the Borel $$\sigma$$-algebra, $$K$$ is Lebesgue measurable.

[^intersection]: Here, $$x \in \bigcap_{j=0}^{N-1} (K + jh)$$ means $$x, x - h, \ldots, x - (N-1)h \in K$$, but we can reverse it to turn decreasing to increasing and vice versa, so the sign here is not overly important.

[^separation]: In particular, it states that if $$K$$ is compact, $$F$$ is closed, and $$K \cap F = \emptyset$$, then $$\operatorname{dist}(K, F) = \inf\{\lvert x - y \rvert : x \in K,\ y \in F\} > 0$$. We take $$F = U^c$$ and the conclusion follows.

[^indexing]: Note here that we are not literally indexing points, since $$K$$ may be uncountable. It's just convenient notation to denote *some member*.

[^translation]: Since Lebesgue measure is translation invariant, i.e. $$\lambda(K + x) = \lambda(K)$$, all $$N$$ sets have the same measure $$\lambda(K)$$.

[^direct]: We can also avoid induction. Alternatively, let $$A_j = K + jh \subseteq U$$ for $$j = 0, \ldots, N-1$$. By De Morgan's law (with complements taken in $$U$$), $$U \setminus \bigcap_j A_j = \bigcup_j (U \setminus A_j)$$, so by subadditivity and translation invariance, $$\lambda\big(U \setminus \bigcap_j A_j\big) \le \sum_{j=0}^{N-1} \lambda(U \setminus A_j) = N\big(\lambda(U) - \lambda(K)\big)$$. Since $$\lambda(U)$$ is finite, $$\lambda\big(\bigcap_j A_j\big) = \lambda(U) - \lambda\big(U \setminus \bigcap_j A_j\big) \ge N\lambda(K) - (N-1)\lambda(U)$$.

[^uncountable]: Since countable sets have Lebesgue measure zero

[^infinite]: If $$E$$ has infinite measure, we can just write $$E = \bigcup_{M=1}^{\infty} (E \cap [-M, M])$$ and work inside one piece.

