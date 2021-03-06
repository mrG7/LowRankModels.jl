# LowRankModels.jl

<!--[![Build Status](https://travis-ci.org/madeleineudell/LowRankModels.jl.svg?branch=master)](https://travis-ci.org/madeleineudell/LowRankModels.jl)-->

LowRankModels.jl is a julia package for modeling and fitting generalized low rank models (GLRMs).
GLRMs model a data array by a low rank matrix, and
include many well known models in data analysis, such as 
principal components analysis (PCA), matrix completion, robust PCA,
nonnegative matrix factorization, k-means, and many more.

For more information on GLRMs, see [our paper][glrmpaper].

LowRankModels.jl makes it easy to mix and match loss functions and regularizers
to construct a model suitable for a particular data set.
In particular, it supports 

* using different loss functions for different columns of the data array, 
  which is useful when data types are heterogeneous 
  (eg, real, boolean, and ordinal columns);
* fitting the model to only *some* of the entries in the table, which is useful for data tables with many missing (unobserved) entries; and
* adding offsets and scalings to the model without destroying sparsity,
  which is useful when the data is poorly scaled.

## Installation

To install, just call
```
Pkg.add("LowRankModels")
```
at the julia prompt.

# Generalized Low Rank Models

GLRMs form a low rank model for tabular data `A` with `m` rows and `n` columns, 
which can be input as an array or any array-like object (for example, a data frame).
It is fine if only some of the entries have been observed 
(i.e., the others are missing or `NA`); the GLRM will only be fit on the observed entries `obs`.
The desired model is specified by choosing a rank `k` for the model,
an array of loss functions `losses`, and two regularizers, `rx` and `ry`.
The data is modeled as `XY`, where `X` is a `m`x`k` matrix and `Y` is a `k`x`n` matrix.
`X` and `Y` are found by solving the optimization problem
<!--``\mbox{minimize} \quad \sum_{(i,j) \in \Omega} L_{ij}(x_i y_j, A_{ij}) + \sum_{i=1}^m r_i(x_i) + \sum_{j=1}^n \tilde r_j(y_j)``-->

    minimize sum_{(i,j) in obs} losses[j](x[i,:] y[:,j], A[i,j]) + sum_i rx(x[i,:]) + sum_j ry(y[:,j])

The basic type used by LowRankModels.jl is the GLRM. To form a GLRM,
the user specifies

* the data `A` (any `AbstractArray`, such as an array, a sparse matrix, or a data frame)
* the array of loss functions `losses`
* the regularizers `rx` and `ry`
* the rank `k`

The user may also specify

* the observed entries `obs`
* starting matrices X₀ and Y₀

`obs` is a list of tuples of the indices of the observed entries in the matrix,
and may be omitted if all the entries in the matrix have been observed.
If `A` is a sparse matrix, implicit zeros are interpreted 
as missing entries by default; 
see the discussion of [sparse matrices](https://github.com/madeleineudell/LowRankModels.jl#fitting-sparse-matrices) below for more details.
`X₀` and `Y₀` are initialization
matrices that represent a starting guess for the optimization.

Losses and regularizers must be of type `Loss` and `Regularizer`, respectively,
and may be chosen from a list of supported losses and regularizers, which include

Losses:

* quadratic loss `QuadLoss`
* hinge loss `HingeLoss`
* logistic loss `LogLoss`
* weighted hinge loss `WeightedHinge`
* l1 loss `L1Loss`
* ordinal hinge loss `OrdinalHinge`
* PeriodicLoss loss `PeriodicLoss`

Regularizers:

* quadratic regularization `QuadReg`
* constrained squared euclidean norm `QuadConstraint`
* l1 regularization `OneReg`
* no regularization `ZeroReg`
* nonnegative constraint `NonNegConstraint` (eg, for nonnegative matrix factorization)
* 1-sparse constraint `OneSparseConstraint` (eg, for orthogonal NNMF)
* unit 1-sparse constraint `UnitOneSparseConstraint` (eg, for k-means)
* simplex constraint `SimplexConstraint`
* l1 regularization, combined with nonnegative constraint `NonNegOneReg`

Each of these losses and regularizers can be scaled 
(for example, to increase the importance of the loss relative to the regularizer) 
by calling `scale!(loss, newscale)`.
Users may also implement their own losses and regularizers,
or adjust internal parameters of the losses and regularizers; 
see [losses.jl](https://github.com/madeleineudell/LowRankModels.jl/blob/src/losses.jl) and [regularizers.jl](https://github.com/madeleineudell/LowRankModels.jl/blob/master/src/regularizers.jl) for more details.

## Example

For example, the following code forms a k-means model with `k=5` on the `100`x`100` matrix `A`:

    using LowRankModels
    m,n,k = 100,100,5
    losses = QuadLoss() # minimize squared distance to cluster centroids
    rx = UnitOneSparseConstraint() # each row is assigned to exactly one cluster
    ry = ZeroReg() # no regularization on the cluster centroids
    glrm = GLRM(A,losses,rx,ry,k)

To fit the model, call

	X,Y,ch = fit!(glrm)

which runs an alternating directions proximal gradient method on `glrm` to find the 
`X` and `Y` minimizing the objective function.
(`ch` gives the convergence history; see 
[Technical details](https://github.com/madeleineudell/LowRankModels.jl#technical-details) 
below for more information.)

The `losses` argument can also be an array of loss functions, 
with one for each column (in order). For example, 
for a data set with 3 columns, you could use 

    losses = [QuadLoss(), LogLoss(), HingeLoss()]

[More examples here.](https://github.com/madeleineudell/LowRankModels.jl/blob/master/examples/simple_glrms.jl)

# Missing data

If not all entries are present in your data table, just tell the GLRM
which observations to fit the model to by listing tuples of their indices in `obs`.
Then initialize the model using

    GLRM(A,losses,rx,ry,k, obs=obs)

If `A` is a DataFrame and you just want the model to ignore 
any entry that is of type `NA`, you can use

    obs = observations(A)

# Standard low rank models

Low rank models can easily be used to fit standard models such as PCA, k-means, and nonnegative matrix factorization. 
The following functions are available:

* `pca`: principal components analysis
* `qpca`: quadratically regularized principal components analysis
* `rpca`: robust principal components analysis
* `nnmf`: nonnegative matrix factorization
* `k-means`: k-means

See [the code](https://github.com/madeleineudell/LowRankModels.jl/blob/master/src/simple_glrms.jl) for usage.
Any keyword argument valid for a `GLRM` object, 
such as an initial value for `X` or `Y`
or a list of observations, 
can also be used with these standard low rank models.

# Scaling and offsets <a name="scaling"></a>

If you choose, LowRankModels.jl can add an offset to your model and scale the loss 
functions and regularizers so all columns have the same pull in the model.
Simply call 

    glrm = GLRM(A,losses,rx,ry,k, offset=true, scale=true)

This transformation generalizes standardization, a common proprocessing technique applied before PCA.
(For more about offsets and scaling, see the code or the paper.)

You can also add offsets and scalings to previously unscaled models:

* Add an offset to the model (by applying no regularization to the last row 
  of the matrix `Y`, and enforcing that the last column of `X` be all 1s) using

      add_offset!(glrm)

* Scale the loss functions and regularizers by calling

      equilibrate_variance!(glrm)

# Fitting DataFrames

Perhaps all this sounds like too much work. Perhaps you happen to have a 
[DataFrame](https://github.com/JuliaStats/DataFrames.jl) `df` lying around 
that you'd like a low rank (eg, `k=2`) model for. For example,

    import RDatasets
    df = RDatasets.dataset("psych", "msq")

Never fear! Just call

	glrm, labels = GLRM(df, k)
	X, Y, ch = fit!(glrm)

This will fit a GLRM with rank `k` to your data, 
using a QuadLoss loss for real valued columns,
HingeLoss loss for boolean columns, 
and ordinal HingeLoss loss for integer columns,
a small amount of QuadLoss regularization,
and scaling and adding an offset to the model as described [here](#scaling).
It returns the column labels for the columns it fit, along with the model.
Right now, all other data types are ignored, as are `NA`s.

The full call signature is
```
GLRM(df::DataFrame, k::Int;
              losses = None, rx = QuadReg(.01), ry = QuadReg(.01),
              offset = true, scale = true, NaNs_to_NAs = false)
```
You can modify the losses or regularizers, or turn off offsets or scaling,
using these keyword arguments.
If your DataFrame has `NaN`s in it, you can turn them into `NA`s that 
will be ignored in the fit by calling `NaNs_to_NAs!(df)`, or form your `GLRM` 
using `GLRM(df, k, NaNs_to_NAs = true)`.

To fit a data frame with categorical values, you can use the function
`expand_categoricals!` to turn categorical columns into a Boolean column for each 
level of the categorical variable. 
For example, `expand_categoricals!(df, [:gender])` will replace the gender 
column with a column corresponding to `gender=male`, 
a column corresponding to `gender=female`, and other columns corresponding to 
labels outside the gender binary, if they appear in the data set.

You can use the model to get some intuition for the data set. For example,
try plotting the columns of `Y` with the labels; you might see
that similar features are close to each other!

# Fitting Sparse Matrices

If you have a very large, sparsely observed dataset, then you may want to
encode your data as a
[sparse matrix](http://julia-demo.readthedocs.org/en/latest/stdlib/sparse.html).
By default, `LowRankModels` interprets the sparse entries of a sparse
matrix as missing entries (i.e. `NA` values). There is no need to
pass the indices of observed entries (`obs`) -- this is done
automatically when `GLRM(A::SparseMatrixCSC,...)` is called.
In addition, calling `fit!(glrm)` when `glrm.A` is a sparse matrix
will use the sparse variant of the proximal gradient descent algorithm,
`fit!(glrm, SparseProxGradParams(); kwargs...)`.

If, instead, you'd like to interpret the sparse entries as zeros, rather
than missing or `NA` entries, use:
```julia
glrm = GLRM(...;sparse_na=false)
```
In this case, the dataset is dense in terms of observations, but sparse
in terms of nonzero values. Thus, it may make more sense to fit the
model with the vanilla proximal gradient descent algorithm,
`fit!(glrm, ProxGradParams(); kwargs...)`.

# Technical details

## Optimization

The function `fit!` uses an alternating directions proximal gradient method
to minimize the objective. This method is *not* guaranteed to converge to 
the optimum, or even to a local minimum. If your code is not converging
or is converging to a model you dislike, there are a number of parameters you can tweak.

### Warm start

The algorithm starts with `glrm.X` and `glrm.Y` as the initial estimates
for `X` and `Y`. If these are not given explicitly, they will be initialized randomly.
If you have a good guess for a model, try setting them explicitly.
If you think that you're getting stuck in a local minimum, try reinitializing your
GLRM (so as to construct a new initial random point) and see if the model you obtain improves.

The function `fit!` sets the fields `glrm.X` and `glrm.Y`
after fitting the model. This is particularly useful if you want to use 
the model you generate as a warm start for further iterations.
If you prefer to preserve the original `glrm.X` and `glrm.Y` (eg, for cross validation),
you should call the function `fit`, which does not mutate its arguments.

You can even start with an easy-to-optimize loss function, run `fit!`,
change the loss function (`glrm.losses = newlosses`), 
and keep going from your warm start by calling `fit!` again to fit 
the new loss functions.

### Initialization

If you don't have a good guess at a warm start for your model, you might try
one of the initializations provided in `LowRankModels`.

* `init_svd!` initializes the model as the truncated SVD of the matrix of observed entries, with unobserved entries filled in with zeros. This initialization is known to result in provably good solutions for a number of "PCA-like" problems. See [our paper][glrmpaper] for details.
* `init_kmeanspp!` initializes the model using a modification of the [kmeans++](https://en.wikipedia.org/wiki/K-means_clustering) algorithm for data sets with missing entries; see [our paper][glrmpaper] for details. This works well for fitting clustering models, and may help in achieving better fits for nonnegative matrix factorization problems as well.
* `init_nndsvd!` initializes the model using a modification of the [NNDSVD](https://github.com/JuliaStats/NMF.jl/blob/master/src/initialization.jl#L18) algorithm as implemented by the [NMF](https://github.com/JuliaStats/NMF.jl) package. This modification handles data sets with missing entries by replacing missing entries with zeros. Optionally, by setting the argument `max_iters=n` with `n>0`, it will iteratively replace missing entries by their values as imputed by the NNDSVD, and call NNDSVD again on the new matrix. (This procedure is similar to the [soft impute](http://dl.acm.org/citation.cfm?id=1859931) method of Mazumder, Hastie and Tibshirani for matrix completion.)

### Parameters

As mentioned earlier, `LowRankModels` uses alternating proximal
gradient descent to derive estimates of `X` and `Y`. This can be done
by two slightly different procedures: (A) compute the full 
reconstruction, `X' * Y`, to compute the gradient and objective function;
(B) only compute the model estimate for entries of `A` that are observed.
The first method is likely preferred when there are few missing entries
for `A` because of hardware level optimizations
(e.g. chucking the operations so they just fit in various caches). The
second method is likely preferred when there are many missing entries of
`A`.

To fit with the first (dense) method:
```julia
fit!(glrm, ProxGradParams(); kwargs...)
```

To fit with the second (sparse) method:
```julia
fit!(glrm, SparseProxGradParams(); kwargs...)
```

The first method is used by default if `glrm.A` is a standard
matrix/array. The second method is used by default if `glrm.A` is a
`SparseMatrixCSC`.

`ProxGradParams()` and `SparseProxGradParams()` run these respective
methods with the default parameters:

* `stepsize`: The step size controls the speed of convergence.
Small step sizes will slow convergence, while large ones will cause 
divergence. `stepsize` should be of order 1;
`autoencode` scales it by the maximum number of entries per column or row
so that step *lengths* remain of order 1.
* `convergence_tol`: The algorithm stops when the decrease in the
objective per iteration is less than `convergence_tol*length(obs)`, 
* `max_iter`: The algorithm also stops if maximum number of rounds
`max_iter` has been reached.
* `min_stepsize`: The algorithm also stops if `stepsize` decreases below 
this limit.
* `inner_iter`: specifies how many proximal gradient steps to take on `X`
before moving on to `Y` (and vice versa).

The default parameters are: `ProxGradParams(stepsize=1.0;max_iter=100,inner_iter=1,convergence_tol=0.00001,min_stepsize=0.01*stepsize)` 

### Convergence
`ch` gives the convergence history so that the success of the optimization can be monitored;
`ch.objective` stores the objective values, and `ch.times` captures the times these objective values were achieved.
Try plotting this to see if you just need to increase `max_iter` to converge to a better model.

# Cross validation

A number of useful functions are available to help you check whether a given low rank model overfits to the test data set. 
These functions should help you choose adequate regularization for your model.

## Cross validation

* `cross_validate(glrm::GLRM, nfolds=5, params=Params(); verbose=false, use_folds=None, error_fn=objective, init=None)`: performs n-fold cross validation and returns average loss among all folds. More specifically, splits observations in `glrm` into `nfolds` groups, and builds new GLRMs, each with one group of observations left out. Fits each GLRM to the training set (the observations revealed to each GLRM) and returns the average loss on the test sets (the observations left out of each GLRM).

    **Optional arguments:**
    * `use_folds`: build `use_folds` new GLRMs instead of `n_folds` new GLRMs, each with `1/nfolds` of the entries left out. (`use_folds` defaults to `nfolds`.)
    * `error_fn`: use a custom error function to evaluate the fit, rather than the objective. For example, one might use the imputation error by setting `error_fn = error_metric`.
    * `init`: initialize the fit using a particular procedure. For example, consider `init=init_svd!`. See [Initialization](https://github.com/madeleineudell/LowRankModels.jl#initialization) for more options.

* `cv_by_iter(glrm::GLRM, holdout_proportion=.1, params=Params(1,1,.01,.01), niters=30; verbose=true)`: computes the test error and train error of the GLRM as it is trained. Splits the observations into a training set (`1-holdout_proportion` of the original observations) and a test set (`holdout_proportion` of the original observations). Performs `params.maxiter` iterations of the fitting algorithm on the training set `niters` times, and returns the test and train error as a function of iteration. 

## Regularization paths

* `regularization_path(glrm::GLRM; params=Params(), reg_params=logspace(2,-2,5), holdout_proportion=.1, verbose=true, ch::ConvergenceHistory=ConvergenceHistory("reg_path"))`: computes the train and test error for GLRMs varying the scaling of the regularization through any scaling factor in the array `reg_params`.

## Utilities

* `get_train_and_test(obs, m, n, holdout_proportion=.1)`: splits observations `obs` into a train and test set. `m` and `n` must be at least as large as the maximal value of the first or second elements of the tuples in `observations`, respectively. Returns `observed_features` and `observed_examples` for both train and test sets.

# Citing this package

If you use LowRankModels for published work, 
we encourage you to cite the software.

Use the following BibTeX citation:

    @article{udell2014,
        title = {Generalized Low Rank Models},
        author ={Udell, Madeleine and Horn, Corinne and Zadeh, Reza and Boyd, Stephen},
        year = {2014},
        archivePrefix = "arXiv",
        eprint = {1410.0342},
        primaryClass = "stat-ml",
        journal={arXiv preprint arXiv:1410.0342},
    }

[glrmpaper]: http://arxiv.org/abs/1410.0342
