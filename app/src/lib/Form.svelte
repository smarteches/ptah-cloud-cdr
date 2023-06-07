<script>
    import { token } from '../stores/app';

    const expiration = 28800;
    const backend = 'wazo_user';
    const host = import.meta.env.VITE_API_URL;
    const tenant = import.meta.env.VITE_TENANT_ID;

    const calls = async (from = '', until = '', number = '') => {
        var url = `${host}/api/call-logd/1.0/cdr?limit=100&recurse=false`;

        if (until && until != '') url += `&until=${encodeURIComponent(until)}`;
        if (from && from != '') url += `&from=${encodeURIComponent(from)}`;
        if (number && number != '') url += `&number=${parseInt(number)}`;

        const headers = { 'X-Auth-Token': $token, 'Wazo-Tenant': tenant };
        const result = await fetch(url, { method: 'GET', headers });

        return await result.json();
    };

    let total = 0;
    let logs = [];
    let from = '';
    let until = '';
    let number = '';
    let loading = false;

    const search = async () => {
        if (!token) return alert('Erro de token');

        if (from && from != '') {
            if (!re.datetime.test(from)) return alert('Erro: Data inicial inválida (dd/mm/yyyy hh:mm)');

            var from = from.split(' ');
            from = [from[0].split('/').reverse().join('-'), from[1]].join('T') + ':00Z';
        }

        if (until && until != '') {
            if (!re.datetime.test(until)) return alert('Erro: Data final inválida (dd/mm/yyyy hh:mm)');

            var until = until.split(' ');
            until = [until[0].split('/').reverse().join('-'), until[1]].join('T') + ':00Z';
        }

        try {
            loading = true;

            const result = await calls(from, until, number);

            if ('filtered' in result) {
                total = result.filtered;
                logs = result.items;
            } else {
                alert(result.message);
            }
        } catch (e) {
            alert(e.message);
        }
        loading = false;
    };

    const download = (cdr_id, rec_id) => {
        open(`${host}/api/call-logd/1.0/cdr/${cdr_id}/recordings/${rec_id}/media?token=${token}`);
    };

    const fillUntil = () => {
        if (from && !until) {
            until = from.split(' ')[0];
        }
    };

    const auth = async () => {
        const username = prompt('Usuário:');
        const password = prompt('Senha:');
        try {
            const a = await login(username, password);

            if ('data' in a) {
                token = a.data.token;

                date1.focus();
            } else {
                alert(a.reason);
            }
        } catch (e) {
            alert(e.message);
        }
    };

    let re = {
        number: /^[0-9]{3,}$/,
        datetime: /^[0-9]{2}\/[0-9]{2}\/[0-9]{4}\s[0-9]{2}\:[0-9]{2}$/,
    };

    const masks = {
        datetime(e) {
            var code;
            if (!e) var e = window.event;
            if (e.keyCode) code = e.keyCode;
            else if (e.which) code = e.which;
            if (code == 8 || code == 46) return false;

            var val = e.target.value;
            val = val.replace(/[^0-9]/g, '');

            val = val.replace(/^([0-9]{2})([0-9]{2})([0-9]{4})([0-9]{2})([0-9]{2})/, '$1/$2/$3 $4:$5');
            val = val.replace(/\/+/g, '/');
            val = val.replace(/^([0-9]{2})([0-9]{2})([0-9]{4})([0-9]{2})/, '$1/$2/$3 $4:');
            val = val.replace(/\/+/g, '/');
            val = val.replace(/^([0-9]{2})([0-9]{2})([0-9]{4})/, '$1/$2/$3 ');
            val = val.replace(/\/+/g, '/');
            val = val.replace(/^([0-9]{2})([0-9]{2})/, '$1/$2/');
            val = val.replace(/\/+/g, '/');
            val = val.replace(/^([0-9]{2})/, '$1/');
            val = val.replace(/\/+/g, '/');

            e.target.value = val;

            if (val.length == 16) {
                if (e.target.id == 'date1') date2.focus();
                else if (e.target.id == 'date2') condo.focus();
            }
        },

        number(e) {
            var val = e.target.value;

            val = val.replace(/[^0-9]/g, '');
            val = val.replace(/^([0-9]{6}).*$/, '$1');

            e.target.value = val;
        },
    };

    let condo;
    let date1;
    let date2;
</script>

<main class="card">
    <div class="card-header d-flex justify-content-between">
        <div>
            <i class="fa fa-search" />
            <strong>Busca de Gravações</strong>
        </div>
    </div>

    <div class="card-body">
        <div class="row">
            <div class="col">
                <div class="form-floating mb-3">
                    <input class="form-control" id="date1" bind:this={date1} bind:value={from} on:keyup={masks.datetime} on:blur={fillUntil} maxlength="16" />
                    <label for="date1">Data Inicial</label>
                </div>
            </div>
            <div class="col">
                <div class="form-floating mb-3">
                    <input class="form-control" id="date2" bind:this={date2} bind:value={until} on:keyup={masks.datetime} maxlength="16" />
                    <label for="date2">Data Final</label>
                </div>
            </div>
            <div class="col">
                <div class="form-floating mb-3">
                    <input class="form-control" id="condo" bind:this={condo} bind:value={number} on:keyup={masks.number} />
                    <label for="condo">Condomínio (número)</label>
                </div>
            </div>
        </div>

        <div class="row">
            <div class="col">
                {#if loading}
                    <button class="btn btn-primary" disabled>Aguarde...</button>
                {:else}
                    <button class="btn btn-primary" on:click={search}><i class="fa fa-search" /> Buscar</button>
                {/if}
            </div>
            <div class="col">
                <span class="float-end">{total} registros encontrados</span>
            </div>
        </div>

        {#if logs.length > 0}
            <table class="table">
                <thead>
                    <tr>
                        <th>Data</th>
                        <th>Origem Nome</th>
                        <th>Origem Exten</th>
                        <th>Destino Nome</th>
                        <th>Destino Exten</th>
                        <th align="center"><i class="fa fa-clock" /></th>
                        <th align="center"><i class="fa fa-download" /></th>
                    </tr>
                </thead>
                <tbody>
                    {#each logs as cdr}
                        {#each cdr.recordings as rec}
                            <tr>
                                <td>{rec.start_time.replace(/(\d{4}\-\d{2}\-\d{2})T(\d{2}\:\d{2}).*/, '$1 $2')}</td>
                                <td>{cdr.source_internal_name}</td>
                                <td>{cdr.source_internal_extension}</td>
                                <td>{cdr.destination_name}</td>
                                <td>{cdr.destination_internal_extension}</td>
                                <td align="center">{cdr.duration || '-'}</td>
                                <td align="center">
                                    {#if cdr.duration}
                                        <button class="btn btn-sm btn-danger" on:click={() => download(cdr.id, rec.uuid)}>
                                            <i class="fa fa-download" />
                                        </button>
                                    {:else}
                                        <button class="btn btn-sm btn-danger" disabled>
                                            <i class="fa fa-download" />
                                        </button>
                                    {/if}
                                </td>
                            </tr>
                        {/each}
                    {/each}
                </tbody>
            </table>
        {/if}
    </div>

    <div class="card-header" style="font-size: 80%;">
        <i class="fa fa-phone" />
        <span>Voice Cloud - Smartech&reg; 2023</span>
        <span class="float-end fw-lighter">ALPHA 0.0.2</span>
    </div>
</main>
