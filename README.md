# Fused unbalanced Gromov-Wasserstein for Python

This package implements multiple GPU-compatible PyTorch solvers
to the Fused Unbalanced Gromov-Wasserstein optimization problem.

**This package is under active development. There is no guarantee that the API and solvers
won't change in the near future.**

## Introduction

### Optimization problem

In short, this code computes a matrix $P$ that matches points of two distributions $s$ and $t$.
$P$ is referred to as the _transport plan_.

We denote as $n$ and $m$ the number of points (we will also call them vertices or voxels)
of $s$ and $t$ respectively.

Two points $i$ and $j$, from $s$ and $t$ respectively, are matched based on similarity
between their respective features $F_i^s$ and $F_j^t$ (Wasserstein loss).
We fuse this loss with one that tries to preserve the respective underlying geometries of these two
distributions. We represent these geometries as kernel matrices $D^s$ and $D^t$ in the figure below.
In essence, two points which were distant from one another in the first distribution
should be matched to points which are distant from one another
in the second distribution (Gromov-Wasserstein loss).
Finally, the unbalancing part of this problem allows to leave points of the first and/or
the second distribution for which no good match can be found.

Details about the implementation and motivations are available in the original
[NeurIPS 2022 paper presenting this work](https://arxiv.org/abs/2206.09398) [(Thual, Tran et al. 2022) [1]](#1),
in which we align cortical structures of human individuals using this method.
In this case, we match areas of the cortex based on similarity of their functional activity
(ie how they behave throughout a series of experiments) while trying to preserve the anatomy of the cortex.

![Introduction to FUGW](assets/fugw_intro.png)

### Solvers

Wasserstein (W) problems are convex. This class of problems is well understood,
and a multitude of solvers exist to compute or approximate good solutions.
For instance, Sinkhorn solvers, which were first applied to W problems
in [(Cuturi 2013) [2]](#2), have proved to be very efficient.

Unfortunately, Gromov-Wasserstein (GW) problems are non-convex, consequently FUGW problems are non-convex too.
To circumvent this issue, authors in [(Séjourné et al. 2021) [3]](#3) derive a lower-bound of GW
by reformulating it as a bi-convex problem.
In short, instead of looking for a transport plan $P$ minimizing GW losses such as
$\sum_{i,j,k,l} |D^s_{i,j} - D^t_{k,l}| P_{i,k} P_{j,l}$,
one can look for 2 transport plans $P$ and $Q$ minimizing
$\sum_{i,j,k,l} |D^s_{i,j} - D^t_{k,l}| P_{i,k} Q_{j,l}$.
Note that if one imposes that $P=Q$, the two problems are equivalent.
In their paper, authors relax this constraint and only impose that
the mass of each plan is equal (ie $\text{m}(P) = \text{m}(Q)$) and the problem is now
convex in $P$ and in $Q$. Finally, they derive a block-coordinate-descent
(BCD) algorithm in which they alternatively freeze the value of $P$ (resp. $Q$)
while running a convex-problem solver (in their case it's a sinkhorn algorithm)
to optimize $Q$ (resp. $P$).

In this work, we adapt the previous approach to approximate solutions
to FUGW losses. Moreover, we provide multiple solvers to run inside the BCD algorithm.
Namely, we provide:

* `sinkhorn`: the classical Sinkhorn procedure described in [(Cuturi 2013) [2]](#2)
* `mm`: a majorize-minimization algorithm described in [(Chapel et al. 2021) [4]](#4)
* `ibpp`: an inexact-bregman-proximal-point algorithm described in [(Xie et al. 2020) [5]](#5)

## Installation

In a dedicated Python env, run:

```bash
pip install fugw
```

If you need to call functions within `fugw.scripts`, you should also run

```bash
pip install "fugw[scripts]"
```

### Install from source

```bash
git clone https://github.com/alexisthual/fugw.git
cd fugw
```

In a dedicated Python env, run:

```bash
pip install -e .
```

Development tests run on CPU and GPU, depending on the configuration of your machine.
To run them, install development dependencies with:

```bash
pip install -e ".[dev]"
```

```bash
pytest
```

## Usage and examples

Check examples given in `./examples` to see tested scripts
tackling real-cases.
Moreover, functions implemented in `./tests` yield some more examples
illustrating how to use this package.

### 1 - Transporting distributions containing less than 10k points

This is the simplest use case, in which $D^s$ and $D^t$ fit in memory.
We use `fugw.FUGW` to compute dense solutions to our problem.

**See [`./examples/1_dense.py`](./examples/1_dense.py)**

### 2 - Transporting distributions containing more than 10k points

Because FUGW computes a matrix $P$ of shape $n \times m$,
the size of $P$ grows quadratically with the number of vertices,
and rapidly won't fit on GPUs.

To circumvent this issue, we pre-implemented the following
coarse-to-fine approach:

1. compute a coarse FUGW solution between sub-samples
of the source and target distributions
which fit on GPUs
2. select good pairs of matched vertices $i$, $j$
according to the transport plan computed at the coarse step
3. define a sparsity mask such all pairs of vertices $k$, $l$
within a neighbour of $i$ and $j$ respectively
apprear in the mask. The neighbourhood of vertices is
defined according to a given radius, for instance vertices $k$
which are within a 5mm-range of $i$.
4. compute a fine-scale FUGW solution using a sparse solver
initialised with a transport plan using previous sparsity mask

#### Generate embeddings from mesh

Similarly to $P$, $D^s$ and $D^t$ can't be stored on GPUs.
Since, in our applications, $D^s$ are low-rank kernel matrices
or can be approximated as such, we derive an embedding $X^s$ of size `(n, k)`
such that $D^s_{i,j} = ||X^s_{i} - X^s_{j}||^2_{2}$
and store $X^s$ instead of $D^s$.

We leverage an approach described in [(Platt 2005) [6]](#6)
to derive an LMDS embedding approximating the geodesic distance on a given mesh.

We use `fugw.scripts.lmds` to compute these embeddings.

**See [`./examples/2_1_lmds.py`](./examples/2_1_lmds.py)**

#### Deriving a high-rank sparse solution

We then apply the coarse-to-fine approach described above to compute a plan.
In particular, we use `fugw.scripts.coarse_to_fine` to handle the whole routine.
This routine uses `fugw.FUGWSparse` to derive sparse solutions to our optimisation problem.

**See [`./examples/2_2_coarse_to_fine.py`](./examples/2_2_coarse_to_fine.py)**

### 3 - Computing a FUGW barycenter from multiple distributions

Currently implemented for dense solvers only.

See `./tests/test_barycenter.py`.

## API references

This repo contains 2 main classes, `fugw.FUGW` and `fugw.FUGWSparse`.
As a rule of thumb, they are respectively suited for problems with less or more than 10k.
Each of these classes implement `.fit()`, `.transform()`
and `inverse_transform()` methods.
`fugw.FUGW` and `fugw.FUGWSparse` both come with implementations for
`"sinkhorn"`, `"mm"` and `"ibpp"`.

### Class `fugw.FUGW` parameters, **for 10k points or less**

In short, this class is suited to compute transport plans between distributions
with 10k points or less.
The size of the transport plan grows quadratically with the size of the
source and target distributions:
in particular, if the source and target distributions have 10k points each,
the transport plan will be a dense tensor with `1e8` and ~500Mo.
While solving a FUGW problem, we store several matrices with such a size
(about 10, among which $P$, $Q$, $D^s$, $D^t$, $(D^s)^2$, $(D^t)^2$ and a cost matrix),
which should help you assess the maximum number of source and target points
you can use to fit such model.

* `alpha`: value in ]0, 1[, controls the relative importance of the Wasserstein and the Gromov-Wasserstein losses in the FUGW loss (see equation)
* `rho`: value in ]0, +inf[, controls the relative importance of the marginal constraints. High values force the mass of each point to be transported ; low values allow for some mass loss
* `eps`: value in ]0, +inf[, controls the relative importance of the entropy loss
* `reg_mode`: `"joint"` or `"independent"` ; controls how marginals of $P$ and $Q$ are penalized in the marginal constraints (for instance, either $\text{KL}(P_{\verb|#1|} \otimes Q_{\verb|#1|}, w^s \otimes w^s)$ or $\text{KL}(P_{\verb|#1|}, w^s) + \text{KL}(Q_{\verb|#1|}, w^s)$ for the marginal constraints relative to the source)

### Method `fugw.FUGW.fit()` parameters

* `source_features`: array of size `(d, n)`, $(F^s)^T$, features of each source vertex. In neuroscience applications, these could be the activations / deactivations
of a given vertex across stimuli
* `target_features`: array of size `(d, m)`, $(F^t)^T$
* `source_geometry`: array of size `(n, n)`, $D^s$, kernel matrix in the GW loss.
In neuroscience applications, $D^s_{i,j}$ would be the geodesic distance
between vertices $i$ and $j$ on the cortical sheet
* `target_geometry`: array of size `(m, m)`, $D^t$
* `source_weights`: array of size `(n)`, $w^s$, weight (or mass) of each source vertex.
Usually, in neuroscience applications, this is a uniform vector
* `target_weights`: array of size `(m)`, $w^t$, weight (or mass) of each target vertex
* `init_plan`: array of size `(n, m)`
* `init_duals`: tuple of arrays of size `(n)` and `(m)` respectively
* `uot_solver`: `"sinkhorn"` or `"mm"` or `"ibpp"`
* `nits_bcd`: number of BCD iterations to run
* `nits_uot`: number of solver iterations to run within each BCD iteration
* `tol_bcd`: Stop the BCD procedure early if the absolute difference between two consecutive transport plans under this threshold
* `tol_uot`: Stop the BCD procedure early if the absolute difference between two consecutive transport plans under this threshold
* `early_stopping_threshold`: Stop the BCD procedure early if the FUGW loss falls under this threshold
* `eval_bcd`: During .fit(), at every eval_bcd step: 1. compute the FUGW loss and store it in an array 2. consider stopping early
* `eval_uot`: During .fit(), at every eval_uot step: 1. consider stopping early
* `ibpp_eps_base`: Regularization parameter specific to the ibpp solver
* `ibpp_nits_sinkhorn`: Number of sinkhorn iterations to run within each uot iteration of the ibpp solver
* `device`: `torch.device` on which computation should happen
* `verbose`: boolean, log training information

### Method `fugw.FUGW.transform()` parameters

* `source_features`: array of size `(d, n)`, source features to project on target
* `device`: `torch.device` on which computation should happen

### Method `fugw.FUGW.inverse_transform()` parameters

* `target_features`: array of size `(d, m)`, target features to project on source
* `device`: `torch.device` on which computation should happen

### Class `fugw.FUGWSparse` parameters, **for 10k points or more**

In short, this class is suited to compute transport plans between distributions
with more than 10k points.
For smaller distributions, `fugw.FUGW` will be much faster.

Available methods and parameters for these methods are the same as for `fugw.FUGW`,
except for `.fit()`.

### Method `fugw.FUGWSparse.fit()` parameters

**Parameters are the same as for `fugw.FUGW.fit()` except that `source_geometry`
(resp. `target_geometry`) is replaced by `source_geometry_embedding`
(resp. `source_geometry`).** Moreover, `init_plan` is required
and takes a sparse torch tensor.

* `source_geometry_embedding`: array of size `(n, k)`, $X^s$,
embedding used to store $D^s$ for high values of `n`
* `target_geometry_embedding`: array of size `(m, k)`, $X^t$,
embedding used to store $D^t$ for high values of `m`
* `init_plan`: sparse torch.Tensor of size `(n, m)`, either COO or CSR, initialization of the transport plan with a sparse matrix whose sparsity mask will be that of the final solution

### Method `fugw.scripts.coarse_to_fine.fit()` parameters

* `coarse_model`: fugw.FUGW, Coarse model to fit
* `coarse_model_fit_params`: dict, Parameters to give to the `.fit()` method of the coarse model
* `coarse_pairs_selection_method`: "topk" or "quantile", Method used to select pairs of source and target features whose neighbourhoods will be used to define the sparsity mask of the solution
* `source_selection_radius`: float, Radius used to determine the neighbourhood of source vertices when defining sparsity mask for fine-scale solution
* `target_selection_radius`: float, Radius used to determine the neighbourhood of target vertices when defining sparsity mask for fine-scale solution
* `fine_model`: fugw.FUGWSparse, Fine-scale model to fit
* `fine_model_fit_params`: dict, Parameters to give to the `.fit()` method of the fine-scale model
* `source_sample_size`: int, Number of vertices to sample from source for coarse step
* `target_sample_size`: int, Number of vertices to sample from target for coarse step
* `source_features`: ndarray(n_features, n), Feature maps for source subject.  n_features is the number of contrast maps, it should be the same for source and target data.  n is the number of nodes on the source graph, it can be different from m, the number of nodes on the target graph.
* `target_features`: ndarray(n_features, m), Feature maps for target subject.
* `source_geometry_embeddings`: array(n, k), Embedding approximating the distance matrix between source vertices
* `target_geometry_embeddings`: array(m, k), Embedding approximating the distance matrix between target vertices
* `source_weights`: ndarray(n) or None, Distribution weights of source nodes.  Should sum to 1. If None, eahc node's weight will be set to 1 / n.
* `target_weights`: ndarray(n) or None, Distribution weights of target nodes.  Should sum to 1. If None, eahc node's weight will be set to 1 / m.
* `device`: "auto" or torch.device, if "auto": use first available gpu if it's available, cpu otherwise.
* `verbose`: bool, optional, defaults to False, Log solving process.

### Method `fugw.scripts.lmds.compute_lmds()` parameters

* `coordinates`: array of size (n, 3), Coordinates of vertices
* `triangles`: array of size (t, 3), Triplets of indices indicating faces
* `n_landmarks`: Number of vertices to sample on mesh to approximate embedding
* `k`: Dimension of embedding
* `n_jobs`: Number of CPUs to use to parallelise computation
* `verbose`: Log solving process

## References

<a id="1">[1]</a> Thual, Alexis, Huy Tran, Tatiana Zemskova, Nicolas Courty, Rémi Flamary, Stanislas Dehaene, and Bertrand Thirion. ‘Aligning Individual Brains with Fused Unbalanced Gromov-Wasserstein’. arXiv, 19 June 2022. <https://doi.org/10.48550/arXiv.2206.09398>.

<a id="2">[2]</a> Chizat, Lenaic, Gabriel Peyré, Bernhard Schmitzer, and François-Xavier Vialard. ‘Scaling Algorithms for Unbalanced Transport Problems’. arXiv, 22 May 2017. <https://arxiv.org/abs/1607.05816>.

<a id="3">[3]</a> Sejourne, Thibault, Francois-Xavier Vialard, and Gabriel Peyré. ‘The Unbalanced Gromov Wasserstein Distance: Conic Formulation and Relaxation’. In Advances in Neural Information Processing Systems, 34:8766–79. Curran Associates, Inc., 2021. <https://proceedings.neurips.cc/paper/2021/hash/4990974d150d0de5e6e15a1454fe6b0f-Abstract.html>.

<a id="4">[4]</a> Chapel, Laetitia, Rémi Flamary, Haoran Wu, Cédric Févotte, and Gilles Gasso. ‘Unbalanced Optimal Transport through Non-Negative Penalized Linear Regression’. In Advances in Neural Information Processing Systems, 34:23270–82. Curran Associates, Inc., 2021. <https://proceedings.neurips.cc/paper/2021/hash/c3c617a9b80b3ae1ebd868b0017cc349-Abstract.html>.

<a id="5">[5]</a> Xie, Yujia, Xiangfeng Wang, Ruijia Wang, and Hongyuan Zha. ‘A Fast Proximal Point Method for Computing Exact Wasserstein Distance’. In Proceedings of The 35th Uncertainty in Artificial Intelligence Conference, 433–53. PMLR, 2020. <https://proceedings.mlr.press/v115/xie20b.html>.

<a id="6">[6]</a> Platt, John. ‘FastMap, MetricMap, and Landmark MDS Are All Nystrom Algorithms’, 1 January 2005. https://www.microsoft.com/en-us/research/publication/fastmap-metricmap-and-landmark-mds-are-all-nystrom-algorithms/.

## Citing this work

If this package was useful to you, please cite it in your work:

```bibtex
@article{Thual-2022-fugw,
  title={Aligning individual brains with Fused Unbalanced Gromov-Wasserstein},
  author={Thual, Alexis and Tran, Huy and Zemskova, Tatiana and Courty, Nicolas and Flamary, Rémi and Dehaene, Stanislas and Thirion, Bertrand},
  publisher={arXiv},
  doi={10.48550/ARXIV.2206.09398},
  url={https://arxiv.org/abs/2206.09398},
  year={2022},
  copyright={Creative Commons Attribution 4.0 International}
}
```
