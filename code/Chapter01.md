---
layout: code
title: Chapter01
permalink: /code/Chapter01.html
---
## Chapter 1

### Inspect the generated code for the power function.

#### For integers:

```julia
@code_native 3^2
```

#### And for floats

```julia
@code_native 3.5^2
```

