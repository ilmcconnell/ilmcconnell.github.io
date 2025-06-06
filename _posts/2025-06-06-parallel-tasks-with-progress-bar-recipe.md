Parallel-process computationally-bound functions over an iterable in python with a progress bar.

Usually you can do this in a very straightforward way:
```python
from tqdm.contrib.concurrent import process_map

def my_function(arg: Any):
  return arg * 10_000

iterable = list(range(100))

results = process_map(my_function, iterable, max_workers=4, chunksize=10)
```
If your iterable is very long, or has many large objects, it might speed things up to increase chunksize to cut down on the relative contribution of inter-process communication.

If you want to capture errors and the traceback in those parallel processes you have to unpack a little more.
```python
import os
import logging
from tqdm.auto import tqdm
from typing import Any, Callable, Iterable


logger = logging.getLogger(__name__)


def my_function(arg1: int, arg2: int, arg3: int):
  return sum([arg1, arg2, arg3])
  

my_static_function_args = dict(
  arg2=10,
  arg3=100,
)
  
my_iterable = list(range(10_000))


def parallel_process_my_function_and_capture_errors(
    fn: Callable[[Any, ...], Any],
    static_function_args: dict[str, Any],
    iterable: Iterable[Any],
    chunksize: int = 1,
    n_workers: int = 8,
) -> List[Any]:
    """
    run multiple parallel instances of fn, capture and log errors, tracebacks.
    """
    partial_fn = partial(fn, **static_function_args)
    max_workers = min(n_workers, os.cpu_count())
    results = []
    with tqdm(total=len(iterable)) as pbar:
        with ProcessPoolExecutor(max_workers=max_workers) as executor:
            futures = [executor.submit(partial_fn, item) for item in iterable]
            for future in concurrent.futures.as_completed(futures):
                try:
                    results.append(
                        future.result()
                    )
                except Exception as e:
                    logger.info(f"error: {e}")
                    logger.info(traceback.format_exc())
                pbar.update(1)
    
    return results


results = parallel_process_my_function(
  fn=my_function,
  static_function_args=my_static_function_args,
  iterable=my_iterable,
)
```
