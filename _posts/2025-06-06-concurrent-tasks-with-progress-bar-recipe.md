Quite often you want to be able to concurrently process work over an iterable in python with a progress bar. Either parallel processing computationally bound tasks or multi-threading I/O bound tasks.

Usually you can do this in a very straightforward way:
```python
from tqdm.contrib.concurrent import process_map #, thread_map


def my_function(arg: Any):
  return arg * 10_000

iterable = list(range(100))

# results = thread_map(my_function, iterable, max_workers=4, chunksize=1)
results = process_map(my_function, iterable, max_workers=4, chunksize=10)
```
If your iterable is very long, or contains large objects (maybe these could be pre-computed?), it might speed things up to increase chunksize to cut down on the relative contribution of inter-process communication.

When you're iterating over large amounts of data, unexpected conditions can throw exceptions which might stall processing. Ideally you can capture those errors to see what is going wrong, but keep your processing going for the majority of cases. If you want to capture errors and the traceback in those parallel processes you have to unpack a little more.
```python
import os
import logging
from tqdm.auto import tqdm
from typing import Any, Callable, Iterable


logger = logging.getLogger(__name__)


def my_function(chunk: List[int], arg1: int, arg2: int, arg3: int) -> List[int]:
  """
  function that does the actual work you're interested in doing concurrently.
  NB this is defined to iterate on a chunk i.e. a list of items - easier to
  control chunksize and interprocess communication overhead this way.
  """
  processed_items = []
  for item in chunk:
    processed_item = sum([item * arg1, arg2, arg3])
    processed_items.append(processed_item)
  return processed_items


def parallel_process_my_function_and_capture_errors(
    fn: Callable[[Any, ...], Any],
    static_function_args: dict[str, Any],
    iterable: Iterable[Any],
    chunksize: int = 10,
    n_workers: int = 8,
) -> List[Any]:
    """
    run multiple parallel instances of fn with progress bar while also capturing and logging errors plus tracebacks.
    """
    partial_fn = partial(fn, **static_function_args)  # define partial function that then only needs the iterator chunk argument
    max_workers = min(n_workers, os.cpu_count())
    chunks = [iterable[i:i + chunksize] for i in range(0, len(iterable), chunksize)]
    results = []
    with tqdm(total=len(iterable)) as pbar:
        # with ThreadPoolExecutor(max_workers=max_workers) as executor:  # for I/O bound tasks
        with ProcessPoolExecutor(max_workers=max_workers) as executor:  # for compute bound tasks
            futures = [executor.submit(partial_fn, chunk) for chunk in chunks]
            for future in concurrent.futures.as_completed(futures):
                try:
                    results.extend(
                        future.result()
                    )
                except Exception as e:
                    logger.info(f"error: {e}")
                    logger.info(traceback.format_exc())
                pbar.update(1)
    
    return results


if __name__ == "__main__":
  my_static_function_args = dict(
    arg2=10,
    arg3=100,
  )
  my_iterable = list(range(10_000)) 

  results = parallel_process_my_function_and_capture_errors(
    fn=my_function,
    static_function_args=my_static_function_args,
    iterable=my_iterable,
  )
```
