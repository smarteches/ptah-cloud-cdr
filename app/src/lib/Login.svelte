<script>
    import { onMount } from 'svelte';
    import { token } from '../stores/app';

    let error = null;
    let username = '';
    let password = '';
    let loading = false;
    let certificate = false;

    const tenant = import.meta.env.VITE_TENANT_ID;
    const host = import.meta.env.VITE_API_URL;
    const url = `${host}/api/auth/0.1/token`;
    const backend = 'wazo_user';
    const expiration = 28800;

    const login = async () => {
        loading = true;

        try {
            const auth = { expiration, backend };
            const basic = btoa(`${username}:${password}`);

            const result = await fetch(url, {
                headers: { Authorization: `Basic ${basic}` },
                body: JSON.stringify(auth),
                method: 'POST',
            });

            const data = await result.json();

            if ('data' in data) {
                token.set(data.data.token);
            } else {
                error = data.reason;
            }
        } catch (e) {
            alert(`Você aceitou o certificado?\nErro: ${e.message}`);
        }

        loading = false;
    };

    const cert = () => window.open(url);

    const check = async () => {
        try {
            await fetch(url);
            certificate = true;
        } catch (e) {
            certificate = false;
            window.addEventListener('focus', check);
        }
    };

    onMount(check);
</script>

{#if error}
    <div class="alert alert-danger">{error}</div>
{/if}

{#if !certificate}
    <div class="alert alert-danger">
        <i class="fa fa-warning" />
        <span>Você precisa aceitar o certificado, clicando no ícone vermelho.</span>
    </div>
{/if}

<main class="card">
    <div class="card-header d-flex items-center justify-content-between">
        <div>
            <i class="fa fa-lock" />
            <strong>Login</strong>
        </div>
        {#if certificate}
            <button class="btn btn-sm btn-success" on:click={() => {}}>
                <i class="fa fa-check" />
            </button>
        {:else}
            <button class="btn btn-sm btn-danger" on:click={cert}>
                <i class="fa fa-warning" />
            </button>
        {/if}
    </div>

    <div class="card-body">
        <div class="row">
            <div class="col">
                <div class="form-floating mb-3">
                    <input class="form-control" id="username" bind:value={username} />
                    <label for="username">Login</label>
                </div>
            </div>
            <div class="col">
                <div class="form-floating mb-3">
                    <input class="form-control" id="password" bind:value={password} type="password" />
                    <label for="password">Senha</label>
                </div>
            </div>
        </div>

        <div class="row">
            <div class="col">
                {#if loading}
                    <button class="btn btn-primary" disabled>Aguarde...</button>
                {:else}
                    <button class="btn btn-primary" on:click={login}>Entrar</button>
                {/if}
            </div>
        </div>
    </div>

    <div class="card-header" style="font-size: 80%;">
        <i class="fa fa-phone" />
        <span>Voice Cloud - Smartech&reg; 2023</span>
        <span class="float-end fw-lighter">ALPHA 0.0.2</span>
    </div>
</main>
