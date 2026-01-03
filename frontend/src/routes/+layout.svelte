<!-- src/routes/+layout.svelte -->
<script lang="ts">
  import '../app.css';
  import { authToken } from '$lib/store';
  import { goto } from '$app/navigation';
  import Navigation from '$lib/components/Navigation.svelte';

  // Redirect to login if not authenticated
  $: if (!$authToken && typeof window !== 'undefined') {
    const path = window.location.pathname;
    // Allow auth routes
    if (!path.includes('/login') && !path.includes('/register')) {
      goto('/login');
    }
  }
</script>

<svelte:head>
  <title>Recipe Agent - Extract recipes to Mealie</title>
  <meta name="description" content="Extract recipes from videos to your Mealie instance using AI" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
</svelte:head>

{#if $authToken}
  <Navigation />
{/if}

<slot />