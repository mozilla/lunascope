<script>
  // eslint-disable-next-line import/no-extraneous-dependencies
  import { spring } from "svelte/motion";

  export let params = { stiffness: 0.8, damping: 0.9 };
  export let value = 0;

  const springValue = spring(value, params);
  $: springValue.set(value);
</script>

<slot springValue={$springValue} />
