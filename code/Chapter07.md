---
layout: code
title: Chapter07
permalink: /code/Chapter07.html
---
## Chapter 7

```julia
;nvidia-smi


using BenchmarkTools
using Pkg
Pkg.add("CUDAnative");
Pkg.add("CuArrays");
Pkg.add("CUDAdrv");
```

### CUDAnative

```julia
using CUDAnative
function cudaprint(n)
    @cuprintf("Thread %ld prints: %ld\n",
        threadIdx().x, n)
     return
 end

 @cuda threads=4 cudaprint(10)

 @device_code_ptx @cuda cudaprint(10)

 @device_code_sass @cuda cudaprint(10)

 @device_code_lowered @cuda cudaprint(10)
```

## CuArrays

```julia
 using CuArrays

 a=CuArray([1f0, 2f0, 3f0])

 b = a.^2 .- a.*2 .+ sqrt.(a)
```

### Monte Carlo simulations on the GPU

```julia
using CuArrays.CURAND

function pi_gpu(n)
   4 * sum(curand(Float64, n).^2 .+ curand(Float64, n).^2 .<= 1) / n
 end

 function pi_serial(n)
   inside = 0
   for i in 1:n
       x, y = rand(), rand()
       inside += (x^2 + y^2 <= 1)
   end
   return 4 * inside / n
end

@btime pi_serial(10_000_000)

@btime pi_gpu(10_000_000)

function pi_gpu32(n)
   4 * sum(curand(Float32, n).^2 .+ curand(Float32, n).^2 .<= 1) / n
end

@btime pi_gpu32(10_000_000)
```

### Writing your own kernels

```julia
function add_gpu!(y, x)
   index = threadIdx().x
   stride = blockDim().x
      for i = index:stride:length(y)
        @inbounds y[i] += x[i]
      end
    return nothing
end

N=1000_000
a = cufill(1.0f0, N)
b = cufill(2.0f0, N)
@cuda threads=256 add_gpu!(b, a)
```

### Measuring GPU Performance

```julia
a = CuArray{Float32}(undef, 1024);

@btime identity.($a);

@btime CuArrays.@sync(identity.($a));

CuArrays.@time pi_gpu(10_000_000)
```

### Performance Tips

#### Scalar Iteration

```julia
function addcol_scalar(a, b)
   s = size(a)
   for j = 1:s[2]
      for i = 1:s[1]
         @inbounds a[i,j] = b[i,j+1] + b[i,j]
      end
   end
 end

 function addcol_fast(a, b)
   s = size(a)
   for j = 1:s[2]
      @inbounds a[:,j] .= b[:,j+1] + b[:,j]
   end
end

a = ones(Float32, 10000, 99);
b = ones(Float32, 10000, 100);
@btime addcol_scalar($a, $b)

@btime addcol_scalar($(CuArray(a)), $(CuArray(b))) samples=1;

@btime addcol_fast($(CuArray(a)), $(CuArray(b)));

CuArrays.allowscalar(false)

addcol_scalar(CuArray(a), CuArray(b))  # Will Error!
```

#### Combine kernels

```julia
function addcol_faster(a, b)
   a .= @views b[:, 2:end] .+ b[:, 1:end-1]
end

@btime addcol_faster($a, $b);

@btime addcol_faster($(CuArray(a)), $(CuArray(b))) ;
```

#### Process more data

```julia
a = ones(Float32,1000_000, 99);

b = ones(Float32, 1000_000, 100);

@btime addcol_faster($a, $b);

@btime addcol_faster($(CuArray(a)), $(CuArray(b))) ;
```

### Deep learning on the GPU

```julia
using Pkg
Pkg.add("Flux")
CuArrays.allowscalar(true)

using Flux, Flux.Data.MNIST, Statistics
using Flux: onehotbatch, onecold, crossentropy, throttle
using Base.Iterators: repeated, partition

imgs = MNIST.images()

labels = onehotbatch(MNIST.labels(), 0:9)

# Partition into batches of size 32
train = [(cat(float.(imgs[i])..., dims = 4), labels[:,i])
 for i in partition(1:60_000, 32)]
# Prepare test set (first 1,000 images)
tX = cat(float.(MNIST.images(:test)[1:1000])..., dims = 4)
tY = onehotbatch(MNIST.labels(:test)[1:1000], 0:9)


m = Chain(
 Conv((3, 3), 1=>32, relu),
 Conv((3, 3), 32=>32, relu),
 x -> maxpool(x, (2,2)),
 Conv((3, 3), 32=>16, relu),
 x -> maxpool(x, (2,2)),
 Conv((3, 3), 16=>10, relu),
 x -> reshape(x, :, size(x, 4)),
 Dense(90, 10), softmax)

 loss(x, y) = crossentropy(m(x), y)
 accuracy(x, y) = mean(onecold(m(x)) .== onecold(y))
 opt = ADAM()
 @time Flux.train!(loss, Flux.params(m), train[1:10], opt)
 @show(accuracy(tX, tY))

gputrain = gpu.(train[1:10])
gpum = gpu(m)
gputX = gpu(tX)
gputY = gpu(tY)
gpuloss(x, y) = crossentropy(gpum(x), y)
gpuaccuracy(x, y) = mean(onecold(gpum(x)) .== onecold(y))
gpuopt = ADAM()
Flux.train!(gpuloss, Flux.params(gpum), gpu.(train[1:1]), opt)
@time Flux.train!(gpuloss, Flux.params(gpum), gputrain, opt)
```

### ArrayFire

```julia
Pkg.add("ArrayFire")
using ArrayFire

a=rand(1000,1000);

b=AFArray(a);

c=b*b;

d=Array(c);

@btime $b*$b;

@btime $a*$a;

@btime begin; sin($b); sync($b); end;

@btime sin.($a);
```

