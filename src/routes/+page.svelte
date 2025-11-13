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

	const ESTADOS_VALIDOS = ['Borrador', 'En revision', 'Aprobada', 'Desactualizada'] as const;
	const UBICACIONES_DISPONIBLES = ['A', 'B', 'C', 'D', 'E', 'V', 'Casetas', 'PTAR', 'Obra exterior'] as const;
	const CATEGORIAS = [
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
	] as const;

	let familias = $state<Familia[]>([]);
	let categoriaSeleccionada = $state('Window');
	let inputClave = $state('');
	let inputAncho = $state<number | ''>('');
	let inputAlto = $state<number | ''>('');
	let inputLargo = $state<number | ''>('');
	let inputUbicacion = $state<string[]>([]);
	let inputOrigen = $state('');
	let inputReferencia = $state('');
	let inputSistema = $state<string[]>([]);
	let showModal = $state(false);
	let selectedFamilia = $state<Familia | null>(null);
	let editingFamilia = $state<Partial<Familia> & { id?: number }>({});
	let createModeNormalized = $state(false);
	let editModeNormalized = $state(false);
	let feedbackMessage = $state('');

	// Utilidades de conversión
	const metrosToCm = (metros: number): number => metros * 100;
	const cmToMetros = (cm: number): number => cm / 100;

	const parseArrayFromString = (value: string | string[] | null | undefined): string[] => {
		if (Array.isArray(value)) return value;
		if (!value) return [];
		return String(value)
			.split(',')
			.map((s) => s.trim())
			.filter(Boolean);
	};

	let familiasFiltradas = $derived(
		familias.filter((familia) => {
			const categoriaMatch =
				!categoriaSeleccionada || String(categoriaSeleccionada).trim() === ''
					? true
					: (familia.familia_rvt || '').toLowerCase() ===
					  String(categoriaSeleccionada).toLowerCase();

			const claveMatch = inputClave
				? String(familia.clave || '')
						.toLowerCase()
						.includes(String(inputClave).toLowerCase().trim())
				: true;

			const anchoFilterActive = inputAncho !== '' && inputAncho != null;
			const altoFilterActive = inputAlto !== '' && inputAlto != null;
			const largoFilterActive = inputLargo !== '' && inputLargo != null;

			const anchoMatch = anchoFilterActive
				? formatDimensionFromMeters(familia.B) === formatDimensionInput(inputAncho)
				: true;

			const altoMatch = altoFilterActive
				? formatDimensionFromMeters(familia.H) === formatDimensionInput(inputAlto)
				: true;

			const largoMatch = largoFilterActive
				? formatDimensionFromMeters(familia.L) === formatDimensionInput(inputLargo)
				: true;

			// Ubicación: si no hay filtros seleccionados, aceptar todo.
			// Si hay filtros, aceptar familias cuyo edificio_localiza (array) contenga
			// al menos una de las ubicaciones seleccionadas.
			const ubicacionMatch =
				inputUbicacion == null || inputUbicacion.length === 0
					? true
					: (() => {
						const famUb = parseArrayFromString(familia.edificio_localiza as any);
						if (!famUb || famUb.length === 0) return false;
						return inputUbicacion.some((sel) => famUb.includes(sel));
					})();

			return (
				categoriaMatch &&
				claveMatch &&
				anchoMatch &&
				altoMatch &&
				largoMatch &&
				ubicacionMatch
			);
		})
	);

	let isCrearButtonDisabled = $derived(
		!inputClave ||
			inputAncho == null ||
			String(inputAncho).trim() === '' ||
			inputUbicacion.length === 0 ||
			(familiasFiltradas && familiasFiltradas.length > 0)
	);

	async function createFamilia(normalize = false) {
		const clave = normalize ? formatAsKey(String(inputClave)) : String(inputClave);
		const ubicacionArray = parseArrayFromString(inputUbicacion);
		const sistemaArray = parseArrayFromString(inputSistema);

		const payload = {
			familia_rvt: categoriaSeleccionada,
			clave: clave,
			B: inputAncho === '' ? null : cmToMetros(Number(inputAncho)),
			H: inputAlto === '' ? null : cmToMetros(Number(inputAlto)),
			L: inputLargo === '' ? null : cmToMetros(Number(inputLargo)),
			edificio_modelo: inputOrigen || null,
			edificio_localiza: ubicacionArray.length > 0 ? ubicacionArray : null,
			sistema: sistemaArray.length > 0 ? sistemaArray : null,
			referencia: inputReferencia || null,
			estado: 'Borrador'
		};

		const { data, error } = await supabase.from('familiasrevit').insert(payload).select().single();

		if (error) {
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
		resetCreateForm();
		setFeedbackMessage('Familia creada exitosamente');
	}

	function resetCreateForm() {
		inputClave = '';
		inputAncho = '';
		inputAlto = '';
		inputLargo = '';
		inputUbicacion = [];
		inputSistema = [];
		inputReferencia = '';
		inputOrigen = '';
		createModeNormalized = false;
	}

	onMount(async () => {
		const { data, error } = await supabase.from('familiasrevit').select('*');

		if (error) {
			setFeedbackMessage('Error al cargar las familias: ' + error.message, 'error');
			return;
		}

		// Normalize certain fields so the UI can rely on predictable types
		familias = data.map((f) => ({
			...f,
			B: f.B ? Number(f.B) : null,
			H: f.H ? Number(f.H) : null,
			L: f.L ? Number(f.L) : null,
			nivel_desplante: f.nivel_desplante ? Number(f.nivel_desplante) : null,
			// Ensure edificio_localiza and sistema are arrays (the DB column is a text[]; sometimes SDK returns string)
			edificio_localiza: parseArrayFromString(f.edificio_localiza),
			sistema: parseArrayFromString(f.sistema)
		}));
	});

	function formatDimensionFromMeters(valueInMeters: number | string | null | undefined): string {
		if (valueInMeters === null || valueInMeters === undefined || String(valueInMeters).trim() === '')
			return '';

		const valueCm = metrosToCm(Number(valueInMeters));

		if (!isFinite(valueCm) || isNaN(valueCm) || valueCm === 0) return '';

		return valueCm % 1 === 0 ? valueCm.toFixed(0) : valueCm.toFixed(1);
	}

	function formatDimensionInput(value: number | string | null | undefined): string {
		if (value === null || value === undefined || String(value).trim() === '') return '';

		const n = Number(value);
		if (!isFinite(n) || isNaN(n) || n === 0) return '';

		return Math.abs(n % 1) < 1e-9 ? n.toFixed(0) : n.toFixed(1);
	}

	function formatAsKey(value: string): string {
		if (!value) return '';

		const normalized = value.normalize('NFD').replace(/[\u0300-\u036f]/g, '');
		const cleaned = normalized.replace(/[^0-9A-Za-z-_]+/g, ' ');

		return cleaned
			.trim()
			.split(/\s+/)
			.filter(Boolean)
			.map((w) => w.toUpperCase())
			.join('_')
			.replace(/[_-]{2,}/g, '_')
			.replace(/^[_-]+|[_-]+$/g, '');
	}

	async function copyFamiliaToClipboard(familia: Familia) {
		const altoPart = formatDimensionFromMeters(familia.H)
			? `x${formatDimensionFromMeters(familia.H)}cm`
			: 'cm';
		const label = `${familia.clave}-${formatDimensionFromMeters(familia.B)}${altoPart}`;

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
		} catch {
			setFeedbackMessage('Error al copiar al portapapeles', 'error');
		}
	}

	function openModal(familia: Familia) {
		selectedFamilia = familia;
		editingFamilia = {
			...familia,
			B: familia.B !== null && familia.B !== undefined ? Number(metrosToCm(Number(familia.B)).toFixed(3)) : null,
			H: familia.H !== null && familia.H !== undefined ? Number(metrosToCm(Number(familia.H)).toFixed(3)) : null,
			L: familia.L !== null && familia.L !== undefined ? Number(metrosToCm(Number(familia.L)).toFixed(3)) : null,
			edificio_localiza: parseArrayFromString(familia.edificio_localiza),
			sistema: parseArrayFromString(familia.sistema)
		};
		editModeNormalized = false;
		showModal = true;
	}

	async function updateFamilia(normalize = false) {
		if (!editingFamilia || !editingFamilia.id) {
			setFeedbackMessage('Error: No hay familia seleccionada', 'error');
			return;
		}

		if (editingFamilia.estado && !ESTADOS_VALIDOS.includes(editingFamilia.estado as any)) {
			setFeedbackMessage(`Estado inválido: ${editingFamilia.estado}`, 'error');
			return;
		}

		const id = editingFamilia.id;

		const { data: existing, error: checkError } = await supabase
			.from('familiasrevit')
			.select('id')
			.eq('id', id)
			.maybeSingle();

		if (checkError) {
			setFeedbackMessage('Error al verificar el registro: ' + checkError.message, 'error');
			return;
		}

		if (!existing) {
			setFeedbackMessage('Error: El registro no existe en la base de datos', 'error');
			return;
		}

		const payload: Record<string, any> = {
			clave: normalize ? formatAsKey(String(editingFamilia.clave ?? '')) : String(editingFamilia.clave ?? ''),
			referencia: editingFamilia.referencia || ''
		};

		// Campos opcionales
		if (editingFamilia.familia_rvt) payload.familia_rvt = editingFamilia.familia_rvt;
		if (editingFamilia.tipo) payload.tipo = editingFamilia.tipo;
		if (editingFamilia.estado) payload.estado = editingFamilia.estado;
		if (editingFamilia.edificio_modelo) payload.edificio_modelo = editingFamilia.edificio_modelo;
		if (editingFamilia.parametros) payload.parametros = editingFamilia.parametros;
		if (editingFamilia.revisor) payload.revisor = editingFamilia.revisor;
		if (editingFamilia.categoria) payload.categoria = editingFamilia.categoria;
		if (editingFamilia.nivel) payload.nivel = editingFamilia.nivel;
		if (editingFamilia.disciplina) payload.disciplina = editingFamilia.disciplina;
		if (editingFamilia.propiedades) payload.propiedades = editingFamilia.propiedades;
		if (editingFamilia.especificacion) payload.especificacion = editingFamilia.especificacion;
		if (editingFamilia.revisiones) payload.revisiones = editingFamilia.revisiones;

		// Dimensiones (convertir de cm a metros)
		if (editingFamilia.B != null) {
			payload.B = cmToMetros(Number(editingFamilia.B));
		}
		if (editingFamilia.H != null) {
			payload.H = cmToMetros(Number(editingFamilia.H));
		}
		if (editingFamilia.L != null) {
			payload.L = cmToMetros(Number(editingFamilia.L));
		}
		if (editingFamilia.nivel_desplante != null) {
			payload.nivel_desplante = Number(editingFamilia.nivel_desplante);
		}

		// Arrays
		if (editingFamilia.edificio_localiza) {
			payload.edificio_localiza = Array.isArray(editingFamilia.edificio_localiza)
				? editingFamilia.edificio_localiza
				: parseArrayFromString(editingFamilia.edificio_localiza);
		}

		if (editingFamilia.sistema && typeof editingFamilia.sistema === 'string' && (editingFamilia.sistema as string).trim()) {
			payload.sistema = parseArrayFromString(editingFamilia.sistema as string);
		}

		const { data, error } = await supabase
			.from('familiasrevit')
			.update(payload)
			.eq('id', id)
			.select()
			.single();

		if (error) {
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

		editModeNormalized = false;
		showModal = false;
		setFeedbackMessage('Familia actualizada exitosamente');
	}

	async function deleteFamilia(id: number) {
		if (!confirm('¿Estás seguro de eliminar esta familia?')) return;

		const { error } = await supabase.from('familiasrevit').delete().eq('id', id);

		if (error) {
			setFeedbackMessage('Error al eliminar la familia: ' + error.message, 'error');
			return;
		}

		familias = familias.filter((f) => f.id !== id);
		setFeedbackMessage('Familia eliminada exitosamente');
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
							<span class="label">Categoría Revit</span>
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
							<span class="label">Clave</span>
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
							<span class="label">B (cm)</span>
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
							<span class="label">H (cm)</span>
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
							<span class="label">Sistemas</span>
							<input
								id="inputSistema"
								type="text"
								placeholder="Sistemas separados por comas"
								bind:value={inputSistema}
								class="input-bordered input input-sm w-full max-w-xs"
							/>
						</label>
					</div>
					<div class="form-control">
						<label class="input" for="inputOrigen">
							<span class="label">Archivo origen</span>
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
							<span class="label">Referencia</span>
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
						<div class="flex flex-wrap gap-2 flex-1" id="filtroUbicacion">
							{#each UBICACIONES_DISPONIBLES as opt}
								<label class="inline-flex items-center gap-2">
									<input
										type="checkbox"
										class="checkbox"
										checked={inputUbicacion.includes(opt)}
										onchange={(e) => {
											const checked = (e.currentTarget as HTMLInputElement).checked;
											if (checked) {
												if (!inputUbicacion.includes(opt)) inputUbicacion = [...inputUbicacion, opt];
											} else {
												inputUbicacion = inputUbicacion.filter((v) => v !== opt);
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
								inputClave = formatAsKey(String(inputClave));
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
					{#each CATEGORIAS as categoria}
						<button
							class="btn btn-sm"
							class:btn-primary={categoriaSeleccionada === categoria}
							class:btn-secundary={categoriaSeleccionada !== categoria}
							onclick={() =>
								(categoriaSeleccionada = categoriaSeleccionada === categoria ? '' : categoria)}
						>
							{categoria}
						</button>
					{/each}
				</div>
			</div>
		</div>

		<div class="divider divider-info">
			&uarr; Rellena para buscar o registrar familias de Revit | Normaliza para eliminar espacios
			y símbolos (para evitar problemas por duplicados) &uarr;
		</div>

		<main class="z-999 mb-20">
			<div class="mb-4 max-h-[calc(100vh-6rem-4rem)] overflow-auto">
				<table class="table w-full">
					<thead>
						<tr>
							<th class="top-0 z-10 w-1/4 bg-base-200 text-left"
								>Nombre sugerido</th
							>
							<th class="top-0 z-10 w-12 bg-base-200 text-left">Nombre actual</th>
							<th class="top-0 z-10 w-12 bg-base-200 text-left">Clave</th>
							<th class="top-0 z-10 w-12 bg-base-200 text-left">Tipo Familia</th>
							<th class="top-0 z-10 w-12 bg-base-200 text-left">B (cm)</th>
							<th class="top-0 z-10 w-12 bg-base-200 text-left">H (cm)</th>
							<th class="top-0 z-10 w-12 bg-base-200 text-left">L (cm)</th>
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
									>{familia.clave}-{formatDimensionFromMeters(familia.B)}{formatDimensionFromMeters(
										familia.H
									)
										? `x${formatDimensionFromMeters(familia.H)}cm`
										: `cm`}</td
								>
								<td>{familia.nombre}</td>
								<td>{familia.clave}</td>
								<td>{familia.familia_rvt}</td>
								<td>{formatDimensionFromMeters(familia.B)}</td>
								<td>{formatDimensionFromMeters(familia.H)}</td>
								<td>{formatDimensionFromMeters(familia.L)}</td>
								<td>{familia.estado}</td>
								<td>{familia.edificio_modelo}</td>
								<td>{familia.edificio_localiza?.join(', ')}</td>
								<td>{familia.parametros}</td>
								<td>{familia.sistema?.join(', ')}</td>
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
							<label class="label w-1/3 text-left" for="editNombre">Nombre</label>
							<input
								type="text"
								id="editNombre"
								bind:value={editingFamilia.nombre}
								class="input-bordered input flex-1"
							/>
						</div>

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
								{#each CATEGORIAS as categoria}
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
								{#each ESTADOS_VALIDOS as estado}
									<option value={estado}>{estado}</option>
								{/each}
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
							<label class="label w-1/3 text-left" for="editEdificio">Ubicación</label>
							<div class="flex flex-wrap gap-2 flex-1" id="opcionesActualizarEdificio">
								{#each UBICACIONES_DISPONIBLES as opt}
									<label class="inline-flex items-center gap-2">
										<input
											type="checkbox"
											class="checkbox"
											checked={Array.isArray(editingFamilia.edificio_localiza)
												? editingFamilia.edificio_localiza.includes(opt)
												: false}
											onchange={(e) => {
												const checked = (e.currentTarget as HTMLInputElement).checked;
												if (!Array.isArray(editingFamilia.edificio_localiza))
													editingFamilia.edificio_localiza = [];
												if (checked) {
													if (!editingFamilia.edificio_localiza.includes(opt))
														editingFamilia.edificio_localiza = [
															...editingFamilia.edificio_localiza,
															opt
														];
												} else {
													editingFamilia.edificio_localiza =
														editingFamilia.edificio_localiza.filter((v) => v !== opt);
												}
											}}
										/>
										<span>{opt}</span>
									</label>
								{/each}
							</div>
						</div>

						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editLocaliza">Archivo origen</label>
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
							<label class="label w-1/3 text-left" for="editSistema">Sistema</label>
							<input
								type="text"
								id="editSistema"
								bind:value={editingFamilia.sistema}
								class="input-bordered input flex-1"
								placeholder="Separados por comas"
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
								if (editingFamilia.clave) {
									editingFamilia.clave = formatAsKey(String(editingFamilia.clave));
								}
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
				<option value="forest">Oscuro</option>
			</select>
		</div>
	</footer>
</div>