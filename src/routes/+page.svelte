<script lang="ts">
  import { onMount } from 'svelte';

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
    !inputNombre || (inputAncho == null || String(inputAncho).trim() === '') || !inputOrigen
  );

  async function createFamilia() {
    const newFamilia = {
      tipo: estadoFamilia,
      nombre: inputNombre,
      ancho: Number(inputAncho),
      alto: Number(inputAlto),
      nivel_desplante: inputPosicion,
      edificio_modelo: inputOrigen,
      referencia: inputReferencia,
      estado: 'Borrador', // O el estado que desees por defecto
    };

    try {
      // Debug: log payload before sending
      console.debug('createFamilia payload:', newFamilia);

      const response = await fetch('https://familias.bimycp.site/api/familias', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(newFamilia),
      });

      if (response.ok) {
        const createdFamilia = await response.json();
        const normalized = {
          ...createdFamilia.data,
          ancho: typeof createdFamilia.data.ancho === 'string' ? parseFloat(createdFamilia.data.ancho) : createdFamilia.data.ancho,
          alto: typeof createdFamilia.data.alto === 'string' ? parseFloat(createdFamilia.data.alto) : createdFamilia.data.alto,
        };
        familias.push(normalized);
        // Reset input fields
  inputNombre = '';
  inputAncho = '';
  inputAlto = '';
        inputPosicion = 0;
        inputOrigen = '';
        inputReferencia = '';
        setFeedbackMessage('Familia creada exitosamente');
      } else {
        // Try to read server error message (if any) to help debugging
        let errText = '';
        try {
          errText = await response.text();
        } catch (e) {
          errText = `${response.status} ${response.statusText}`;
        }
        console.error('createFamilia failed:', response.status, response.statusText, errText);
        setFeedbackMessage(`Error al crear la familia: ${errText}`.slice(0, 200), 'error');
      }
    } catch (error) {
      console.error('Error creating familia (network):', error);
      setFeedbackMessage('Error de conexión al crear la familia', 'error');
    }
  }

  onMount(async () => {
    try {
      const response = await fetch('https://familias.bimycp.site/api/familias');

      if (response.ok) {
        const data = await response.json();
        // Normalize ancho/alto to numbers to keep consistent types
        familias = data.data.map((f: any) => ({
          ...f,
          ancho: typeof f.ancho === 'string' ? parseFloat(f.ancho) : f.ancho,
          alto: typeof f.alto === 'string' ? parseFloat(f.alto) : f.alto,
        }));
      } else {
        console.error('Error fetching data:', response.statusText);
      }
    } catch (error) {
      console.error('Error fetching data:', error);
    }
  });

  function formatNumber(num: number | string) {
    if (num === null || num === undefined || String(num).trim() === '') return '';
    const number = typeof num === 'string' ? parseFloat(num) : num;
    if (!isFinite(number) || isNaN(number)) {
      return '';
    }
    if (Math.abs(number % 1) < Number.EPSILON) {
      return number.toFixed(0);
    } else {
      return number.toFixed(1);
    }
  }

  function openModal(familia: Familia) {
    selectedFamilia = { ...familia };
    showModal = true;
  }

  async function updateFamilia() {
    if (!selectedFamilia) return;

    try {
      const response = await fetch(`https://familias.bimycp.site/api/familias/${selectedFamilia!.id}`, {
        method: 'PUT',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(selectedFamilia),
      });

      if (response.ok) {
        const updatedFamilia = await response.json();
        const index = familias.findIndex(f => f.id === selectedFamilia!.id);
        if (index !== -1) {
          familias[index] = updatedFamilia.data;
        }
        showModal = false;
        setFeedbackMessage('Familia actualizada exitosamente');
      } else {
        setFeedbackMessage('Error al actualizar la familia', 'error');
      }
    } catch (error) {
      setFeedbackMessage('Error de conexión al actualizar', 'error');
    }
  }

  async function deleteFamilia(id: number) {
    if (!confirm('¿Estás seguro de eliminar esta familia?')) return;

    try {
      const response = await fetch(`https://familias.bimycp.site/api/familias/${id}`, {
        method: 'DELETE',
      });

      if (response.ok) {
        familias = familias.filter(f => f.id !== id);
        setFeedbackMessage('Familia eliminada exitosamente');
      } else {
        setFeedbackMessage('Error al eliminar la familia', 'error');
      }
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
        <label class="label" for="inputNombre">Tipo</label>
        <input id="inputTipo" type="text" bind:value={estadoFamilia} class="input input-bordered input-lg w-full max-w-xs" disabled />
      </div>
      <div class="form-control">
      <label class="label" for="inputNombre">Clave</label>
      <input id="inputNombre" type="text" placeholder="Nombre o clave" bind:value={inputNombre} class="input input-bordered input-lg w-full max-w-xs" />
      </div>
      <div class="form-control">
        <label class="label" for="inputAncho">Ancho</label>
        <input id="inputAncho" type="number" placeholder="Ancho" bind:value={inputAncho} class="input input-bordered input-lg w-full max-w-xs" />
      </div>
      <div class="form-control">
      <label class="label" for="inputAlto">Alto</label>
      <input id="inputAlto" type="number" placeholder="Alto" bind:value={inputAlto} class="input input-bordered input-lg w-full max-w-xs" />
      </div>
      <div class="form-control">
      <label class="label" for="inputPosicion">Posición</label>
      <input id="inputPosicion" type="number" placeholder="Posicion" bind:value={inputPosicion} class="input input-bordered input-lg w-full max-w-xs" />
      </div>
      <div class="form-control">
      <label class="label" for="inputOrigen">Origen</label>
      <input id="inputOrigen" type="text" placeholder="Origen" bind:value={inputOrigen} class="input input-bordered input-lg w-full max-w-xs" />
      </div>
      <div class="form-control">
      <label class="label" for="inputReferencia">Referencia</label>
      <input id="inputReferencia" type="text" placeholder="Referencia" bind:value={inputReferencia} class="input input-bordered input-lg w-full max-w-xs" />
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
              <th class="top-0 bg-base-200 z-10 w-1/4 text-left">Familia/Tipo</th>
              <th class="top-0 bg-base-200 z-10 w-12 text-left">Ancho</th>
              <th class="top-0 bg-base-200 z-10 w-12 text-left">Alto</th>
              <th class="top-0 bg-base-200 z-10 w-12 text-left">Posicion</th>
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
                <td>{familia.tipo}-{familia.nombre}-{formatNumber(familia.ancho)}x{formatNumber(familia.alto)}cm</td>
                <td>{familia.ancho}</td>
                <td>{familia.alto}</td>
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
          <div class="form-control">
            <label class="label" for="editNombre">Nombre</label>
            <input type="text" id="editNombre" bind:value={selectedFamilia.nombre} class="input input-bordered" />
          </div>
          <div class="form-control">
            <label class="label" for="editAncho">Ancho</label>
            <input type="number" id="editAncho" bind:value={selectedFamilia.ancho} class="input input-bordered" />
          </div>
          <div class="form-control">
            <label class="label" for="editAlto">Alto</label>
            <input type="number" id="editAlto" bind:value={selectedFamilia.alto} class="input input-bordered" />
          </div>
          <div class="form-control">
            <label class="label" for="editPosicion">Posicion</label>
            <input type="number" id="editPosicion" bind:value={selectedFamilia.nivel_desplante} class="input input-bordered" />
          </div>
          <div class="form-control">
            <label class="label" for="editOrigen">Origen</label>
            <input type="text" id="editOrigen" bind:value={selectedFamilia.edificio_modelo} class="input input-bordered" />
          </div>
          <div class="form-control">
            <label class="label" for="editParametros">Parámetros</label>
            <input type="text" id="editParametros" bind:value={selectedFamilia.parametros} class="input input-bordered" />
          </div>
          <div class="form-control">
            <label class="label" for="editReferencia">Referencia</label>
            <input type="text" id="editReferencia" bind:value={selectedFamilia.referencia} class="input input-bordered" />
          </div>
          <div class="form-control">
            <label class="label" for="editRevisor">Revisor</label>
            <input type="text" id="editRevisor" bind:value={selectedFamilia.revisor} class="input input-bordered" />
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
