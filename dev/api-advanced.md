---
url: https://pysr.ai/dev/api-advanced.md
---
# Internal Reference

## Julia Interface

### init_julia

`init_julia(*args, **kwargs)`

*No documentation available.*

### install

`install(*args, **kwargs)`

*No documentation available.*

## Exporting to LaTeX

### sympy2latex

`sympy2latex(expr, prec=3, full_prec=True, **settings) -> 'str'`

Convert sympy expression to LaTeX with custom precision.

### sympy2latextable

`sympy2latextable(equations: 'pd.DataFrame', indices: 'list[int] | None' = None, precision: 'int' = 3, columns: 'list[str]' = ['equation', 'complexity', 'loss', 'score'], max_equation_length: 'int' = 50, output_variable_name: 'str' = 'y') -> 'str'`

Generate a booktabs-style LaTeX table for a single set of equations.

### sympy2multilatextable

`sympy2multilatextable(equations: 'list[pd.DataFrame]', indices: 'list[list[int]] | None' = None, precision: 'int' = 3, columns: 'list[str]' = ['equation', 'complexity', 'loss', 'score'], output_variable_names: 'list[str] | None' = None) -> 'str'`

Generate multiple latex tables for a list of equation sets.

### generate_table_environment

`generate_table_environment(columns: 'list[str]' = ['equation', 'complexity', 'loss']) -> 'tuple[str, str]'`

*No documentation available.*

## Exporting to JAX

### sympy2jax

`sympy2jax(expression, symbols_in, selection=None, extra_jax_mappings=None)`

Returns a function f and its parameters;

the function takes an input matrix, and a list of arguments:
f(X, parameters)
where the parameters appear in the JAX equation.

**Examples**

Let's create a function in SymPy:

```python
x, y = symbols('x y')
cosx = 1.0 * sympy.cos(x) + 3.2 * y
```

Let's get the JAX version. We pass the equation, and
the symbols required.

```python
f, params = sympy2jax(cosx, [x, y])
```

The order you supply the symbols is the same order
you should supply the features when calling
the function `f` (shape `[nrows, nfeatures]`).
In this case, features=2 for x and y.
The `params` in this case will be
`jnp.array([1.0, 3.2])`. You pass these parameters
when calling the function, which will let you change them
and take gradients.

Let's generate some JAX data to pass:

```python
key = random.PRNGKey(0)
X = random.normal(key, (10, 2))
```

We can call the function with:

```python
f(X, params)

#> DeviceArray([-2.6080756 ,  0.72633684, -6.7557726 , -0.2963162 ,
#                6.6014843 ,  5.032483  , -0.810931  ,  4.2520013 ,
#                3.5427954 , -2.7479894 ], dtype=float32)
```

We can take gradients with respect
to the parameters for each row with JAX
gradient parameters now:

```python
jac_f = jax.jacobian(f, argnums=1)
jac_f(X, params)

#> DeviceArray([[ 0.49364874, -0.9692889 ],
#               [ 0.8283714 , -0.0318858 ],
#               [-0.7447336 , -1.8784496 ],
#               [ 0.70755106, -0.3137085 ],
#               [ 0.944834  ,  1.767703  ],
#               [ 0.51673377,  1.4111717 ],
#               [ 0.87347716, -0.52637756],
#               [ 0.8760679 ,  1.0549792 ],
#               [ 0.9961824 ,  0.79581654],
#               [-0.88465923, -0.5822907 ]], dtype=float32)
```

We can also JIT-compile our function:

```python
compiled_f = jax.jit(f)
compiled_f(X, params)

#> DeviceArray([-2.6080756 ,  0.72633684, -6.7557726 , -0.2963162 ,
#                6.6014843 ,  5.032483  , -0.810931  ,  4.2520013 ,
#                3.5427954 , -2.7479894 ], dtype=float32)
```

### sympy2jaxtext

`sympy2jaxtext(expr, parameters, symbols_in, extra_jax_mappings=None)`

*No documentation available.*

## Exporting to PyTorch

### sympy2torch

`sympy2torch(expression, symbols_in, selection=None, extra_torch_mappings=None)`

Returns a module for a given sympy expression with trainable parameters;

This function will assume the input to the module is a matrix X, where
each column corresponds to each symbol you pass in `symbols_in`.
