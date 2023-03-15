<script>
  import { onMount } from "svelte";

  const expiration = 28800;
  const backend = "wazo_user";
  const host = "https://kihonlab.com";
  const tenantId = "399b32db-d1d2-432c-a85d-aed5f7939b1c";

  const login = async (username, password) => {
    const auth = { expiration, backend };
    const url = `${host}/api/auth/0.1/token`;
    const token = btoa(`${username}:${password}`);

    const result = await fetch(url, {
      headers: { Authorization: `Basic ${token}` },
      body: JSON.stringify(auth),
      method: "POST",
    });

    return await result.json();
  };

  const calls = async (from, until, number) => {
    var url = `${host}/api/call-logd/1.0/cdr`;
    url += `?from=${encodeURIComponent(from)}`;
    url += `&until=${encodeURIComponent(until)}`;
    url += `&number=${parseInt(number)}`;
    url += `&limit=10&recurse=false`;

    const result = await fetch(url, {
      headers: { "X-Auth-Token": token },
      method: "GET",
    });

    return await result.json();
  };

  let from;
  let until;
  let number;
  let token;
  let total = 0;
  let logs = [];
  let loading = false;

  const search = async () => {
    if (!from) return alert("Erro: Data inicial");
    if (!until) return alert("Erro: Data final");
    if (!token) return alert("Erro de token");

    console.log({ from, until });

    if (!re.datetime.test(from)) return alert("Erro: Data inicial inválida (dd/mm/yyyy hh:mm)");
    if (!re.datetime.test(until)) return alert("Erro: Data final inválida (dd/mm/yyyy hh:mm)");

    var _from = from.split(" ");
    _from = [_from[0].split("/").reverse().join("-"), _from[1]].join(" ");

    var _until = until.split(" ");
    _until = [_until[0].split("/").reverse().join("-"), _until[1]].join(" ");

    try {
      loading = true;

      const result = await calls(_from, _until, number);

      if ("filtered" in result) {
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
      until = from.split(" ")[0];
    }
  };

  onMount(async () => {
    const username = prompt("Usuário:");
    const password = prompt("Senha:");
    try {
      const a = await login(username, password);

      if ("data" in a) {
        token = a.data.token;

        date1.focus();
      } else {
        alert(a.reason);
      }
    } catch (e) {
      alert(e.message);
    }
  });

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
      val = val.replace(/[^0-9]/g, "");

      val = val.replace(/^([0-9]{2})([0-9]{2})([0-9]{4})([0-9]{2})([0-9]{2})/, "$1/$2/$3 $4:$5");
      val = val.replace(/\/+/g, "/");
      val = val.replace(/^([0-9]{2})([0-9]{2})([0-9]{4})([0-9]{2})/, "$1/$2/$3 $4:");
      val = val.replace(/\/+/g, "/");
      val = val.replace(/^([0-9]{2})([0-9]{2})([0-9]{4})/, "$1/$2/$3 ");
      val = val.replace(/\/+/g, "/");
      val = val.replace(/^([0-9]{2})([0-9]{2})/, "$1/$2/");
      val = val.replace(/\/+/g, "/");
      val = val.replace(/^([0-9]{2})/, "$1/");
      val = val.replace(/\/+/g, "/");

      e.target.value = val;

      if (val.length == 16) {
        if (e.target.id == "date1") date2.focus();
        else if (e.target.id == "date2") condo.focus();
      }
    },

    number(e) {
      var val = e.target.value;

      val = val.replace(/[^0-9]/g, "");
      val = val.replace(/^([0-9]{6}).*$/, "$1");

      e.target.value = val;
    },
  };

  let condo;
  let date1;
  let date2;
</script>

<main class="card">
  <div class="card-header">
    <i class="fa fa-search" />
    <strong>Busca de Gravações</strong>
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
                <td>{rec.start_time.replace(/(\d{4}\-\d{2}\-\d{2})T(\d{2}\:\d{2}).*/, "$1 $2")}</td>
                <td>{cdr.source_internal_name}</td>
                <td>{cdr.source_internal_extension}</td>
                <td>{cdr.destination_name}</td>
                <td>{cdr.destination_internal_extension}</td>
                <td align="center">{cdr.duration || "-"}</td>
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
    <span>Ptah Cloud - Smartech&reg; 2023</span>
    <span class="float-end fw-lighter">ALPHA 0.0.1</span>
  </div>
</main>
