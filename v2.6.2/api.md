---
url: https://pysr.ai/v2.6.2/api.md
---
# API Reference

`PySRRegressor` has many options for controlling a symbolic regression search.
Let's look at them below.

### The Algorithm

#### Creating the Search Space

* **`binary_operators`**

  List of strings for binary operators used in the search.
  See the [operators page](https://pysr.ai/operators)
  for more details.

  *Default:* `["+", "-", "*", "/"]`

* **`unary_operators`**

  Operators which only take a single scalar as input.
  For example, `"cos"` or `"exp"`.

  *Default:* `None`

* **`operators`**

  Generic operators by arity (number of arguments). Keys are integers
  representing arity, values are lists of operator strings.
  Example: `{1: ["sin", "cos"], 2: ["+", "-", "*"], 3: ["muladd"]}`.
  Cannot be used with `binary_operators` or `unary_operators`.

  *Default:* `None`

* **`expression_spec`**

  The type of expression to search for. By default,
  this is just `ExpressionSpec()`. You can also use
  `TemplateExpressionSpec(...)` which allows you to specify
  a custom template for the expressions.

  *Default:* `ExpressionSpec()`

* **`type_spec`**

  Declarative custom value type. PySR builds an isolated Julia wrapper,
  interface methods, operators, and loss from this specification.

  *Default:* `None`

* **`maxsize`**

  Max complexity of an equation.

  *Default:* `30`

* **`maxdepth`**

  Max depth of an equation. You can use both `maxsize` and
  `maxdepth`. `maxdepth` is by default not used.

  *Default:* `None`

#### Setting the Search Size

* **`niterations`**

  Number of iterations of the algorithm to run. The best
  equations are printed and migrate between populations at the
  end of each iteration.

  *Default:* `100`

* **`populations`**

  Number of populations running.

  *Default:* `31`

* **`population_size`**

  Number of individuals in each population.

  *Default:* `27`

* **`ncycles_per_iteration`**

  Number of total mutations to run, per 10 samples of the
  population, per iteration.

  *Default:* `380`

#### The Objective

* **`elementwise_loss`**

  String of Julia code specifying an elementwise loss function.
  Can either be a loss from LossFunctions.jl, or your own loss
  written as a function. Examples of custom written losses include:
  `myloss(x, y) = abs(x-y)` for non-weighted, or
  `myloss(x, y, w) = w*abs(x-y)` for weighted.
  The included losses include:
  Regression: `LPDistLoss{P}()`, `L1DistLoss()`,
  `L2DistLoss()` (mean square), `LogitDistLoss()`,
  `HuberLoss(d)`, `L1EpsilonInsLoss(ϵ)`, `L2EpsilonInsLoss(ϵ)`,
  `PeriodicLoss(c)`, `QuantileLoss(τ)`.
  Classification: `ZeroOneLoss()`, `PerceptronLoss()`,
  `L1HingeLoss()`, `SmoothedL1HingeLoss(γ)`,
  `ModifiedHuberLoss()`, `L2MarginLoss()`, `ExpLoss()`,
  `SigmoidLoss()`, `DWDMarginLoss(q)`.

  *Default:* `"L2DistLoss()"`

* **`loss_function`**

  Alternatively, you can specify the full objective function as
  a snippet of Julia code, including any sort of custom evaluation
  (including symbolic manipulations beforehand), and any sort
  of loss function or regularizations. The default `loss_function`
  used in SymbolicRegression.jl is roughly equal to:

  ```julia
  function eval_loss(tree, dataset::Dataset{T,L}, options)::L where {T,L}
      prediction, flag = eval_tree_array(tree, dataset.X, options)
      if !flag
          return L(Inf)
      end
      return sum((prediction .- dataset.y) .^ 2) / dataset.n
  end
  ```

  where the example elementwise loss is mean-squared error.
  You may pass a function with the same arguments as this (note
  that the name of the function doesn't matter). Here,
  both `prediction` and `dataset.y` are 1D arrays of length `dataset.n`.

  *Default:* `None`

* **`loss_function_expression`**

  Similar to `loss_function`, but takes as input the full
  expression object as the first argument, rather than
  the innermost `AbstractExpressionNode`. This is useful
  for specifying custom loss functions on `TemplateExpressionSpec`.

  *Default:* `None`

* **`loss_scale`**

  Determines how loss values are scaled when computing scores.
  "log" (default) uses logarithmic scaling of loss ratios; this mode
  requires non-negative loss values and is ideal for traditional
  loss functions that are always non-negative.
  "linear" uses direct
  differences between losses; this mode handles any loss values
  (including negative) and is useful for custom loss functions,
  especially those based on likelihoods.

* **`model_selection`**

  Model selection criterion when selecting a final expression from
  the list of best expression at each complexity.
  Can be `'accuracy'`, `'best'`, or `'score'`.
  `'accuracy'` selects the candidate model with the lowest loss
  (highest accuracy).
  `'score'` selects the candidate model with the highest score.
  Score is defined as the negated derivative of the log-loss with
  respect to complexity - if an expression has a much better
  loss at a slightly higher complexity, it is preferred.
  `'best'` selects the candidate model with the highest score
  among expressions with a loss better than at least 1.5x the
  most accurate model.

  *Default:* `'best'`

* **`dimensional_constraint_penalty`**

  Additive penalty for if dimensional analysis of an expression fails.
  By default, this is `1000.0`.

* **`dimensionless_constants_only`**

  Whether to only search for dimensionless constants, if using units.

  *Default:* `False`

#### Working with Complexities

* **`parsimony`**

  Multiplicative factor for how much to punish complexity.

  *Default:* `0.0`

* **`constraints`**

  Dictionary of int (unary) or tuples (multi-arity), this enforces
  maxsize constraints on the individual arguments of operators.
  E.g., `'pow': (-1, 1)` says that power laws can have any
  complexity left argument, but only 1 complexity in the right
  argument. For arity-3 operators like muladd, use 3-tuples like
  `'muladd': (-1, -1, 1)` to constrain each argument's complexity.
  Use this to force more interpretable solutions.

  *Default:* `None`

* **`nested_constraints`**

  Specifies how many times a combination of operators can be
  nested. For example, `{"sin": {"cos": 0}}, "cos": {"cos": 2}}`
  specifies that `cos` may never appear within a `sin`, but `sin`
  can be nested with itself an unlimited number of times. The
  second term specifies that `cos` can be nested up to 2 times
  within a `cos`, so that `cos(cos(cos(x)))` is allowed
  (as well as any combination of `+` or `-` within it), but
  `cos(cos(cos(cos(x))))` is not allowed. When an operator is not
  specified, it is assumed that it can be nested an unlimited
  number of times. This requires that there is no operator which
  is used both in the unary operators and the binary operators
  (e.g., `-` could be both subtract, and negation). For binary
  operators, you only need to provide a single number: both
  arguments are treated the same way, and the max of each
  argument is constrained.

  *Default:* `None`

* **`complexity_of_operators`**

  If you would like to use a complexity other than 1 for an
  operator, specify the complexity here. For example,
  `{"sin": 2, "+": 1}` would give a complexity of 2 for each use
  of the `sin` operator, and a complexity of 1 for each use of
  the `+` operator (which is the default). You may specify real
  numbers for a complexity, and the total complexity of a tree
  will be rounded to the nearest integer after computing.

  *Default:* `None`

* **`complexity_of_constants`**

  Complexity of constants.

  *Default:* `1`

* **`complexity_of_variables`**

  Global complexity of variables. To set different complexities for
  different variables, pass a list of complexities to the `fit` method
  with keyword `complexity_of_variables`. You cannot use both.

  *Default:* `1`

* **`complexity_mapping`**

  Alternatively, you can pass a function (a string of Julia code) that
  takes the expression as input and returns the complexity. Make sure that
  this operates on `AbstractExpression` (and unpacks to `AbstractExpressionNode`),
  and returns an integer.

  *Default:* `None`

* **`warmup_maxsize_by`**

  Whether to slowly increase max size from a small number up to
  the maxsize (if greater than 0).  If greater than 0, says the
  fraction of training time at which the current maxsize will
  reach the user-passed maxsize.

  *Default:* `0.0`

* **`use_frequency`**

  Whether to measure the frequency of complexities, and use that
  instead of parsimony to explore equation space. Will naturally
  find equations of all complexities.

  *Default:* `True`

* **`use_frequency_in_tournament`**

  Whether to use the frequency mentioned above in the tournament,
  rather than just the simulated annealing.

  *Default:* `True`

* **`adaptive_parsimony_scaling`**

  If the adaptive parsimony strategy (`use_frequency` and
  `use_frequency_in_tournament`), this is how much to (exponentially)
  weight the contribution. If you find that the search is only optimizing
  the most complex expressions while the simpler expressions remain stagnant,
  you should increase this value.

  *Default:* `1040.0`

* **`should_simplify`**

  Whether to use algebraic simplification in the search. Note that only
  a few simple rules are implemented.

  *Default:* `True`

#### Mutations

* **`default_mutations`**

  Default mutation configurations and their weights. When provided, these
  replace the SymbolicRegression.jl defaults and plugin-contributed mutations.

  *Default:* `None`

* **`mutations`**

  Mutation configurations and their weights. Entries override or extend
  `default_mutations` by mutation type.

  *Default:* `None`

* **`default_plugins`**

  Default plugin configurations.

  *Default:* `None`

* **`plugins`**

  Plugin configurations. Entries override or extend `default_plugins` by
  plugin type.

  *Default:* `None`

* **`weight_add_node`**

  Relative likelihood for mutation to add a node.

  *Default:* `None` (mapping to `2.47`

* **`weight_insert_node`**

  Relative likelihood for mutation to insert a node.

  *Default:* `None` (mapping to `0.0112`

* **`weight_delete_node`**

  Relative likelihood for mutation to delete a node.

  *Default:* `None` (mapping to `0.870`

* **`weight_do_nothing`**

  Relative likelihood for mutation to leave the individual.

  *Default:* `None` (mapping to `0.273`

* **`weight_mutate_constant`**

  Relative likelihood for mutation to change the constant slightly
  in a random direction.

  *Default:* `None` (mapping to `0.0346`

* **`weight_mutate_operator`**

  Relative likelihood for mutation to swap an operator.

  *Default:* `None` (mapping to `0.293`

* **`weight_mutate_feature`**

  Relative likelihood for mutation to change which feature a variable node references.

  *Default:* `None` (mapping to `0.1`

* **`weight_swap_operands`**

  Relative likehood for swapping operands in binary operators.

  *Default:* `None` (mapping to `0.198`

* **`weight_rotate_tree`**

  How often to perform a tree rotation at a random node.

  *Default:* `None` (mapping to `4.26`

* **`weight_randomize`**

  Relative likelihood for mutation to completely delete and then
  randomly generate the equation

  *Default:* `None` (mapping to `0.000502`

* **`weight_simplify`**

  Relative likelihood for mutation to simplify constant parts by evaluation

  *Default:* `None` (mapping to `0.00209`

* **`weight_optimize`**

  Constant optimization can also be performed as a mutation, in addition to
  the normal strategy controlled by `optimize_probability` which happens
  every iteration. Using it as a mutation is useful if you want to use
  a large `ncycles_periteration`, and may not optimize very often.

  *Default:* `None` (mapping to `0.0`

* **`weight_backsolve`**

  Relative likelihood for backsolve mutation. To configure its parameters,
  pass `BacksolveMutation(...)` through `mutations` instead.

  *Default:* `None` (mapping to `0.0`

* **`crossover_probability`**

  Absolute probability of crossover-type genetic operation, instead of a mutation.

  *Default:* `0.2`

* **`annealing`**

  Whether to use annealing.

  *Default:* `True`

* **`alpha`**

  Initial temperature for simulated annealing
  (requires `annealing` to be `True`).

  *Default:* `3.17`

* **`perturbation_factor`**

  Constants are perturbed by a max factor of
  (perturbation_factor\*T + 1). Either multiplied by this or
  divided by this.

  *Default:* `0.129`

* **`probability_negate_constant`**

  Probability of negating a constant in the equation when mutating it.

  *Default:* `0.00743`

* **`skip_mutation_failures`**

  Whether to skip mutation and crossover failures, rather than
  simply re-sampling the current member.

  *Default:* `True`

#### Tournament Selection

* **`tournament_selection_n`**

  Number of expressions to consider in each tournament.

  *Default:* `15`

* **`tournament_selection_p`**

  Probability of selecting the best expression in each
  tournament. The probability will decay as p\*(1-p)^n for other
  expressions, sorted by loss.

  *Default:* `0.982`

#### Constant Optimization

* **`optimizer_algorithm`**

  Optimization scheme to use for optimizing constants. Can currently
  be `NelderMead` or `BFGS`.

  *Default:* `"BFGS"`

* **`optimizer_nrestarts`**

  Number of time to restart the constants optimization process with
  different initial conditions.

  *Default:* `2`

* **`optimizer_f_calls_limit`**

  How many function calls to allow during optimization.

  *Default:* `10_000`

* **`optimize_probability`**

  Probability of optimizing the constants during a single iteration of
  the evolutionary algorithm.

  *Default:* `0.14`

* **`optimizer_iterations`**

  Number of iterations that the constants optimizer can take.

  *Default:* `8`

* **`should_optimize_constants`**

  Whether to numerically optimize constants (Nelder-Mead/Newton)
  at the end of each iteration.

  *Default:* `True`

#### Migration between Populations

* **`fraction_replaced`**

  How much of population to replace with migrating equations from
  other populations.

  *Default:* `0.00036`

* **`fraction_replaced_hof`**

  How much of population to replace with migrating equations from
  hall of fame.

  *Default:* `0.0614`

* **`fraction_replaced_guesses`**

  How much of the population to replace with migrating equations from
  guesses.

  *Default:* `0.001`

* **`migration`**

  Whether to migrate.

  *Default:* `True`

* **`hof_migration`**

  Whether to have the hall of fame migrate.

  *Default:* `True`

* **`topn`**

  How many top individuals migrate from each population.

  *Default:* `12`

### Data Preprocessing

* **`denoise`**

  Whether to use a Gaussian Process to denoise the data before
  inputting to PySR. Can help PySR fit noisy data.

  *Default:* `False`

* **`select_k_features`**

  Whether to run feature selection in Python using random forests,
  before passing to the symbolic regression code. None means no
  feature selection; an int means select that many features.

  *Default:* `None`

### Stopping Criteria

* **`max_evals`**

  Limits the total number of evaluations of expressions to
  this number.

  *Default:* `None`

* **`timeout_in_seconds`**

  Make the search return early once this many seconds have passed.

  *Default:* `None`

* **`early_stop_condition`**

  Stop the search early if this loss is reached. You may also
  pass a string containing a Julia function which
  takes a loss and complexity as input, for example:
  `"f(loss, complexity) = (loss < 0.1) && (complexity < 10)"`.

  *Default:* `None`

### Performance and Parallelization

* **`parallelism`**

  Parallelism to use for the search. Can be `"serial"`, `"multithreading"`, or `"multiprocessing"`.

  *Default:* `"multithreading"`

* **`procs`**

  Number of processes to use for parallelism. If `None`, defaults to `cpu_count()`.

  *Default:* `None`

* **`cluster_manager`**

  For distributed computing, this sets the job queue system. Set to
  "slurm" to use an existing Slurm allocation, with `procs` equal to
  its task count. Other supported values are "pbs", "lsf", "sge",
  "qrsh", "scyld", and "htc".

  *Default:* `None`

* **`heap_size_hint_in_bytes`**

  For multiprocessing, this sets the `--heap-size-hint` parameter
  for new Julia processes. This can be configured when using
  multi-node distributed compute, to give a hint to each process
  about how much memory they can use before aggressive garbage
  collection.

* **`worker_timeout`**

  Timeout in seconds for worker processes during multiprocessing to respond.
  If a worker does not respond within this time, it will be restarted.

  *Default:* `None`

* **`worker_imports`**

  Module names to import on multiprocessing workers. For example,
  `["MyPackage", "OtherPackage"]` runs
  `using MyPackage, OtherPackage` on every worker. TypeSpec preambles,
  operators, objectives, and expression specifications may depend on
  packages listed here; arbitrary bindings from `Main` are unavailable.

  *Default:* `None`

* **`batching`**

  Whether to compare population members on small batches during
  evolution. Still uses full dataset for comparing against hall
  of fame. `"auto"` lets SymbolicRegression.jl choose based on the
  dataset.

  *Default:* `"auto"`

* **`batch_size`**

  The batch size to use if batching. If None, uses the
  full dataset when N<=1000, 128 for N<5000, 256 for N<50000,
  or 512 for N≥50000.

  *Default:* `None`

* **`precision`**

  What precision to use for the data. By default this is `32`
  (float32), but you can select `64` or `16` as well, giving
  you 64 or 16 bits of floating point precision, respectively.
  If you pass complex data, the corresponding complex precision
  will be used (i.e., `64` for complex128, `32` for complex64).

  *Default:* `32`

* **`fast_cycle`**

  Batch over population subsamples. This is a slightly different
  algorithm than regularized evolution, but does cycles 15%
  faster. May be algorithmically less efficient.

  *Default:* `False`

* **`turbo`**

  (Experimental) Whether to use LoopVectorization.jl to speed up the
  search evaluation. Certain operators may not be supported.
  Does not support 16-bit precision floats.

  *Default:* `False`

* **`bumper`**

  (Experimental) Whether to use Bumper.jl to speed up the search
  evaluation. Does not support 16-bit precision floats.

  *Default:* `False`

* **`autodiff_backend`**

  Which backend to use for automatic differentiation during constant
  optimization. Currently `"Zygote"`, `"Mooncake"`, and `"Enzyme"` are supported.
  The default, `None`, uses forward-mode or finite difference.

  *Default:* `None`

### Determinism

* **`random_state`**

  Pass an int for reproducible results across multiple function calls.
  See :term:`Glossary <random_state>`.

  *Default:* `None`

* **`deterministic`**

  Make a PySR search give the same result every run.
  To use this, you must turn off parallelism
  (with `parallelism="serial"`),
  and set `random_state` to a fixed seed.

  *Default:* `False`

* **`warm_start`**

  Tells fit to continue from where the last call to fit finished.
  If false, each call to fit will be fresh, overwriting previous results.
  Plugin runtime state is reinitialized for each call to `fit`.

  *Default:* `False`

* **`guesses`**

  Initial guesses for expressions to seed the search. Examples:
  `["x0 + x1", "x0^2"]`, `[["x0"], ["x1"]]` (multi-output),
  or `[{"f": "#1 + #2", "p": [5.0, 10.0]}]` for a
  `TemplateExpressionSpec` with component `f` and parameter vector `p`.
  Here `#1` and `#2` are the first and second arguments of `f`.
  Parameter values use the data precision or the model's `TypeSpec`, and
  supplied vectors must match the lengths declared by the template.
  Parameter names may be omitted to retain their normal initialization.

  *Default:* `None`

### Monitoring

* **`verbosity`**

  What verbosity level to use. 0 means minimal print statements.

  *Default:* `1`

* **`update_verbosity`**

  What verbosity level to use for package updates.
  Will take value of `verbosity` if not given.

  *Default:* `None`

* **`print_precision`**

  How many significant digits to print for floats.

  *Default:* `5`

* **`progress`**

  Whether to use a progress bar instead of printing to stdout.

  *Default:* `True`

* **`logger_spec`**

  Logger specification for the Julia backend. See, for example,
  `TensorBoardLoggerSpec`.

  *Default:* `None`

* **`input_stream`**

  The stream to read user input from, used for the `'q'` + `<enter>`
  command that stops a search early. `None` reads from `"stdin"` when
  attached to an interactive terminal and otherwise disables stdin
  watching via `"devnull"` (for example in Jupyter, where typed input
  never reaches the search). You can also pass `"stdin"` or `"devnull"`
  explicitly, or reference an arbitrary Julia object in the `Main`
  namespace.

  *Default:* `None`

* **`use_tracing`**

  Whether to write a JSONL trace of the search to `tracing_file`.
  One record per line holds the live members of one population at one
  iteration, along with every mutation, crossover, tuning and death
  event. Uses JSON.jl, which is installed into the Julia environment
  on first use.

  *Default:* `False`

* **`tracing_file`**

  Where to write the trace when `use_tracing` is set. Any existing
  file is overwritten when the search starts.

  *Default:* `"pysr_trace.jsonl"`

### Environment

* **`temp_equation_file`**

  Whether to put the hall of fame file in the temp directory.
  Deletion is then controlled with the `delete_tempfiles`
  parameter.

  *Default:* `False`

* **`tempdir`**

  directory for the temporary files.

  *Default:* `None`

* **`delete_tempfiles`**

  Whether to delete the temporary files after finishing.

  *Default:* `True`

* **`update`**

  Whether to automatically update Julia packages when `fit` is called.
  You should make sure that PySR is up-to-date itself first, as
  the packaged Julia packages may not necessarily include all
  updated dependencies.

  *Default:* `False`

### Exporting the Results

* **`output_directory`**

  The base directory to save output files to. Files
  will be saved in a subdirectory according to the run ID.
  Will be set to `outputs/` if not provided.

  *Default:* `None`

* **`run_id`**

  A unique identifier for the run. Will be generated using the
  current date and time if not provided.

  *Default:* `None`

* **`output_jax_format`**

  Whether to create a 'jax_format' column in the output,
  containing jax-callable functions and the default parameters in
  a jax array.

  *Default:* `False`

* **`output_torch_format`**

  Whether to create a 'torch_format' column in the output,
  containing a torch module with trainable parameters.

  *Default:* `False`

* **`extra_sympy_mappings`**

  Provides mappings between custom `binary_operators` or
  `unary_operators` defined in julia strings, to those same
  operators defined in sympy.
  E.G if `unary_operators=["inv(x)=1/x"]`, then for the fitted
  model to be export to sympy, `extra_sympy_mappings`
  would be `{"inv": lambda x: 1/x}`.

  *Default:* `None`

* **`extra_torch_mappings`**

  The same as `extra_jax_mappings` but for model export
  to pytorch. Note that the dictionary keys should be callable
  pytorch expressions.
  For example: `extra_torch_mappings={sympy.sin: torch.sin}`.

  *Default:* `None`

* **`extra_jax_mappings`**

  Similar to `extra_sympy_mappings` but for model export
  to jax. The dictionary maps sympy functions to jax functions.
  For example: `extra_jax_mappings={sympy.sin: "jnp.sin"}` maps
  the `sympy.sin` function to the equivalent jax expression `jnp.sin`.

  *Default:* `None`

## PySRRegressor Functions

### fit

`fit(self, X, y, *, Xresampled=None, weights=None, variable_names: 'ArrayLike[str] | None' = None, complexity_of_variables: 'int | float | list[int | float] | None' = None, X_units: 'ArrayLike[str] | None' = None, y_units: 'str | ArrayLike[str] | None' = None) -> "'PySRRegressor'"`

Search for equations to fit the dataset and store them in `self.equations_`.

**Parameters**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| `X` | `ndarray \| pandas.DataFrame` | Training data of shape (n_samples, n_features). | *required* |
| `y` | `ndarray \| pandas.DataFrame` | Target values of shape (n_samples,) or (n_samples, n_targets). Will be cast to X's dtype if necessary. | *required* |
| `Xresampled` | `ndarray \| pandas.DataFrame` | Resampled training data, of shape (n_resampled, n_features), to generate a denoised data on. This will be used as the training data, rather than `X`. | `None` |
| `weights` | `ndarray \| pandas.DataFrame` | Weight array of the same shape as `y`. Each element is how to weight the mean-square-error loss for that particular element of `y`. Alternatively, if a custom `loss` was set, it will can be used in arbitrary ways. | `None` |
| `variable_names` | `list[str]` | A list of names for the variables, rather than "x0", "x1", etc. If `X` is a pandas dataframe, the column names will be used instead of `variable_names`. Cannot contain spaces or special characters. Avoid variable names which are also function names in `sympy`, such as "N". | `None` |
| `X_units` | `list[str]` | A list of units for each variable in `X`. Each unit should be a string representing a Julia expression. See DynamicQuantities.jl https://ai.damtp.cam.ac.uk/dynamicquantities/dev/units for more information. | `None` |
| `y_units` | `str \| list[str]` | Similar to `X_units`, but as a unit for the target variable, `y`. If `y` is a matrix, a list of units should be passed. If `X_units` is given but `y_units` is not, then `y_units` will be arbitrary. | `None` |

**Returns**

* **Type:** `object`
* Fitted estimator.

### predict

`predict(self, X, index: 'int | list[int] | None' = None) -> 'ndarray'`

Predict y from input X using the equation chosen by `model_selection`.

You may see what equation is used by printing this object. X should
have the same columns as the training data.

**Parameters**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| `X` | `ndarray \| pandas.DataFrame` | Training data of shape `(n_samples, n_features)`. | *required* |
| `index` | `int \| list[int]` | If you want to compute the output of an expression using a particular row of `self.equations_`, you may specify the index here. For multiple output equations, you must pass a list of indices in the same order. | `None` |

**Returns**

* **Type:** `ndarray of shape (n_samples, nout_)`
* Values predicted by substituting `X` into the fitted symbolic
  regression model.

**Raises**

* **`ValueError`**: Raises if the `best_equation` cannot be evaluated.

### from_file

`from_file(equation_file: 'None' = None, *, run_directory: 'PathLike', binary_operators: 'list[str] | None' = None, unary_operators: 'list[str] | None' = None, operators: 'dict[int, list[str]] | None' = None, n_features_in: 'int | None' = None, feature_names_in: 'ArrayLike[str] | None' = None, selection_mask: 'NDArray[np.bool_] | None' = None, nout: 'int' = 1, **pysr_kwargs) -> "'PySRRegressor'"`

Create a model from a saved model checkpoint or equation file.

**Parameters**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| `run_directory` | `str` | The directory containing outputs from a previous run. This is of the form `[output_directory]/[run_id]`. | *required* |
| `binary_operators` | `list[str]` | The same binary operators used when creating the model. Not needed if loading from a pickle file. | `None` |
| `unary_operators` | `list[str]` | The same unary operators used when creating the model. Not needed if loading from a pickle file. | `None` |
| `operators` | `dict[int, list[str]]` | Operator mapping by arity used when creating the model. Provide this if the original run relied on the generic `operators` parameter. Not needed if loading from a pickle file. | `None` |
| `n_features_in` | `int` | Number of features passed to the model. Not needed if loading from a pickle file. | `None` |
| `feature_names_in` | `list[str]` | Names of the features passed to the model. Not needed if loading from a pickle file. | `None` |
| `selection_mask` | `NDArray[np.bool_]` | If using `select_k_features`, you must pass `model.selection_mask_` here. Not needed if loading from a pickle file. | `None` |
| `nout` | `int` | Number of outputs of the model. Not needed if loading from a pickle file. | `1` |
| `**pysr_kwargs` | `dict` | Any other keyword arguments to initialize the PySRRegressor object. These will overwrite those stored in the pickle file. Not needed if loading from a pickle file. | *required* |

**Returns**

* **Type:** `PySRRegressor`
* The model with fitted equations.

### sympy

`sympy(self, index: 'int | list[int] | None' = None)`

Return sympy representation of the equation(s) chosen by `model_selection`.

**Parameters**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| `index` | `int \| list[int]` | If you wish to select a particular equation from `self.equations_`, give the index number here. This overrides the `model_selection` parameter. If there are multiple output features, then pass a list of indices with the order the same as the output feature. | `None` |

**Returns**

* **Type:** `str, list[str] of length nout_`
* SymPy representation of the best equation.

### latex

`latex(self, index: 'int | list[int] | None' = None, precision: 'int' = 3) -> 'str | list[str]'`

Return latex representation of the equation(s) chosen by `model_selection`.

**Parameters**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| `index` | `int \| list[int]` | If you wish to select a particular equation from `self.equations_`, give the index number here. This overrides the `model_selection` parameter. If there are multiple output features, then pass a list of indices with the order the same as the output feature. | `None` |
| `precision` | `int` | The number of significant figures shown in the LaTeX representation. | `3` |

**Returns**

* **Type:** `str or list[str] of length nout_`
* LaTeX expression of the best equation.

### pytorch

`pytorch(self, index=None)`

Return pytorch representation of the equation(s) chosen by `model_selection`.

Each equation (multiple given if there are multiple outputs) is a PyTorch module
containing the parameters as trainable attributes. You can use the module like
any other PyTorch module: `module(X)`, where `X` is a tensor with the same
column ordering as trained with.

**Parameters**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| `index` | `int \| list[int]` | If you wish to select a particular equation from `self.equations_`, give the index number here. This overrides the `model_selection` parameter. If there are multiple output features, then pass a list of indices with the order the same as the output feature. | `None` |

**Returns**

* **Type:** `torch.nn.Module`
* PyTorch module representing the expression.

### jax

`jax(self, index=None)`

Return jax representation of the equation(s) chosen by `model_selection`.

Each equation (multiple given if there are multiple outputs) is a dictionary
containing {"callable": func, "parameters": params}. To call `func`, pass
func(X, params). This function is differentiable using `jax.grad`.

**Parameters**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| `index` | `int \| list[int]` | If you wish to select a particular equation from `self.equations_`, give the index number here. This overrides the `model_selection` parameter. If there are multiple output features, then pass a list of indices with the order the same as the output feature. | `None` |

**Returns**

* **Type:** `dict[str, Any]`
* Dictionary of callable jax function in "callable" key,
  and jax array of parameters as "parameters" key.

### latex_table

`latex_table(self, indices: 'list[int] | None' = None, precision: 'int' = 3, columns: 'list[str]' = ['equation', 'complexity', 'loss', 'score']) -> 'str'`

Create a LaTeX/booktabs table for all, or some, of the equations.

**Parameters**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| `indices` | `list[int] \| list[list[int]]` | If you wish to select a particular subset of equations from `self.equations_`, give the row numbers here. By default, all equations will be used. If there are multiple output features, then pass a list of lists. | `None` |
| `precision` | `int` | The number of significant figures shown in the LaTeX representations. | `3` |
| `columns` | `list[str]` | Which columns to include in the table. | `['equation', 'complexity', 'loss', 'score']` |

**Returns**

* **Type:** `str`
* A string that will render a table in LaTeX of the equations.

### refresh

`refresh(self, run_directory: 'PathLike | None' = None) -> 'None'`

Update self.equations\_ with any new options passed.

For example, updating `extra_sympy_mappings`
will require a `.refresh()` to update the equations.

**Parameters**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| `checkpoint_file` | `str or Path` | Path to checkpoint hall of fame file to be loaded. The default will use the set `equation_file_`. | *required* |

## Expression Specifications

### ExpressionSpec

`ExpressionSpec()`

The default expression specification, with no special behavior.

### TemplateExpressionSpec

`TemplateExpressionSpec(combine: 'str', expressions: 'list[str]', variable_names: 'list[str]', parameters: 'dict[str, int] | None' = None) -> None`

Spec for templated expressions.

This class allows you to specify how multiple sub-expressions should be combined
in a structured way, with constraints on which variables each sub-expression can use.
Pass this to PySRRegressor with the `expression_spec` argument.

**Parameters**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| `combine` | `str` | Julia function string that defines how the sub-expressions are combined. For example: "sin(f(x1, x2)) + g(x3)^2" would constrain f to use x1,x2 and g to use x3. | *required* |
| `expressions` | `list[str]` | List of symbols representing the inner expressions (e.g., \["f", "g"]). These will be used as keys in the template structure. | *required* |
| `variable_names` | `list[str]` | List of variable names that will be used in the combine function. | *required* |
| `parameters` | `dict[str, int]` | Dictionary mapping parameter names to their lengths. For example, {"p1": 2, "p2": 1} means p1 is a vector of length 2 and p2 is a vector of length 1. These parameters will be optimized during the search. | `None` |

### AbstractExpressionSpec

`AbstractExpressionSpec()`

Abstract base class describing expression types.

This basically just holds the options for the expression type,
as well as explains how to parse and evaluate them.

All expression types must implement:

1. julia_expression_spec(): The actual expression specification, returned as a Julia object.
   This will get passed as `expression_spec` in `SymbolicRegression.Options`.
2. create_exports(), which will be used to create the exports of the equations, such as
   the executable format, the SymPy format, etc.

Implementations may also declare the export formats they support.

## Type Specifications

### TypeSpec

`TypeSpec(name: 'str', fields: 'dict[str, str]', sample: 'str', scalar_constants: 'str | None' = None, with_scalar_constants: 'str | None' = None, init: 'str | None' = None, mutate: 'str | None' = None, is_valid: 'str | None' = None, string: 'str | None' = None, preamble: 'str | None' = None, definitions: 'str | None' = None, loss_type: 'str | None' = None, init_invalid: 'str | None' = None) -> None`

Definition of a custom value type for symbolic regression.

**Parameters**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| `name` | `str` | Name of the generated Julia type. Use this name in operator and hook definitions. | *required* |
| `fields` | `dict[str, str]` | Ordered mapping from field names to Julia field types. | *required* |
| `sample` | `str` | Julia callable with signature `rng -> (...)::{name}`. | *required* |
| `scalar_constants` | `str` | Julia callable with signature `value::{name} -> (...)::Vector{Float64}`. Provide together with `with_scalar_constants` to enable continuous constant optimization. | `None` |
| `with_scalar_constants` | `str` | Julia callable with signature `(value::{name}, scalar_constants::Vector{Float64}) -> (...)::{name}`. | `None` |
| `init` | `str` | Julia callable with signature `() -> value::{name}`. The default repurposes `sample` using a deterministic local random-number generator. | `None` |
| `mutate` | `str` | Julia callable with signature `(rng, value::{name}, temperature::Float64) -> value::{name}`. Use this to mutate a value of the type. The `temperature` (from simulated annealing) is between 0 and 1. High temperatures generally should be used for more aggressive mutations, and low temperatures for more conservative mutations. The default mutation repurposes `scalar_constants` and `with_scalar_constants` to mutate the set with SymbolicRegression.jl's default constant mutation, and put it back into a new value. | `None` |
| `is_valid` | `str` | Julia callable with signature `value::{name} -> (...)::Bool`. Use this to check if your type is in a valid state. For example, if your type has been processed in a way that invalidates it (like a NaN, but specific to your type), you may use this function to check for that. This will be used to quit evaluation early. The default checks that every scalar constant is finite, or accepts every value for a non-optimizable type. | `None` |
| `init_invalid` | `str` | Julia callable with signature `() -> value::{name}` that constructs an invalid value. The result must have this type and fail `is_valid`. Used by DynamicExpressions to fill failed convenience evaluations. Omit this hook for types with no invalid state. It receives no prototype or shape information. | `None` |
| `string` | `str` | Julia callable with signature `value::{name} -> (...)::AbstractString` used to print values. | `None` |
| `preamble` | `str` | Julia source evaluated once before the generated type definition. Types and functions it defines are visible to the hooks and to operator and objective sources; imports are not, so any source needing a package beyond SymbolicRegression must import it itself. | `None` |
| `definitions` | `str` | Julia source evaluated once directly after the generated type, for constructors and methods that mention it. | `None` |
| `loss_type` | `str` | Concrete Julia `AbstractFloat` type returned by a custom full objective. Elementwise loss return types are inferred. | `None` |

## Logger Specifications

### TensorBoardLoggerSpec

`TensorBoardLoggerSpec(log_dir: 'str' = 'logs/run', log_interval: 'int' = 1, overwrite: 'bool' = False) -> None`

Specification for TensorBoard logger.

**Parameters**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| `log_dir` | `str` | Directory where TensorBoard logs will be saved. If `overwrite` is `False`, new logs will be saved to `{log_dir}_1`, and so on. | `'logs/run'` |
| `log_interval` | `int` | Interval (in steps) at which logs are written. Default is 10. | `1` |
| `overwrite` | `bool` | Whether to overwrite existing logs in the directory. Default is False. | `False` |

### AbstractLoggerSpec

`AbstractLoggerSpec()`

Abstract base class for logger specifications.

## Mutations

### ConstantMutation

`ConstantMutation(perturbation_factor: 'float' = 0.086, probability_negate: 'float' = 0.01) -> None`

Perturb a constant.

SymbolicRegression.jl default weight: `0.0346`.

Defaults match SymbolicRegression.jl.

### OperatorMutation

`OperatorMutation() -> None`

Replace an operator with another operator of the same arity.

SymbolicRegression.jl default weight: `0.293`.

### FeatureMutation

`FeatureMutation() -> None`

Change the feature referenced by a variable node.

SymbolicRegression.jl default weight: `0.1`.

### SwapOperandsMutation

`SwapOperandsMutation() -> None`

Swap the operands of a binary operator.

SymbolicRegression.jl default weight: `0.198`.

### AddNodeMutation

`AddNodeMutation() -> None`

Append a node to the expression.

SymbolicRegression.jl default weight: `2.47`.

### InsertNodeMutation

`InsertNodeMutation() -> None`

Insert a node above an existing node.

SymbolicRegression.jl default weight: `0.0112`.

### DeleteNodeMutation

`DeleteNodeMutation() -> None`

Delete a node from the expression.

SymbolicRegression.jl default weight: `0.870`.

### RotateTreeMutation

`RotateTreeMutation() -> None`

Rotate a subtree.

SymbolicRegression.jl default weight: `4.26`.

### BacksolveMutation

`BacksolveMutation(max_library_size: 'int' = 500, max_terms: 'int' = 8, min_improvement: 'float' = 0.001, node_attempts: 'int' = 8) -> None`

Fit a replacement expression by backsolving through the expression.

SymbolicRegression.jl default weight: `0.0`.

### SimplifyMutation

`SimplifyMutation() -> None`

Simplify constant parts of the expression.

SymbolicRegression.jl default weight: `0.00209`.

### RandomizeMutation

`RandomizeMutation() -> None`

Replace the expression with a random expression.

SymbolicRegression.jl default weight: `0.000502`.

### OptimizeMutation

`OptimizeMutation() -> None`

Optimize constants as a mutation.

SymbolicRegression.jl default weight: `0.0`.

### DoNothingMutation

`DoNothingMutation() -> None`

Leave the expression unchanged.

SymbolicRegression.jl default weight: `0.273`.

### AbstractMutation

`AbstractMutation()`

Base class for mutation configurations.

## Plugins

### AdaptiveMutationWeightsPlugin

`AdaptiveMutationWeightsPlugin(smoothing: 'float' = 0.02, floor: 'float' = 0.05, reward: "Literal['cost', 'loss']" = 'cost') -> None`

Adapt mutation weights from their observed success rates.

Enabled by default in PySR. Pass an explicit `default_plugins` that
omits this plugin to disable it.
Learned runtime state is reinitialized for each call to `fit`.

### AdaptiveParsimonyPlugin

`AdaptiveParsimonyPlugin(tournament: 'bool' = True, mutation_acceptance: 'bool' = True) -> None`

Apply frequency-based parsimony during selection and mutation.

Enabled by default in PySR through `use_frequency=True` and
`use_frequency_in_tournament=True`.

### MutationBurstPlugin

`MutationBurstPlugin(retry_attempts: 'int' = 4, compound_probability: 'float' = 0.25, compound_max_steps: 'int' = 2) -> None`

Retry rejected mutations and optionally chain accepted mutations.

Disabled by default in PySR.

### SimulatedAnnealingPlugin

`SimulatedAnnealingPlugin(alpha: 'float' = 0.1) -> None`

Apply simulated annealing during mutation acceptance.

Enabled by default through `annealing=True`.

### AbstractPlugin

`AbstractPlugin()`

Base class for plugin configurations.
