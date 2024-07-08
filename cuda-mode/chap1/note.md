
# How to profile CUDA kernels in PyTorch


CUDA is async.

profile 方法

## CUDA profile torch.autograd.profiler

```python
with torch.autograd.profiler.profile(use_cuda=True) as prof:
    torch.square(b)
```

可以打印出 各种操作的用时：
```
=============
Profiling torch.square
=============
STAGE:2024-07-08 21:40:06 30357:30357 ActivityProfilerController.cpp:314] Completed Stage: Warm Up
STAGE:2024-07-08 21:40:06 30357:30357 ActivityProfilerController.cpp:320] Completed Stage: Collection
STAGE:2024-07-08 21:40:06 30357:30357 ActivityProfilerController.cpp:324] Completed Stage: Post Processing
-------------------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  
                     Name    Self CPU %      Self CPU   CPU total %     CPU total  CPU time avg     Self CUDA   Self CUDA %    CUDA total  CUDA time avg    # of Calls  
-------------------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  
             aten::square         0.60%      12.000us         4.12%      83.000us      83.000us       3.000us         0.15%       1.951ms       1.951ms             1  
                aten::pow         2.53%      51.000us         3.42%      69.000us      69.000us       1.946ms        99.74%       1.948ms       1.948ms             1   这里是使用的pow函数
        aten::result_type         0.10%       2.000us         0.10%       2.000us       2.000us       1.000us         0.05%       1.000us       1.000us             1  
                 aten::to         0.00%       0.000us         0.00%       0.000us       0.000us       1.000us         0.05%       1.000us       1.000us             1  
          cudaEventRecord         0.35%       7.000us         0.35%       7.000us       0.875us       0.000us         0.00%       0.000us       0.000us             8  
         cudaLaunchKernel         0.79%      16.000us         0.79%      16.000us      16.000us       0.000us         0.00%       0.000us       0.000us             1  
    cudaDeviceSynchronize        95.63%       1.928ms        95.63%       1.928ms       1.928ms       0.000us         0.00%       0.000us       0.000us             1  
-------------------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  
Self CPU time total: 2.016ms
Self CUDA time total: 1.951ms

=============
Profiling a * a
=============
STAGE:2024-07-08 21:40:06 30357:30357 ActivityProfilerController.cpp:314] Completed Stage: Warm Up
STAGE:2024-07-08 21:40:06 30357:30357 ActivityProfilerController.cpp:320] Completed Stage: Collection
STAGE:2024-07-08 21:40:06 30357:30357 ActivityProfilerController.cpp:324] Completed Stage: Post Processing
-------------------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  
                     Name    Self CPU %      Self CPU   CPU total %     CPU total  CPU time avg     Self CUDA   Self CUDA %    CUDA total  CUDA time avg    # of Calls  
-------------------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  
                aten::mul         1.54%      30.000us         2.26%      44.000us      44.000us       1.889ms       100.00%       1.889ms       1.889ms             1   而这里使用mul函数， mul函数要比pow快一些
          cudaEventRecord         0.36%       7.000us         0.36%       7.000us       3.500us       0.000us         0.00%       0.000us       0.000us             2  
         cudaLaunchKernel         0.72%      14.000us         0.72%      14.000us      14.000us       0.000us         0.00%       0.000us       0.000us             1  
    cudaDeviceSynchronize        97.38%       1.893ms        97.38%       1.893ms       1.893ms       0.000us         0.00%       0.000us       0.000us             1  
-------------------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  
Self CPU time total: 1.944ms
Self CUDA time total: 1.889ms

=============
Profiling a ** 2
=============
STAGE:2024-07-08 21:40:06 30357:30357 ActivityProfilerController.cpp:314] Completed Stage: Warm Up
STAGE:2024-07-08 21:40:06 30357:30357 ActivityProfilerController.cpp:320] Completed Stage: Collection
STAGE:2024-07-08 21:40:06 30357:30357 ActivityProfilerController.cpp:324] Completed Stage: Post Processing
-------------------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  
                     Name    Self CPU %      Self CPU   CPU total %     CPU total  CPU time avg     Self CUDA   Self CUDA %    CUDA total  CUDA time avg    # of Calls  
-------------------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  
                aten::pow         2.01%      39.000us         2.58%      50.000us      50.000us       1.876ms        99.89%       1.878ms       1.878ms             1  
        aten::result_type         0.05%       1.000us         0.05%       1.000us       1.000us       1.000us         0.05%       1.000us       1.000us             1  
                 aten::to         0.00%       0.000us         0.00%       0.000us       0.000us       1.000us         0.05%       1.000us       1.000us             1  
          cudaEventRecord         0.26%       5.000us         0.26%       5.000us       0.833us       0.000us         0.00%       0.000us       0.000us             6  
         cudaLaunchKernel         0.52%      10.000us         0.52%      10.000us      10.000us       0.000us         0.00%       0.000us       0.000us             1  
    cudaDeviceSynchronize        97.16%       1.883ms        97.16%       1.883ms       1.883ms       0.000us         0.00%       0.000us       0.000us             1  
-------------------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  
Self CPU time total: 1.938ms
Self CUDA time total: 1.878ms
```



## pytorch profiler

```
import torch
from torch.profiler import profile, record_function, ProfilerActivity


# ## Default way to use profiler
# with profile(activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA]) as prof:
#     for _ in range(10):
#         a = torch.square(torch.randn(10000, 10000).cuda())

# prof.export_chrome_trace("trace.json")


## With warmup and skip
# https://pytorch.org/docs/stable/profiler.html

# Non-default profiler schedule allows user to turn profiler on and off
# on different iterations of the training loop;
# trace_handler is called every time a new trace becomes available
def trace_handler(prof):
    print(prof.key_averages().table(
        sort_by="self_cuda_time_total", row_limit=-1))
    prof.export_chrome_trace("/tmp/test_trace_" + str(prof.step_num) + ".json")

with torch.profiler.profile(
    activities=[
        torch.profiler.ProfilerActivity.CPU,
        torch.profiler.ProfilerActivity.CUDA,
    ],

    # In this example with wait=1, warmup=1, active=2, repeat=1,
    # profiler will skip the first step/iteration,
    # start warming up on the second, record
    # the third and the forth iterations,
    # after which the trace will become available
    # and on_trace_ready (when set) is called;
    # the cycle repeats starting with the next step

    schedule=torch.profiler.schedule(
        wait=1,
        warmup=1,
        active=2,
        repeat=1),
    on_trace_ready=trace_handler
    # on_trace_ready=torch.profiler.tensorboard_trace_handler('./log')
    # used when outputting for tensorboard
    ) as p:
        for iter in range(10):
            torch.square(torch.randn(10000, 10000).cuda())
            # send a signal to the profiler that the next iteration has started
            p.step()
            print(p.step_num)
```