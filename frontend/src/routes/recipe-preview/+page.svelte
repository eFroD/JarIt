<!-- src/routes/recipe-preview/+page.svelte -->
<script lang="ts">
  import RecipePreview from '$lib/components/RecipePreview.svelte';
  import { authToken } from '$lib/store';
  import { goto } from '$app/navigation';

  // Require authentication
  $: if (!$authToken && typeof window !== 'undefined') {
    goto('/login');
  }
</script>

<main class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8">
  <RecipePreview />
</main>

<style>
  :global(main) {
    min-height: calc(100vh - 4rem);
  }
</style>