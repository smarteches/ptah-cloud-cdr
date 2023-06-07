<script>
    import Login from './lib/Login.svelte';
    import Form from './lib/Form.svelte';

    import { token } from './stores/app';
</script>

{#if $token}
    <Form />
{:else}
    <Login />
{/if}
