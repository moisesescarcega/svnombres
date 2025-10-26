<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabase';

  interface Familia {
    id?: number;
    tipo: string;
    nombre: string;
    ancho: number;
    alto: number;
    nivel_desplante: string;
    estado: string;
    edificio_modelo: string;
    edificio_localiza: string;
    parametros: string;
    referencia: string;
    revisor: string;
  }
  let familias = $state<Familia[]>([]);
  let estadoFamilia = $state('Muro');
  let categorias = ['Muro', 'Puerta', 'Ventana', 'Louver'];

  let inputNombre = $state('');
  let inputAncho = $state('');
  let inputAlto = $state('');
  let inputPosicion = $state(0);
  let inputOrigen = $state('');
  let inputReferencia = $state('');
  let showModal = $state(false);
  let selectedFamilia = $state<Familia | null>(null);
  let feedbackMessage = $state('');

  let familiasFiltradas = $derived(
    familias.filter((familia) => {
      const tipoMatch = familia.tipo === estadoFamilia;
      const nombreMatch = inputNombre ? familia.nombre.toLowerCase().includes(inputNombre.toLowerCase()) : true;
      const anchoFilterActive = !(inputAncho == null || String(inputAncho).trim() === '');
      const altoFilterActive = !(inputAlto == null || String(inputAlto).trim() === '');
      const anchoMatch = anchoFilterActive ? formatNumber(familia.ancho) === formatNumber(inputAncho) : true;
      const altoMatch = altoFilterActive ? formatNumber(familia.alto) === formatNumber(inputAlto) : true;
      return tipoMatch && nombreMatch && anchoMatch && altoMatch;
    })
  );

  let isCrearButtonDisabled = $derived(
    !inputNombre || (inputAncho == null || String(inputAncho).trim() === '') || !inputOrigen || (familiasFiltradas && familiasFiltradas.length > 0 )
  );

  async function createFamilia() {
    const nombre = toPascalCase(inputNombre);
    const payload = {
      tipo: estadoFamilia,
      nombre,
      ancho: inputAncho === '' ? null : Number(inputAncho),
      alto: inputAlto === '' ? null : Number(inputAlto),
      nivel_desplante: inputPosicion,
      edificio_modelo: inputOrigen,
      referencia: inputReferencia,
      estado: 'Borrador'
    };

    console.debug('createFamilia payload:', payload);
    const { data, error } = await supabase.from('familiasrevit').insert(payload).select().single();
    if (error) {
      console.log('supabase insert error', error);
      setFeedbackMessage('Error al crear la familia:' + error.message, 'error');
      return;
    }
    const normalized = {
      ...data,
      ancho: typeof data.ancho === 'string' ? parseFloat(data.ancho) : data.ancho,
      alto: typeof data.alto === 'string' ? parseFloat(data.alto) : data.alto,
    };
    familias.push(normalized);
    inputNombre = '';
    inputAncho = '';
    inputAlto = '';
    inputPosicion = 0;
    inputOrigen = '';
    inputReferencia = '';
    setFeedbackMessage('Familia creada exitosamente')
  }

  onMount(async () => {
    try {
        const { data, error } = await supabase.from('familiasrevit').select('*');
        if (error) {
          console.error('supabase select error', error);
          return;
        }
        familias = data.map((f: any) => ({
          ...f,
          ancho: typeof f.ancho === 'string' ? parseFloat(f.ancho) : f.ancho,
          alto: typeof f.alto === 'string' ? parseFloat(f.alto) : f.alto,
        }));
      } catch (err) {
        console.error('Error fetching data:', err);
    }
  });

  function formatNumber(num: number | string) {
    if (num === null || num === undefined || String(num).trim() === '') return '';
    const number = typeof num === 'string' ? parseFloat(num) : num;
    if (!isFinite(number) || isNaN(number)) {
      return '';
    }
    // If the numeric value is exactly zero, present as empty string
    if (number === 0) return '';

    // Treat as integer when it's mathematically an integer (avoid floating precision issues)
    if (Number.isFinite(number) && Math.round(number) === number) {
      return number.toFixed(0);
    }
    // Otherwise round to one decimal
    return Number(number.toFixed(1)).toString();
  }

  function toPascalCase(value: string) {
    if (!value) return '';
    // Normalize and remove diacritics (á -> a, ñ -> n, etc.)
    const normalized = value.normalize('NFD').replace(/[\u0300-\u036f]/g, '');
    // Replace any non-alphanumeric characters with a space
    const cleaned = normalized.replace(/[^0-9A-Za-z]+/g, ' ');
    // Split into words, capitalize each, join without separator
    return cleaned
      .trim()
      .split(/\s+/)
      .filter(Boolean)
      .map(w => w.charAt(0).toUpperCase() + w.slice(1).toLowerCase())
      .join('_');
  }

  async function copyFamiliaToClipboard(familia: Familia) {
    const altoPart = formatNumber(familia.alto) ? `x${formatNumber(familia.alto)}cm` : 'cm';
    const label = `${familia.tipo}-${familia.nombre}-${formatNumber(familia.ancho)}${altoPart}`;
    try {
      if (navigator.clipboard && navigator.clipboard.writeText) {
        await navigator.clipboard.writeText(label);
      } else {
        // Fallback para navegadores más antiguos
        const textarea = document.createElement('textarea');
        textarea.value = label;
        textarea.style.position = 'fixed';
        textarea.style.left = '-9999px';
        document.body.appendChild(textarea);
        textarea.select();
        document.execCommand('copy');
        textarea.remove();
      }
      setFeedbackMessage(`Copiado al portapapeles: ${label}`);
    } catch (err) {
      console.error('Error copiando al portapapeles:', err);
      setFeedbackMessage('Error al copiar al portapapeles', 'error');
    }
  }

  function openModal(familia: Familia) {
    selectedFamilia = { ...familia };
    showModal = true;
  }

  async function updateFamilia() {
    if (!selectedFamilia) return;

    try {
      // Allowed enum values (must match backend ENUM)
      const allowedEstados = ['Borrador', 'En revision', 'Aprobada', 'Desactualizada'];

      if (!allowedEstados.includes(selectedFamilia.estado)) {
        setFeedbackMessage(`Estado inválido: ${selectedFamilia.estado}`, 'error');
        console.error('Estado inválido antes de enviar update:', selectedFamilia.estado);
        return;
      }

      // Build payload and normalize numeric fields to proper types
      const payload: any = { ...selectedFamilia };
      payload.nombre = toPascalCase(String(payload.nombre ?? ''));
      if (payload.ancho !== null && payload.ancho !== undefined && payload.ancho !== '') {
        payload.ancho = typeof payload.ancho === 'string' ? parseFloat(payload.ancho) : payload.ancho;
      }
      if (payload.alto !== null && payload.alto !== undefined && payload.alto !== '') {
        payload.alto = typeof payload.alto === 'string' ? parseFloat(payload.alto) : payload.alto;
      }

      console.debug('updateFamilia payload:', payload);

      const { data, error } = await supabase
        .from('familiasrevit')
        .update(payload)
        .eq('id', selectedFamilia.id)
        .select()
        .single();

      if (error) {
        console.error('supabase update error', error);
        setFeedbackMessage('Error al actualizar la familia: ' + error.message, 'error');
        return;
      }
      const index = familias.findIndex(f => f.id === selectedFamilia!.id);
      if (index !== -1) familias[index] = data;
      showModal = false;
      setFeedbackMessage('Familia actualizada exitosamente');
    } catch (error) {
      setFeedbackMessage('Error de conexión al actualizar', 'error');
    }
  }

  async function deleteFamilia(id: number) {
    if (!confirm('¿Estás seguro de eliminar esta familia?')) return;

    try {
      const { error } = await supabase.from('familiasrevit').delete().eq('id', id);
     if (error) {
        console.error('supabase delete error', error);
        setFeedbackMessage('Error al eliminar la familia', 'error');
        return;
      }
      familias = familias.filter(f => f.id !== id);
      setFeedbackMessage('Familia eliminada exitosamente');
    } catch (error) {
      setFeedbackMessage('Error de conexión al eliminar', 'error');
    }
  }

  function setFeedbackMessage(message: string, type: 'success' | 'error' = 'success') {
    feedbackMessage = message;
    setTimeout(() => {
      feedbackMessage = '';
    }, 3000);
  }
</script>

<div class="min-h-screen flex flex-col">
  <div class="container mx-auto p-4 flex-grow">
    <div id="formCreacionFamilia" class="top-0 left-0 right-0 z-30 bg-base-100 shadow">
      <div class="container mx-auto p-4">
        <div class="flex flex-wrap gap-4 mb-4">
      <div>
        <label class="label" for="inputTipo">Tipo</label>
        <input id="inputTipo" type="text" bind:value={estadoFamilia} class="input input-bordered input-md w-full max-w-xs" disabled />
      </div>
      <div class="form-control">
      <label class="label" for="inputNombre">Nombre de Clave</label>
      <input id="inputNombre" type="text" bind:value={inputNombre} onblur={() => inputNombre = toPascalCase(inputNombre)} class="input input-bordered input-md w-full max-w-xs" />
      </div>
      <div class="form-control">
        <label class="label" for="inputAncho">Ancho</label>
        <input id="inputAncho" type="number" step="0.1" placeholder="Ancho" bind:value={inputAncho} class="input input-bordered input-md w-full max-w-xs" />
      </div>
      <div class="form-control">
      <label class="label" for="inputAlto">Alto</label>
      <input id="inputAlto" type="number" step="0.1" placeholder="Alto" bind:value={inputAlto} class="input input-bordered input-md w-full max-w-xs" />
      </div>
      <div class="form-control">
      <label class="label" for="inputPosicion">Nivel/Posición</label>
      <input id="inputPosicion" type="number" placeholder="Posicion" bind:value={inputPosicion} class="input input-bordered input-md w-full max-w-xs" />
      </div>
      <div class="form-control">
      <label class="label" for="inputOrigen">Origen</label>
      <select id="inputOrigen" bind:value={inputOrigen} class="select select-bordered input-md w-full max-w-xs">
        <option value="">Selecciona...</option>
        <option value="EA">EA</option>
        <option value="EB">EB</option>
        <option value="EC">EC</option>
        <option value="ED">ED</option>
        <option value="EE">EE</option>
        <option value="EVP">EVP</option>
        <option value="EXT">EXT</option>
      </select>
      </div>
      <div class="form-control">
      <label class="label" for="inputReferencia">Referencia</label>
      <input id="inputReferencia" type="text" placeholder="Referencia" bind:value={inputReferencia} class="input input-bordered input-md w-full max-w-xs" />
      </div>
      <div class="flex items-end">
      <button class="btn btn-lg" onclick={createFamilia} disabled={isCrearButtonDisabled}>Crear</button>
      </div>
        </div>
        <div class="flex flex-wrap gap-2 mb-4">
        {#each categorias as categoria}
          <button
            class="btn"
            class:btn-primary={estadoFamilia === categoria}
            class:btn-secundary={estadoFamilia !== categoria}
            onclick={() => (estadoFamilia = categoria)}
          >
            {categoria}
          </button>
        {/each}
        </div>
      </div>
    </div>

    <!-- Main content: add top margin to avoid being hidden behind the fixed header -->
    <main class="mb-20 z-999">

      <!-- Table container: fixed max height, scrollable body-->
      <div class="overflow-auto mb-4 max-h-[calc(100vh-6rem-4rem)]"> 
        <table class="table w-full">
          <thead>
            <tr>
              <th class="top-0 bg-base-200 z-10 w-1/4 text-left">Familia/Tipo (Click para copiar)</th>
              <th class="top-0 bg-base-200 z-10 w-12 text-left">Ancho</th>
              <th class="top-0 bg-base-200 z-10 w-12 text-left">Alto</th>
              <th class="top-0 bg-base-200 z-10 w-12 text-left">Nivel</th>
              <th class="top-0 bg-base-200 z-10 w-1/8 text-left">Estado</th>
              <th class="top-0 bg-base-200 z-10 w-24 text-left">Origen</th>
              <th class="top-0 bg-base-200 z-10 w-1/8 text-left">Ubicacion</th>
              <th class="top-0 bg-base-200 z-10 w-1/8 text-left">Parametros</th>
              <th class="top-0 bg-base-200 z-10 w-1/8 text-left">Referencia</th>
              <th class="top-0 bg-base-200 z-10 w-24 text-left">Revisor</th>
              <th class="top-0 bg-base-200 z-10 w-1/8 text-left">Acciones</th>
            </tr>
          </thead>
          <tbody>
            {#each familiasFiltradas as familia}
              <tr>
                <td
                  class="cursor-pointer"
                  title="Click para copiar"
                  onclick={() => copyFamiliaToClipboard(familia)}
                >{familia.tipo}-{familia.nombre}-{formatNumber(familia.ancho)}{formatNumber(familia.alto) ? `x${formatNumber(familia.alto)}cm` : `cm`}</td>
                <td>{formatNumber(familia.ancho)}</td>
                <td>{formatNumber(familia.alto)}</td>
                <td>{familia.nivel_desplante}</td>
                <td>{familia.estado}</td>
                <td>{familia.edificio_modelo}</td>
                <td>{familia.edificio_localiza}</td>
                <td>{familia.parametros}</td>
                <td>{familia.referencia}</td>
                <td>{familia.revisor}</td>
                <td>
                  <!-- svelte-ignore a11y_consider_explicit_label -->
                  <button class="btn btn-ghost btn-xs" onclick={() => openModal(familia)}>
                    <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15.232 5.232l3.536 3.536m-2.036-5.036a2.5 2.5 0 113.536 3.536L6.5 21.036H3v-3.5L14.732 3.732z" /></svg>
                  </button>
                  <!-- svelte-ignore a11y_consider_explicit_label -->
                  <button class="btn btn-ghost btn-xs" onclick={() => deleteFamilia(familia.id!)}>
                    <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" /></svg>
                  </button>
                </td>
              </tr>
            {/each}
          </tbody>
        </table>
      </div>
    </main>
  </div>

  {#if showModal && selectedFamilia}
    <dialog class="modal modal-open">
      <div class="modal-box">
        <h3 class="font-bold text-lg">Editar Familia</h3>
        <form onsubmit={(e) => { e.preventDefault(); updateFamilia(); }}>
          <div class="space-y-4">
            <div class="form-control flex items-center justify-between gap-4">
              <label class="label w-1/3 text-left" for="editNombre">Nombre</label>
              <input type="text" id="editNombre" bind:value={selectedFamilia.nombre} class="input input-bordered w-2/3" />
            </div>

            <div class="form-control flex items-center justify-between gap-4">
              <label class="label w-1/3 text-left" for="editEstado">Estado</label>
              <select id="editEstado" bind:value={selectedFamilia.estado} class="select select-bordered w-2/3">
                <option value="Borrador">Borrador</option>
                <option value="En revision">En revisión</option>
                <option value="Aprobada">Aprobada</option>
                <option value="Desactualizada">Desactualizada</option>
              </select>
            </div>

            <div class="form-control flex items-center justify-between gap-4">
              <label class="label w-1/3 text-left" for="editAncho">Ancho</label>
              <input type="number" step="0.1" id="editAncho" bind:value={selectedFamilia.ancho} class="input input-bordered w-2/3" />
            </div>

            <div class="form-control flex items-center justify-between gap-4">
              <label class="label w-1/3 text-left" for="editAlto">Alto</label>
              <input type="number" step="0.1" id="editAlto" bind:value={selectedFamilia.alto} class="input input-bordered w-2/3" />
            </div>

            <div class="form-control flex items-center justify-between gap-4">
              <label class="label w-1/3 text-left" for="editPosicion">Nivel/Posición</label>
              <input type="number" id="editPosicion" bind:value={selectedFamilia.nivel_desplante} class="input input-bordered w-2/3" />
            </div>

            <div class="form-control flex items-center justify-between gap-4">
              <label class="label w-1/3 text-left" for="editOrigen">Origen</label>
              <select id="editOrigen" bind:value={selectedFamilia.edificio_modelo} class="select select-bordered w-2/3">
                <option value="">Selecciona...</option>
                <option value="EA">EA</option>
                <option value="EB">EB</option>
                <option value="EC">EC</option>
                <option value="ED">ED</option>
                <option value="EE">EE</option>
                <option value="EVP">EVP</option>
                <option value="EXT">EXT</option>
              </select>
            </div>

            <div class="form-control flex items-center justify-between gap-4">
              <label class="label w-1/3 text-left" for="editLocaliza">Ubicación</label>
              <input type="text" id="editLocaliza" bind:value={selectedFamilia.edificio_localiza} class="input input-bordered w-2/3" />
            </div>

            <div class="form-control flex items-center justify-between gap-4">
              <label class="label w-1/3 text-left" for="editParametros">Parámetros</label>
              <input type="text" id="editParametros" bind:value={selectedFamilia.parametros} class="input input-bordered w-2/3" />
            </div>

            <div class="form-control flex items-center justify-between gap-4">
              <label class="label w-1/3 text-left" for="editReferencia">Referencia</label>
              <input type="text" id="editReferencia" bind:value={selectedFamilia.referencia} class="input input-bordered w-2/3" />
            </div>

            <div class="form-control flex items-center justify-between gap-4">
              <label class="label w-1/3 text-left" for="editRevisor">Revisor</label>
              <input type="text" id="editRevisor" bind:value={selectedFamilia.revisor} class="input input-bordered w-2/3" />
            </div>
          </div>

          <div class="modal-action">
            <button type="submit" class="btn btn-primary">Guardar</button>
            <button type="button" class="btn" onclick={() => showModal = false}>Cancelar</button>
          </div>
        </form>
      </div>
    </dialog>
  {/if}

  <footer class="footer footer-center fixed bottom-0 left-0 right-0 p-4 bg-base-300 text-base-content z-40">
    {#if feedbackMessage}
      <div class="alert" class:alert-success={!feedbackMessage.includes('Error')} class:alert-error={feedbackMessage.includes('Error')}>
        <span>{feedbackMessage}</span>
      </div>
    {/if}
    <div>
      <p>Nombres de familias para elementos en Revit - Proyecto Yecapixtla</p>
    </div>
    <div class="flex mt-4">
      <span class="mr-2">Tema:</span>
      <select class="select select-bordered select-sm" onchange={(e) => document.documentElement.setAttribute('data-theme', e.currentTarget.value)}>
        <option value="dracula">Oscuro</option>
        <option value="retro">Claro</option>
      </select>
    </div>
  </footer>
</div>
