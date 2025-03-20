
# Understanding Dependencies and Asynchronous Pitfalls in Svelte 5

Svelte 5 introduces new reactive APIs like `$state`, `$derived`, and `$effect`, which help streamline state management and reactivity. However, one key challenge remains: understanding how dependencies are tracked—especially when asynchronous code is involved.

## How Dependencies Are Tracked

According to the Svelte 5 documentation, any reactive value that is read synchronously inside an `$effect` is automatically tracked as a dependency. This includes values from `$state`, `$derived`, or even `$props`. When these dependencies change, Svelte schedules the effect to re-run.

For example, consider this snippet from the documentation:

```svelte
$effect(() => {
  const context = canvas.getContext('2d');
  context.clearRect(0, 0, canvas.width, canvas.height);

  // This runs whenever `color` changes…
  context.fillStyle = color;

  setTimeout(() => {
    // …but not when `size` changes, because it's read asynchronously
    context.fillRect(0, 0, size, size);
  }, 0);
});
```

Here, `color` is tracked because it’s accessed synchronously, whereas `size` isn’t, as it’s read inside a `setTimeout` callback. This means that changes to `size` will not trigger the effect to re-run.

## The Asynchronous Issue: A Practical Example

Let’s explore an example to illustrate this behavior and see how we might work around it.

### Problematic Example

In this code, we try to update `count2` based on `count`, but the update is scheduled asynchronously. Because the asynchronous callback isn’t tracked as a dependency, `count2` doesn’t update as expected.

```svelte
<script>
  let count = $state(0);
  let count2 = $state(0);
  
  async function increment() {
    // This updates count immediately
    count += 1; 
  }

  $effect(() => {
    // The promise and setTimeout cause this to run asynchronously,
    // so the dependency on `count` is not tracked.
    new Promise((res) => setTimeout(res, 1000)).then(() => {
       count2 = count - 1;
    });
  });
</script>

<button onclick={increment}>ok</button>
<div>{count}</div>
<div>{count2}</div>
```

In this example, even when `count` is updated, `count2` remains unchanged because the asynchronous callback is outside the scope of the synchronous dependency tracking.

### Workaround: Updating Inside an Async Function

One way to address this issue is to ensure that the reactive value is read inside an asynchronous function that is directly managed by `$effect`. For instance:

```svelte
<script>
  let count = $state(0);
  let count2 = $state(0);
  
  async function increment() {
    await new Promise((res) => setTimeout(res, 1000));
    count += 1;
  }

  async function decrement(value) {
    await new Promise((res) => setTimeout(res, 1000));
    count2 = value - 1;
  }

  $effect(async () => {
    // Here, `count` is read synchronously at the beginning of the async effect.
    decrement(count);
  });
</script>

<button onclick={increment}>ok</button>
<div>{count}</div>
<div>{count2}</div>
```

By restructuring the code so that the value of `count` is accessed synchronously (even within an async effect), the dependency is correctly tracked and `count2` will be updated as expected.

### Another Approach: Using a Reference Object

If you need to capture the latest state inside an asynchronous callback, you can use a reference object to hold the current value:

```svelte
<script>
  let count = $state(0);
  let count2 = $state(0);
  
  async function increment() {
    count += 1; 
  }

  // Create a reference object to always hold the latest count
  let countRef = { current: count };
  
  // Update the reference whenever count changes
  $effect(() => {
    countRef.current = count;
  });

  function fn(ref) {
      new Promise((res) => setTimeout(res, 1000)).then(() => {
        count2 = ref.current - 1; 
      });    
  }

  $effect(() => {
    // We ensure `count` is read so that this effect re-runs when it changes
    count;
    fn(countRef);
  });
</script>

<button onclick={increment}>ok</button>
<div>{count}</div>
<div>{count2}</div>
```

This method uses a reference (`countRef`) that is updated synchronously via an `$effect` and then accessed later in the asynchronous callback, ensuring that the latest value of `count` is used.

## Conclusion

The Svelte 5 documentation provides important insights into how reactive dependencies are tracked, but the asynchronous nature of JavaScript can lead to some unexpected behavior. As shown in the examples:

- **Synchronous Reads:** Values accessed synchronously inside `$effect` are tracked.
- **Asynchronous Reads:** Values accessed after an `await` or within a `setTimeout` are not tracked, leading to potential pitfalls.
- **Workarounds:** Restructuring your code to access values synchronously or using reference objects can help overcome these issues.

While Svelte 5's approach to reactivity can initially seem complex, understanding these nuances is key to writing robust applications. By sharing solutions and workarounds, we can better navigate the challenges of asynchronous effects in modern reactive frameworks.
