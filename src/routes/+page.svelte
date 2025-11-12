<script lang="ts">
	import { onMount } from 'svelte';
	import { supabase } from '$lib/supabase';

	interface Familia {
		id?: number;
		nombre: string;
		familia_rvt: string | null;
		tipo: string | null;
		B: number | null;
		H: number | null;
		L: number | null;
		nivel_desplante: number | null;
		estado: string | null;
		edificio_modelo: string | null;
		edificio_localiza: string[] | null;
		parametros: string | null;
		referencia: string;
		revisor: string | null;
		categoria: string | null;
		nivel: string | null;
		clave: string;
		disciplina: string | null;
		propiedades: string | null;
		especificacion: string | null;
		sistema: string[] | null;
		revisiones: string | null;
	}

	let familias = $state<Familia[]>([]);
	let categoriaSeleccionada = $state('Window');
	let categorias = [
		'Door',
		'Window',
		'Curtain Wall',
		'Wall',
		'Floor',
		'Roof',
		'Ceiling',
		'Column',
		'Beam',
		'Plumbing Fixture',
		'Mechanical Equipment',
		'Air Terminal',
		'Duct',
		'Pipe',
		'Electrical Equipment',
		'Lighting Fixture',
		'Furniture',
		'Generic Model',
		'Railings',
		'Stairs',
		'Ramp',
		'Structural Foundation',
		'Site Component',
		'Landscaping',
		'Building Pad',
		'In-Place Family',
		'Equipment',
		'Fire Protection',
		'Security Device',
		'Structural Framing',
		'Structural Rebar',
		'Casework',
		'Specialty Equipment'
	];

	let inputClave = $state('');
	let inputAncho = $state<number | ''>('');
	let inputAlto = $state<number | ''>('');
	let inputPosicion = $state<number | ''>('');
	let inputUbicacion = $state<string[] | ''>([]);
  let inputOrigen = $state('');
	let inputReferencia = $state('');
  let inputSistema = $state<string[] | ''>([]);
	let showModal = $state(false);
	let selectedFamilia = $state<Familia | null>(null);
	let editingFamilia = $state<any>(null);
	// Flags to control normalization state for create and edit flows
	let createModeNormalized = $state(false);
	let editModeNormalized = $state(false);
	let feedbackMessage = $state('');

	let familiasFiltradas = $derived(
		familias.filter((familia) => {
			// If no category is selected (empty string or null), show all categories
			const categoriaMatch = !categoriaSeleccionada || String(categoriaSeleccionada).trim() === ''
				? true
				: (familia.familia_rvt || '').toLowerCase() === String(categoriaSeleccionada).toLowerCase();
			const claveMatch = inputClave
				? (String(familia.clave || '').toLowerCase().includes(String(inputClave).toLowerCase().trim()))
				: true;
			const anchoFilterActive = !(inputAncho == null || String(inputAncho).trim() === '');
			const altoFilterActive = !(inputAlto == null || String(inputAlto).trim() === '');
			// Familia.B/H are stored in meters. inputAncho/inputAlto are treated as centimetros (cm)
			// Compare by converting familia values to cm string (formatNumber) and comparing to
			// a cm-formatted string from the input (formatCmInput).
			const anchoMatch = anchoFilterActive
				? formatNumber(familia.B) === formatCmInput(inputAncho)
				: true;
			const altoMatch = altoFilterActive
				? formatNumber(familia.H) === formatCmInput(inputAlto)
				: true;
			return categoriaMatch && claveMatch && anchoMatch && altoMatch;
		})
	);

	let isCrearButtonDisabled = $derived(
		!inputClave ||
			inputAncho == null ||
			String(inputAncho).trim() === '' ||
			(Array.isArray(inputUbicacion) ? inputUbicacion.length === 0 : !inputUbicacion) ||
			(familiasFiltradas && familiasFiltradas.length > 0)
	);

	async function createFamilia(normalize = false) {
		const clave = normalize ? formatAsKey(String(inputClave ?? '')) : String(inputClave ?? '');
		// Ensure ubicacion is an array when building payload
		const ubicacionArray = Array.isArray(inputUbicacion)
			? inputUbicacion
			: inputUbicacion
			? String(inputUbicacion)
					.split(',')
					.map((s: string) => s.trim())
					.filter(Boolean)
			: [];
    const sistemaArray = Array.isArray(inputSistema)
			? inputSistema
			: inputSistema
			? String(inputSistema)
					.split(',')
					.map((s: string) => s.trim())
					.filter(Boolean)
			: [];
		const payload = {
			familia_rvt: categoriaSeleccionada, // Categoría en inglés (Revit)
			clave: clave,
			// Inputs are provided in cm in the UI. Convert to meters for the database.
			B: inputAncho === '' ? null : Number(inputAncho) / 100,
			H: inputAlto === '' ? null : Number(inputAlto) / 100,
			nivel_desplante: inputPosicion === '' ? null : Number(inputPosicion),
			edificio_modelo: inputOrigen,
			edificio_localiza: ubicacionArray,
      sistema: sistemaArray,
			referencia: inputReferencia,
			estado: 'Borrador'
		};

		console.debug('createFamilia payload:', payload);
		const { data, error } = await supabase.from('familiasrevit').insert(payload).select().single();
		if (error) {
			console.log('supabase insert error', error);
			setFeedbackMessage('Error al crear la familia: ' + error.message, 'error');
			return;
		}

		const newFamilia: Familia = {
			...data,
			B: data.B ? Number(data.B) : null,
			H: data.H ? Number(data.H) : null,
			L: data.L ? Number(data.L) : null,
			nivel_desplante: data.nivel_desplante ? Number(data.nivel_desplante) : null
		};
		familias.push(newFamilia);
		inputClave = '';
		inputAncho = '';
		inputAlto = '';
		inputPosicion = '';
		inputUbicacion = [];
    inputSistema = [];
		inputReferencia = '';
		// Reset create normalization mode after creation
		createModeNormalized = false;
		setFeedbackMessage('Familia creada exitosamente');
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
				B: f.B ? Number(f.B) : null,
				H: f.H ? Number(f.H) : null,
				L: f.L ? Number(f.L) : null,
				nivel_desplante: f.nivel_desplante ? Number(f.nivel_desplante) : null
			}));
		} catch (err) {
			console.error('Error fetching data:', err);
		}
	});

	function formatNumber(num: number | string | null | undefined): string {
		if (num === null || num === undefined || String(num).trim() === '') return '';
		const number = Number(num);
		const number_cm = number * 100;

		if (!isFinite(number_cm) || isNaN(number_cm)) {
			return '';
		}
		if (number_cm === 0) return '';

		if (number_cm % 1 === 0) {
			return number_cm.toFixed(0);
		}
		return number_cm.toFixed(1);
	}

	/**
	 * Formatea un valor de entrada que se considera en centímetros para comparar
	 * con la salida de formatNumber (que convierte metros->cm como string).
	 * Devuelve cadena vacía para valores nulos/vacíos.
	 */
	function formatCmInput(value: number | string | null | undefined): string {
		if (value === null || value === undefined || String(value).trim() === '') return '';
		const n = Number(value);
		if (!isFinite(n) || isNaN(n)) return '';
		if (n === 0) return '';
		// Si es entero en cm, mostrar sin decimales, si tiene fracción mostrar 1 decimal
		if (Math.abs(n % 1) < 1e-9) return n.toFixed(0);
		return n.toFixed(1);
	}

	function formatAsKey(value: string) {
		if (!value) return '';
		const normalized = value.normalize('NFD').replace(/[\u0300-\u036f]/g, '');
		const cleaned = normalized.replace(/[^0-9A-Za-z-_]+/g, ' ');
		return cleaned
			.trim()
			.split(/\s+/)
			.filter(Boolean)
			.map((w) => w.toUpperCase())
			.join('_')
			.replace(/_-/g, '-') // limpia posibles mezclas raras "_-"
			.replace(/-_/g, '-'); // idem
	}

	async function copyFamiliaToClipboard(familia: Familia) {
		const altoPart = formatNumber(familia.H) ? `x${formatNumber(familia.H)}cm` : 'cm';
		const label = `${familia.clave}-${formatNumber(familia.B)}${altoPart}`;
		try {
			if (navigator.clipboard && navigator.clipboard.writeText) {
				await navigator.clipboard.writeText(label);
			} else {
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
		selectedFamilia = familia;
		// Convertir B/H/L (que vienen en metros) a centímetros para mostrarse en el modal
		editingFamilia = {
			...familia,
			B:
				familia.B !== null && familia.B !== undefined
					? Number((Number(familia.B) * 100).toFixed(3))
					: '',
			H:
				familia.H !== null && familia.H !== undefined
					? Number((Number(familia.H) * 100).toFixed(3))
					: '',
			L:
				familia.L !== null && familia.L !== undefined
					? Number((Number(familia.L) * 100).toFixed(3))
					: '',
			edificio_localiza: Array.isArray(familia.edificio_localiza)
          ? [...familia.edificio_localiza]
          : (familia.edificio_localiza ? String(familia.edificio_localiza).split(',').map((s: string) => s.trim()).filter(Boolean) : []),
      sistema: familia.sistema?.join(', ') ?? ''
		};
		// Ensure edit normalization flag resets when opening modal
		editModeNormalized = false;
		showModal = true;
	}

	async function updateFamilia(normalize = false) {
		if (!editingFamilia || !editingFamilia.id) {
			setFeedbackMessage('Error: No hay familia seleccionada', 'error');
			return;
		}

		try {
			const allowedEstados = ['Borrador', 'En revision', 'Aprobada', 'Desactualizada'];
			if (editingFamilia.estado && !allowedEstados.includes(editingFamilia.estado)) {
				setFeedbackMessage(`Estado inválido: ${editingFamilia.estado}`, 'error');
				return;
			}

			const id = editingFamilia.id;

			// Verificar que el registro existe
			const { data: existing, error: checkError } = await supabase
				.from('familiasrevit')
				.select('id')
				.eq('id', id)
				.maybeSingle();

			if (checkError) {
				console.error('Error verificando registro:', checkError);
				setFeedbackMessage('Error al verificar el registro: ' + checkError.message, 'error');
				return;
			}

			if (!existing) {
				setFeedbackMessage('Error: El registro no existe en la base de datos', 'error');
				return;
			}

			// Preparar payload - solo incluir campos que no sean null/undefined
			const payload: any = {
				// Only normalize if requested (when user clicked the Normalize button or submits via the normalized action)
				clave: normalize
					? formatAsKey(String(editingFamilia.clave ?? ''))
					: String(editingFamilia.clave ?? ''),
				referencia: editingFamilia.referencia || ''
			};

			// Agregar campos opcionales solo si tienen valor
			if (editingFamilia.familia_rvt !== null && editingFamilia.familia_rvt !== undefined) {
				payload.familia_rvt = editingFamilia.familia_rvt;
			}
			if (editingFamilia.tipo !== null && editingFamilia.tipo !== undefined) {
				payload.tipo = editingFamilia.tipo;
			}
			if (editingFamilia.B !== '' && editingFamilia.B !== null && editingFamilia.B !== undefined) {
				// editingFamilia.B is shown/edited in cm in the modal — convert to meters for DB
				payload.B = Number(editingFamilia.B) / 100;
			}
			if (editingFamilia.H !== '' && editingFamilia.H !== null && editingFamilia.H !== undefined) {
				payload.H = Number(editingFamilia.H) / 100;
			}
			if (editingFamilia.L !== '' && editingFamilia.L !== null && editingFamilia.L !== undefined) {
				payload.L = Number(editingFamilia.L) / 100;
			}
			if (
				editingFamilia.nivel_desplante !== '' &&
				editingFamilia.nivel_desplante !== null &&
				editingFamilia.nivel_desplante !== undefined
			) {
				payload.nivel_desplante = Number(editingFamilia.nivel_desplante);
			}
			if (editingFamilia.estado) {
				payload.estado = editingFamilia.estado;
			}
      if (editingFamilia.edificio_modelo) {
        payload.edificio_modelo = editingFamilia.edificio_modelo;
      }
			if (editingFamilia.edificio_localiza) {
				payload.edificio_localiza = editingFamilia.edificio_localiza;
			}
			if (
				editingFamilia.edificio_localiza &&
				typeof editingFamilia.edificio_localiza === 'string' &&
				editingFamilia.edificio_localiza.trim()
			) {
				payload.edificio_localiza = editingFamilia.edificio_localiza
					.split(',')
					.map((s: string) => s.trim());
			}
			if (editingFamilia.parametros) {
				payload.parametros = editingFamilia.parametros;
			}
			if (editingFamilia.revisor) {
				payload.revisor = editingFamilia.revisor;
			}
			if (editingFamilia.categoria) {
				payload.categoria = editingFamilia.categoria;
			}
			if (editingFamilia.nivel) {
				payload.nivel = editingFamilia.nivel;
			}
			if (editingFamilia.clave) {
				payload.clave = editingFamilia.clave;
			}
			if (editingFamilia.disciplina) {
				payload.disciplina = editingFamilia.disciplina;
			}
			if (editingFamilia.propiedades) {
				payload.propiedades = editingFamilia.propiedades;
			}
			if (editingFamilia.especificacion) {
				payload.especificacion = editingFamilia.especificacion;
			}
			if (
				editingFamilia.sistema &&
				typeof editingFamilia.sistema === 'string' &&
				editingFamilia.sistema.trim()
			) {
				payload.sistema = editingFamilia.sistema.split(',').map((s: string) => s.trim());
			}
			if (editingFamilia.revisiones) {
				payload.revisiones = editingFamilia.revisiones;
			}

			console.debug('updateFamilia - ID:', id);
			console.debug('updateFamilia - payload:', payload);

			const { data, error } = await supabase
				.from('familiasrevit')
				.update(payload)
				.eq('id', id)
				.select()
				.single();

			if (error) {
				console.error('supabase update error:', error);
				console.error('error details:', JSON.stringify(error, null, 2));
				setFeedbackMessage('Error al actualizar la familia: ' + error.message, 'error');
				return;
			}

			if (!data) {
				setFeedbackMessage('Error: No se recibieron datos después de la actualización', 'error');
				return;
			}

			const updatedFamilia: Familia = {
				...data,
				B: data.B ? Number(data.B) : null,
				H: data.H ? Number(data.H) : null,
				L: data.L ? Number(data.L) : null,
				nivel_desplante: data.nivel_desplante ? Number(data.nivel_desplante) : null
			};

			const index = familias.findIndex((f) => f.id === id);
			if (index !== -1) {
				familias[index] = updatedFamilia;
			}
			// Reset edit normalization flag after successful update and close modal
			editModeNormalized = false;
			showModal = false;
			setFeedbackMessage('Familia actualizada exitosamente');
		} catch (error) {
			console.error('Error al actualizar:', error);
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
			familias = familias.filter((f) => f.id !== id);
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

<div class="flex min-h-screen flex-col">
	<div class="container mx-auto flex-grow px-4">
		<div id="formCreacionFamilia" class="top-0 right-0 left-0 z-30 bg-base-100 shadow">
			<div class="container mx-auto p-4">
				<div class="mb-2 flex flex-wrap gap-4">
					<div>
						<label class="input" for="inputCategoria">
              <span class="label">
                Categoría Revit
              </span>
              <input
                id="inputCategoria"
                type="text"
                bind:value={categoriaSeleccionada}
                class="input-bordered input input-sm w-full max-w-xs"
                disabled
              />
            </label>
					</div>
					<div class="form-control">
						<label class="input" for="inputClave">
              <span class="label">
                Clave
              </span>
              <input
                id="inputClave"
                type="text"
                bind:value={inputClave}
                class="input-bordered input input-sm w-full max-w-xs"
              />
            </label>
					</div>
					<div class="form-control">
						<label class="input" for="inputAncho">
              <span class="label">
                B (cm)
              </span>
              <input
                id="inputAncho"
                type="number"
                step="0.1"
                placeholder="Ancho"
                bind:value={inputAncho}
                class="input-bordered input input-sm w-full max-w-xs"
              />
            </label>
					</div>
					<div class="form-control">
						<label class="input" for="inputAlto">
            <span class="label">
              H (cm)
            </span>
						<input
              id="inputAlto"
              type="number"
              step="0.1"
              placeholder="Alto"
              bind:value={inputAlto}
              class="input-bordered input input-sm w-full max-w-xs"
            />
            </label>
					</div>
          <div class="form-control">
						<label class="input" for="inputSistema">
              <span class="label">
                Sistemas
              </span>
              <input
                id="inputSistema"
                type="text"
                placeholder="Sistemas separados por comas"
                bind:value={inputSistema}
                class="input-bordered input input-sm w-full max-w-xs"
              />
            </label>
					</div>
					<!-- <div class="form-control">
						<label class="input" for="inputPosicion">
              <span class="label">
                Nivel Desplante (en metros)
              </span>
              <input
                id="inputPosicion"
                type="number"
                step="any"
                placeholder="Posicion"
                bind:value={inputPosicion}
                class="input-bordered input input-sm w-full max-w-xs"
              />
            </label>
					</div> -->
          <div class="form-control">
						<label class="input" for="inputOrigen">
              <span class="label">
                Archivo origen
              </span>
              <input
                id="inputOrigen"
                type="text"
                placeholder="Origen"
                bind:value={inputOrigen}
                class="input-bordered input input-sm w-full max-w-xs"
              />
            </label>
					</div>
          <div class="form-control">
						<label class="input" for="inputReferencia">
              <span class="label">
                Referencia
              </span>
              <input
                id="inputReferencia"
                type="text"
                placeholder="Referencia"
                bind:value={inputReferencia}
                class="input-bordered input input-sm w-full max-w-xs"
              />
            </label>
					</div>
					<div class="form-control">
						<label class="label w-1/3 text-left" for="inputUbicacion">Ubicación</label>
						<div class="flex flex-wrap gap-2 flex-1">
							{#each ['A','B','C','D','E','V','Casetas','PTAR','Obra exterior'] as opt}
								<label class="inline-flex items-center gap-2">
									<input
										type="checkbox"
										class="checkbox"
										checked={Array.isArray(inputUbicacion) ? inputUbicacion.includes(opt) : false}
										onchange={(e) => {
											const checked = (e.currentTarget as HTMLInputElement).checked;
											if (!Array.isArray(inputUbicacion)) inputUbicacion = [];
											if (checked) {
												if (!inputUbicacion.includes(opt)) inputUbicacion = [...inputUbicacion, opt];
											} else {
												inputUbicacion = inputUbicacion.filter((v: any) => v !== opt);
											}
										}}
									/>
									<span>{opt}</span>
								</label>
							{/each}
						</div>
					</div>
					<div class="flex items-end gap-2">
						<button
							type="button"
							class="btn"
							onclick={() => {
								inputClave = formatAsKey(String(inputClave ?? ''));
								createModeNormalized = true;
							}}
							disabled={isCrearButtonDisabled}>Normalizar</button
						>
						<button
							type="button"
							class={createModeNormalized ? 'btn' : 'btn btn-warning'}
							onclick={() => createFamilia(createModeNormalized)}
							disabled={isCrearButtonDisabled}
							>{createModeNormalized ? 'Crear' : 'Crear sin normalizar'}</button
						>
					</div>
				</div>
				<div class="mt-6 mb-2 flex flex-wrap gap-2">
					{#each categorias as categoria}
						<button
							class="btn btn-sm"
							class:btn-primary={categoriaSeleccionada === categoria}
							class:btn-secundary={categoriaSeleccionada !== categoria}
							onclick={() => (categoriaSeleccionada = categoriaSeleccionada === categoria ? '' : categoria)}
						>
							{categoria}
						</button>
					{/each}
				</div>
			</div>
		</div>

    <div class="divider divider-info">&uarr; Rellena para buscar o registrar familias de Revit | Normaliza para eliminar espacios y símbolos (para evitar problemas por duplicados) &uarr;</div>

		<main class="z-999 mb-20">
			<div class="mb-4 max-h-[calc(100vh-6rem-4rem)] overflow-auto">
				<table class="table w-full">
					<thead>
						<tr>
							<th class="top-0 z-10 w-1/4 bg-base-200 text-left"
								>Nombre sugerido (Click para copiar)</th
							>
              <th class="top-0 z-10 w-12 bg-base-200 text-left">Clave</th>
              <th class="top-0 z-10 w-12 bg-base-200 text-left">Tipo Familia</th>
							<th class="top-0 z-10 w-12 bg-base-200 text-left">B (cm)</th>
							<th class="top-0 z-10 w-12 bg-base-200 text-left">H (cm)</th>
							<th class="top-0 z-10 w-12 bg-base-200 text-left">N. desplante <br/> (METROS)</th>
							<th class="top-0 z-10 w-1/8 bg-base-200 text-left">Estado</th>
							<th class="top-0 z-10 w-24 bg-base-200 text-left">Origen</th>
							<th class="top-0 z-10 w-1/8 bg-base-200 text-left">Ubicación</th>
							<th class="top-0 z-10 w-1/8 bg-base-200 text-left">Parámetros</th>
              <th class="top-0 z-10 w-1/8 bg-base-200 text-left">Sistema</th>
							<th class="top-0 z-10 w-1/8 bg-base-200 text-left">Referencia</th>
							<th class="top-0 z-10 w-24 bg-base-200 text-left">Revisor</th>
							<th class="top-0 z-10 w-1/8 bg-base-200 text-left">Acciones</th>
						</tr>
					</thead>
					<tbody>
						{#each familiasFiltradas as familia}
							<tr>
								<td
									class="cursor-pointer"
									title="Click para copiar"
									onclick={() => copyFamiliaToClipboard(familia)}
									>{familia.clave}-{formatNumber(familia.B)}{formatNumber(familia.H)
										? `x${formatNumber(familia.H)}cm`
										: `cm`}</td
								>
                <td>{familia.clave}</td>
                <td>{familia.familia_rvt}</td>
								<td>{formatNumber(familia.B)}</td>
								<td>{formatNumber(familia.H)}</td>
								<td>{familia.nivel_desplante}</td>
								<td>{familia.estado}</td>
								<td>{familia.edificio_modelo}</td>
								<td>{familia.edificio_localiza?.join(', ')}</td>
								<td>{familia.parametros}</td>
                <td>{familia.sistema}</td>
								<td>{familia.referencia}</td>
								<td>{familia.revisor}</td>
								<td>
									<button
										class="btn btn-ghost btn-xs"
										onclick={() => openModal(familia)}
										aria-label="Actualizar"
									>
										<svg
											xmlns="http://www.w3.org/2000/svg"
											class="h-4 w-4"
											fill="none"
											viewBox="0 0 24 24"
											stroke="currentColor"
											><path
												stroke-linecap="round"
												stroke-linejoin="round"
												stroke-width="2"
												d="M15.232 5.232l3.536 3.536m-2.036-5.036a2.5 2.5 0 113.536 3.536L6.5 21.036H3v-3.5L14.732 3.732z"
											/></svg
										>
									</button>
									<button
										class="btn btn-ghost btn-xs"
										onclick={() => deleteFamilia(familia.id!)}
										aria-label="Eliminar"
									>
										<svg
											xmlns="http://www.w3.org/2000/svg"
											class="h-4 w-4"
											fill="none"
											viewBox="0 0 24 24"
											stroke="currentColor"
											><path
												stroke-linecap="round"
												stroke-linejoin="round"
												stroke-width="2"
												d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"
											/></svg
										>
									</button>
								</td>
							</tr>
						{/each}
					</tbody>
				</table>
			</div>
		</main>
	</div>

	{#if showModal && editingFamilia}
		<dialog class="modal-open modal">
			<div class="modal-box max-w-3xl">
				<h3 class="mb-4 text-lg font-bold">Editar Familia</h3>
				<form
					onsubmit={(e) => {
						e.preventDefault();
						updateFamilia(editModeNormalized);
					}}
				>
					<div class="space-y-4">
						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editClave">Clave</label>
							<input
								type="text"
								id="editClave"
								bind:value={editingFamilia.clave}
								class="input-bordered input flex-1"
							/>
						</div>

						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editFamiliaRvt">Categoría Revit</label>
							<select
								id="editFamiliaRvt"
								bind:value={editingFamilia.familia_rvt}
								class="select-bordered select flex-1"
							>
								{#each categorias as categoria}
									<option value={categoria}>{categoria}</option>
								{/each}
							</select>
						</div>

						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editTipo">Tipo (español)</label>
							<input
								type="text"
								id="editTipo"
								bind:value={editingFamilia.tipo}
								class="input-bordered input flex-1"
							/>
						</div>

						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editEstado">Estado</label>
							<select
								id="editEstado"
								bind:value={editingFamilia.estado}
								class="select-bordered select flex-1"
							>
								<option value="Borrador">Borrador</option>
								<option value="En revision">En revisión</option>
								<option value="Aprobada">Aprobada</option>
								<option value="Desactualizada">Desactualizada</option>
							</select>
						</div>

						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editAncho">B (cm)</label>
							<input
								type="number"
								step="0.1"
								id="editAncho"
								bind:value={editingFamilia.B}
								class="input-bordered input flex-1"
							/>
						</div>

						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editAlto">H (cm)</label>
							<input
								type="number"
								step="0.1"
								id="editAlto"
								bind:value={editingFamilia.H}
								class="input-bordered input flex-1"
							/>
						</div>

						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editLargo">L (cm)</label>
							<input
								type="number"
								step="0.1"
								id="editLargo"
								bind:value={editingFamilia.L}
								class="input-bordered input flex-1"
							/>
						</div>

						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editPosicion">Nivel Desplante</label>
							<input
								type="number"
								id="editPosicion"
								step="any"
								bind:value={editingFamilia.nivel_desplante}
								class="input-bordered input flex-1"
							/>
						</div>

            <div class="form-control flex flex-row items-center gap-4">
              <label class="label w-1/3 text-left" for="editEdificio">Edificio</label>
              <div class="flex flex-wrap gap-2 flex-1" id="opcionesActualizarEdificio">
                {#each ['A','B','C','D','E','V','PTAR','Casetas','Obra exterior'] as opt}
                  <label class="inline-flex items-center gap-2">
                    <input
                      type="checkbox"
                      class="checkbox"
                      checked={Array.isArray(editingFamilia.edificio_localiza) ? editingFamilia.edificio_localiza.includes(opt) : false}
                      onchange={(e) => {
                        const checked = (e.currentTarget as HTMLInputElement).checked;
                        if (!Array.isArray(editingFamilia.edificio_localiza)) editingFamilia.edificio_localiza = [];
                        if (checked) {
                          if (!editingFamilia.edificio_localiza.includes(opt)) editingFamilia.edificio_localiza = [...editingFamilia.edificio_localiza, opt];
                        } else {
                          editingFamilia.edificio_localiza = editingFamilia.edificio_localiza.filter((v: any) => v !== opt);
                        }
                      }}
                    />
                    <span>{opt}</span>
                  </label>
                {/each}
              </div>
            </div>

						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editLocaliza"
								>Archivo origen</label
							>
							<input
								type="text"
								id="editLocaliza"
								bind:value={editingFamilia.edificio_modelo}
								class="input-bordered input flex-1"
							/>
						</div>

						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editParametros">Parámetros</label>
							<input
								type="text"
								id="editParametros"
								bind:value={editingFamilia.parametros}
								class="input-bordered input flex-1"
							/>
						</div>

						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editReferencia">Referencia</label>
							<input
								type="text"
								id="editReferencia"
								bind:value={editingFamilia.referencia}
								class="input-bordered input flex-1"
							/>
						</div>

						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editRevisor">Revisor</label>
							<input
								type="text"
								id="editRevisor"
								bind:value={editingFamilia.revisor}
								class="input-bordered input flex-1"
							/>
						</div>
					</div>

					<div class="modal-action">
						<button
							type="button"
							class="btn"
							onclick={() => {
								editingFamilia.clave = formatAsKey(String(editingFamilia.clave ?? ''));
								editModeNormalized = true;
							}}>Normalizar</button
						>
						<button
							type="button"
							class={editModeNormalized ? 'btn btn-primary' : 'btn btn-warning'}
							onclick={() => updateFamilia(editModeNormalized)}
							>{editModeNormalized ? 'Actualizar' : 'Actualizar sin normalizar'}</button
						>
						<button type="button" class="btn" onclick={() => (showModal = false)}>Cancelar</button>
					</div>
				</form>
			</div>
		</dialog>
	{/if}

	<footer
		class="footer-center fixed right-0 bottom-0 left-0 z-40 footer bg-base-300 p-4 text-base-content"
	>
		{#if feedbackMessage}
			<div
				class="alert"
				class:alert-success={!feedbackMessage.includes('Error')}
				class:alert-error={feedbackMessage.includes('Error')}
			>
				<span>{feedbackMessage}</span>
			</div>
		{/if}
		<div>
			<p>Nombres de familias para elementos en Revit - Proyecto Yecapixtla</p>
		</div>
		<div class="mt-4 flex">
			<span class="mr-2">Tema:</span>
			<select
				class="select-bordered select select-sm"
				onchange={(e) => document.documentElement.setAttribute('data-theme', e.currentTarget.value)}
			>
				<option value="cupcake">Claro</option>
				<option value="dracula">Oscuro</option>
			</select>
		</div>
	</footer>
</div>
