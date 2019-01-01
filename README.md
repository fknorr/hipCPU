# hipCPU - Implementation of the AMD HIP API and programming model for CPUs

hipCPU is a header-only implementation of (a subset of) AMD's HIP API for CPUs. Parallelization is presently implemented with OpenMP.
For now, the main use case of the hipCPU project is to allow for easier debugging of HIP applications on CPUs instead of GPUs. Additionally, it also allows for a straight-forward implementation of host-fallbacks for HIP applications. In fact, the main motivation behind the hipCPU project is to serve as host fallback for the [hipSYCL](https://github.com/illuhad/hipSYCL) SYCL implementation.
As such, while performance will certainly be a priority later on, it is not of prime concern for the current priorities of the project. The current parallelization scheme will likely be inefficient for most existing HIP applications, but this is an ongoing effort and may improve in the future.

Possible use cases for the hipCPU project include
* Debugging of HIP applications on CPUs
* Development of HIP applications on systems without HIP-capable GPU (e.g. many notebooks)
* Host fallbacks
* hipCPU allows arbitrary functions to be executed in HIP streams. Since each stream comes with its own worker thread, the hipCPU stream API can hence also be used as some sort of thread pool. If the executed functions do not need to obey the HIP execution model (threads/blocks etc) this can be quite efficient. Use hipCPU's `hipLaunchTask()` for such situations (see section 'Extensions' for more information).

## Installation and usage
As a header-only library, no special steps must be taken to install hipCPU. Just put the include files into your project's include directory and include `hip_runtime.h` as with regular HIP.
You can then compile your HIP code with a regular, OpenMP-capable C++ compiler. OpenMP is required, so make sure to enable it with the appropriate compiler flag (usually `-fopenmp`)

## Limitations
* CUDA-style kernel launch syntax with `<<< >>>` is unsupported since this requires special compiler support. Use the traditional `hipLaunchKernel` and `hipLaunchKernelGGL` macros instead.
* Dynamic shared memory is supported, but must be requested with e.g.:
  ```cpp
  int* shared_mem = (int*)HIP_DYNAMIC_SHARED_MEMORY;
  ```
  instead of the usual
  ```cpp
  extern __shared__ int shared_mem[];
  ```
  This is because the latter requires explicit compiler support. Static shared memory can be used as usual:
  ```cpp
  __shared__ int shared_mem[SIZE];
  ```
* Dynamic shared memory can only be declared in function scope
* Anything related to explicit context/module management is unsupported
* Textures/surfaces are unimplemented at the moment
* Many device intrinsics are unimplemented at the moment
* A couple of other minor things are unimplemented; for now the implementation effort has focused on the most common functions and tools: kernel launches, streams, events, memory management.
* At the moment, hipCPU cannot coexist together with regular HIP at runtime; i.e. you have to decide at compile-time which HIP you want to compile against. This restriction will probably be lifted at least to some extent in the future.

## Extensions

There are a couple of things available in hipCPU that are unavailable in regular SYCL:
* hipCPU defines the `__HIPCPU__` macro. This can be used by your code to check if regular HIP or hipCPU is used.
* `hipLaunchTask(function, stream, args...)` can be used to execute `function(args...)` in the given stream as single, unparallelized task. This does not follow the HIP kernel execution model anymore, and anything related to it is unavailable: `hipThreadIdx_*`, `hipBlockDim_*`, shared memory etc. `hipLaunchTask` is particularly useful if you want to use HIP streams as some sort of thread pool or want to have your own OpenMP parallelization scheme in your task.
* `hipMallocManaged()` is supported in analogy to `cudaMallocManaged()`

## Implementation
At the moment, hipCPU executes the blocks one at a time, with one OpenMP thread per HIP thread. This is necessary to ensure correct synchronization semantics within a block (i.e. `__syncthreads()`). And yes, this is highly inefficient for programs following HIP's fine-grained parallelism model for various reasons (false sharing, thread management overhead, ...).
This inefficiency can be largely circumvented if you put a lot of work in one HIP thread, but this is usually not the way existing HIP programs for GPUs are structured.

`hipThreadIdx_x` etc are macros which access the static runtime object and return the thread id based on the OpenMP thread id and the block id based on the currently processed block. Since these macros must be well-defined globally, only a single kernel is executed at a time, even if you have several kernel launches in different non-blocking streams. This is not the case for the `hipLaunchTask` extension since `hipLaunchTask` does not need to guarantee correctness of `hipThreadIdx_x` etc.

Streams are implemented as a worker thread that processes a FIFO queue.


## Example
The following code adds two vectors. It is valid HIP code that can be executed both on the GPU and on the CPU with hipCPU:
```cpp
#include <hip/hip_runtime.h>
#include <vector>
#include <cassert>
#include <stdexcept>
#include <string>
#include <iostream>

void check_error(hipError_t err)
{
  if(err != hipSuccess)
    throw std::runtime_error{"Caught hip error: "+std::to_string(static_cast<int>(err))};
}

template<class T>
__global__ void vector_add(const T* in_a, const T* in_b, T* out, int num_elements)
{
  int gid = hipBlockIdx_x * hipBlockDim_x + hipThreadIdx_x;
  
  if(gid < num_elements)
    out[gid] = in_a[gid] + in_b[gid];
}
  
constexpr std::size_t buff_size = 1024;
constexpr std::size_t block_size = 16;

static_assert(buff_size % block_size == 0, 
              "buffer size must be a multiple of the block size");
  
int main()
{
  std::vector<int> in_a(buff_size);
  std::vector<int> in_b(buff_size);
  std::vector<int> result(buff_size);
  
  for(std::size_t i = 0; i < buff_size; ++i)
  {
    in_a[i] = static_cast<int>(i);
    in_b[i] = static_cast<int>(i+1);
  }
  
  int *d_in_a, *d_in_b, *d_result;
  check_error(hipMalloc((void**)&d_in_a, buff_size * sizeof(int)));
  check_error(hipMalloc((void**)&d_in_b, buff_size * sizeof(int)));
  check_error(hipMalloc((void**)&d_result, buff_size * sizeof(int)));
  
  check_error(hipMemcpy(d_in_a, in_a.data(), buff_size * sizeof(int), hipMemcpyHostToDevice));
  check_error(hipMemcpy(d_in_b, in_b.data(), buff_size * sizeof(int), hipMemcpyHostToDevice));
  
  hipLaunchKernelGGL(vector_add, buff_size/block_size, block_size, 0, 0, 
                     d_in_a, d_in_b, d_result, buff_size);
  
  check_error(hipMemcpy(result.data(), d_result, buff_size * sizeof(int), hipMemcpyDeviceToHost));
  
  for(std::size_t i = 0; i < result.size(); ++i)
    assert(result[i] == (in_a[i] + in_b[i]));
  
}
```


