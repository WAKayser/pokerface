<script>
    import '@skeletonlabs/skeleton/themes/theme-skeleton.css';

import '@skeletonlabs/skeleton/styles/skeleton.css';
  import "../app.postcss";
  import { autoModeWatcher } from '@skeletonlabs/skeleton';
</script>

<svelte:head>{@html `<script>${autoModeWatcher.toString()} autoModeWatcher();</script>`}</svelte:head>

<slot />
