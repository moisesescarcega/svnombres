<script lang="ts">
	import { onMount } from 'svelte';
	import { supabase } from '$lib/supabase';

	// Constantes para opciones reutilizables
	const EDIFICIO_OPCIONES = ['A', 'B', 'C', 'D', 'E', 'V', 'Casetas', 'PTAR', 'Obra exterior'];
	const ESTADOS_PERMITIDOS = ['Borrador', 'En revision', 'Aprobada', 'Desactualizada'];

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

	// Estado principal
	let familias = $state<Familia[]>([]);
	let categoriaSeleccionada = $state('Window');
	
	// Formulario de creación
	let formState = $state({
		clave: '',
		ancho: '',
		alto: '',
		posicion: '',
		ubicaciones: [] as string[],
		origen: '',
		referencia: '',
		sistemas: [] as string[],
		claveNormalizada: false
	});

	const categorias = [
		'Door', 'Window', 'Curtain Wall', 'Wall', 'Floor', 'Roof', 'Ceiling', 'Column', 'Beam',
		'Plumbing Fixture', 'Mechanical Equipment', 'Air Terminal', 'Duct', 'Pipe',
		'Electrical Equipment', 'Lighting Fixture', 'Furniture', 'Generic Model', 'Railings',
		'Stairs', 'Ramp', 'Structural Foundation', 'Site Component', 'Landscaping',
		'Building Pad', 'In-Place Family', 'Equipment', 'Fire Protection', 'Security Device',
		'Structural Framing', 'Structural Rebar', 'Casework', 'Specialty Equipment'
	];

	// Estado para edición
	let editState = $state({
		visible: false,
		familia: null as Familia | null,
		claveNormalizada: false
	});

	let feedbackMessage = $state('');

	// Funciones de utilidad
	function formatNumber(num: number | string | null | undefined): string {
		if (num === null || num === undefined || String(num).trim() === '') return '';
		const number = Number(num);
		const number_cm = number * 100;
		if (!isFinite(number_cm) || isNaN(number_cm)) return '';
		if (number_cm === 0) return '';
		return number_cm % 1 === 0 ? number_cm.toFixed(0) : number_cm.toFixed(1);
	}

	function formatCmInput(value: number | string | null | undefined): string {
		if (value === null || value === undefined || String(value).trim() === '') return '';
		const n = Number(value);
		if (!isFinite(n) || isNaN(n)) return '';
		if (n === 0) return '';
		return Math.abs(n % 1) < 1e-9 ? n.toFixed(0) : n.toFixed(1);
	}

	function normalizeKey(value: string): string {
		if (!value) return '';
		return value
			.normalize('NFD')
			.replace(/[\u0300-\u036f]/g, '')
			.replace(/[^0-9A-Za-z-_]+/g, ' ')
			.trim()
			.split(/\s+/)
			.filter(Boolean)
			.map(w => w.toUpperCase())
			.join('_')
			.replace(/_-/g, '-')
			.replace(/-_/g, '-');
	}

	function parseArrayInput(value: string[] | string | null | undefined): string[] {
		if (Array.isArray(value)) return [...value];
		if (!value) return [];
		return String(value)
			.split(',')
			.map(s => s.trim())
			.filter(Boolean);
	}

	function setFeedbackMessage(message: string, isError = false) {
		feedbackMessage = message;
		setTimeout(() => {
			feedbackMessage = '';
		}, 3000);
	}

	// Lógica de filtros (optimizada con $derived)
	let familiasFiltradas = $derived(
		familias.filter(familia => {
			const categoriaMatch = !categoriaSeleccionada || String(categoriaSeleccionada).trim() === '' ||
				(familia.familia_rvt || '').toLowerCase() === String(categoriaSeleccionada).toLowerCase();
			
			const claveMatch = formState.clave
				? (String(familia.clave || '').toLowerCase().includes(String(formState.clave).toLowerCase().trim()))
				: true;
			
			const anchoFilterActive = formState.ancho !== null && String(formState.ancho).trim() !== '';
			const altoFilterActive = formState.alto !== null && String(formState.alto).trim() !== '';
			
			// Comparaciones de dimensiones
			const anchoMatch = anchoFilterActive
				? formatNumber(familia.B) === formatCmInput(formState.ancho)
				: true;
			
			const altoMatch = altoFilterActive
				? formatNumber(familia.H) === formatCmInput(formState.alto)
				: true;
			
			return categoriaMatch && claveMatch && anchoMatch && altoMatch;
		})
	);

	let isCrearButtonDisabled = $derived(
		!formState.clave ||
		formState.ancho === '' ||
		formState.ubicaciones.length === 0 ||
		(familiasFiltradas.length > 0)
	);

	// Operaciones con la base de datos
	async function createFamilia() {
		const clave = formState.claveNormalizada 
			? normalizeKey(formState.clave) 
			: formState.clave;
		
		const payload = {
			familia_rvt: categoriaSeleccionada,
			clave: clave || '',
			B: formState.ancho ? Number(formState.ancho) / 100 : null,
			H: formState.alto ? Number(formState.alto) / 100 : null,
			nivel_desplante: formState.posicion ? Number(formState.posicion) : null,
			edificio_modelo: formState.origen || null,
			edificio_localiza: formState.ubicaciones,
			sistema: formState.sistemas,
			referencia: formState.referencia || '',
			estado: 'Borrador'
		};

		try {
			const { data, error } = await supabase
				.from('familiasrevit')
				.insert(payload)
				.select()
				.single();
				
			if (error) throw error;
			
			const newFamilia: Familia = {
				...data,
				B: data.B ? Number(data.B) : null,
				H: data.H ? Number(data.H) : null,
				L: data.L ? Number(data.L) : null,
				nivel_desplante: data.nivel_desplante ? Number(data.nivel_desplante) : null
			};
			
			familias.push(newFamilia);
			
			// Resetear formulario
			formState = {
				clave: '',
				ancho: '',
				alto: '',
				posicion: '',
				ubicaciones: [],
				origen: '',
				referencia: '',
				sistemas: [],
				claveNormalizada: false
			};
			
			setFeedbackMessage('Familia creada exitosamente');
		} catch (error: any) {
			console.error('Error creando familia:', error);
			setFeedbackMessage('Error al crear la familia: ' + (error.message || error), true);
		}
	}

	async function loadFamilias() {
		try {
			const { data, error } = await supabase.from('familiasrevit').select('*');
			
			if (error) throw error;
			
			familias = data.map(f => ({
				...f,
				B: f.B ? Number(f.B) : null,
				H: f.H ? Number(f.H) : null,
				L: f.L ? Number(f.L) : null,
				nivel_desplante: f.nivel_desplante ? Number(f.nivel_desplante) : null
			}));
		} catch (error: any) {
			console.error('Error cargando familias:', error);
			setFeedbackMessage('Error al cargar las familias: ' + (error.message || error), true);
		}
	}

	async function copyFamiliaToClipboard(familia: Familia) {
		const altoPart = formatNumber(familia.H) ? `x${formatNumber(familia.H)}cm` : 'cm';
		const label = `${familia.clave}-${formatNumber(familia.B)}${altoPart}`;
		
		try {
			await navigator.clipboard.writeText(label);
			setFeedbackMessage(`Copiado al portapapeles: ${label}`);
		} catch (err) {
			console.error('Error copiando al portapapeles:', err);
			setFeedbackMessage('Error al copiar al portapapeles', true);
		}
	}

	function openEditModal(familia: Familia) {
		editState.familia = {
			...familia,
			// Convertir de metros a cm para la edición
			B: familia.B != null ? Number((familia.B * 100).toFixed(1)) : null,
			H: familia.H != null ? Number((familia.H * 100).toFixed(1)) : null,
			L: familia.L != null ? Number((familia.L * 100).toFixed(1)) : null,
			edificio_localiza: parseArrayInput(familia.edificio_localiza),
			sistema: familia.sistema || []
		};
		
		editState.claveNormalizada = false;
		editState.visible = true;
	}

	async function updateFamilia() {
		if (!editState.familia || !editState.familia.id) {
			setFeedbackMessage('Error: No hay familia seleccionada para actualizar', true);
			return;
		}
		
		if (editState.familia.estado && !ESTADOS_PERMITIDOS.includes(editState.familia.estado)) {
			setFeedbackMessage(`Estado inválido: ${editState.familia.estado}`, true);
			return;
		}
		
		try {
			const { error: checkError } = await supabase
				.from('familiasrevit')
				.select('id')
				.eq('id', editState.familia.id)
				.maybeSingle();
				
			if (checkError) throw checkError;
			
			// Preparar payload para actualización
			const payload: Partial<Familia> = {
				clave: editState.claveNormalizada
					? normalizeKey(String(editState.familia.clave || ''))
					: String(editState.familia.clave || ''),
				familia_rvt: editState.familia.familia_rvt || null,
				tipo: editState.familia.tipo || null,
				referencia: editState.familia.referencia || '',
				estado: editState.familia.estado || null,
				edificio_modelo: editState.familia.edificio_modelo || null,
				edificio_localiza: Array.isArray(editState.familia.edificio_localiza)
					? editState.familia.edificio_localiza
					: parseArrayInput(editState.familia.edificio_localiza),
				sistema: Array.isArray(editState.familia.sistema)
					? editState.familia.sistema
					: parseArrayInput(editState.familia.sistema)
			};
			
			// Añadir dimensiones si existen (convertir de cm a metros)
			if (editState.familia.B != null) {
				payload.B = Number(editState.familia.B) / 100;
			}
			if (editState.familia.H != null) {
				payload.H = Number(editState.familia.H) / 100;
			}
			if (editState.familia.L != null) {
				payload.L = Number(editState.familia.L) / 100;
			}
			if (editState.familia.nivel_desplante != null) {
				payload.nivel_desplante = Number(editState.familia.nivel_desplante);
			}
			
			const { data, error } = await supabase
				.from('familiasrevit')
				.update(payload)
				.eq('id', editState.familia.id)
				.select()
				.single();
				
			if (error) throw error;
			
			const updatedFamilia: Familia = {
				...data,
				B: data.B ? Number(data.B) : null,
				H: data.H ? Number(data.H) : null,
				L: data.L ? Number(data.L) : null,
				nivel_desplante: data.nivel_desplante ? Number(data.nivel_desplante) : null
			};
			
			const index = familias.findIndex(f => f.id === updatedFamilia.id);
			if (index !== -1) {
				familias[index] = updatedFamilia;
			}
			
			editState.visible = false;
			setFeedbackMessage('Familia actualizada exitosamente');
		} catch (error: any) {
			console.error('Error actualizando familia:', error);
			setFeedbackMessage('Error al actualizar la familia: ' + (error.message || error), true);
		}
	}

	async function deleteFamilia(id: number) {
		if (!confirm('¿Estás seguro de eliminar esta familia?')) return;
		
		try {
			const { error } = await supabase
				.from('familiasrevit')
				.delete()
				.eq('id', id);
				
			if (error) throw error;
			
			familias = familias.filter(f => f.id !== id);
			setFeedbackMessage('Familia eliminada exitosamente');
		} catch (error: any) {
			console.error('Error eliminando familia:', error);
			setFeedbackMessage('Error al eliminar la familia: ' + (error.message || error), true);
		}
	}

	// Lifecycle hooks
	onMount(loadFamilias);

	// Manejadores de eventos para el formulario
	function toggleUbicacion(ubicacion: string) {
		if (formState.ubicaciones.includes(ubicacion)) {
			formState.ubicaciones = formState.ubicaciones.filter(u => u !== ubicacion);
		} else {
			formState.ubicaciones = [...formState.ubicaciones, ubicacion];
		}
	}
	
	function toggleUbicacionEdicion(ubicacion: string) {
		if (!editState.familia) return;
		
		const ubicaciones = Array.isArray(editState.familia.edificio_localiza)
			? [...editState.familia.edificio_localiza]
			: [];
			
		if (ubicaciones.includes(ubicacion)) {
			ubicaciones.splice(ubicaciones.indexOf(ubicacion), 1);
		} else {
			ubicaciones.push(ubicacion);
		}
		
		editState.familia.edificio_localiza = ubicaciones;
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
								bind:value={formState.clave}
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
								bind:value={formState.ancho}
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
								bind:value={formState.alto}
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
								bind:value={formState.sistemas}
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
								bind:value={formState.origen}
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
								bind:value={formState.referencia}
								class="input-bordered input input-sm w-full max-w-xs"
							/>
						</label>
					</div>
					<div class="form-control">
						<label class="label w-1/3 text-left" for="inputUbicacion">Ubicación</label>
						<div class="flex flex-wrap gap-2 flex-1">
							{#each EDIFICIO_OPCIONES as opt}
								<label class="inline-flex items-center gap-2">
									<input
										type="checkbox"
										class="checkbox"
										checked={formState.ubicaciones.includes(opt)}
										onchange={() => toggleUbicacion(opt)}
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
								formState.clave = normalizeKey(formState.clave);
								formState.claveNormalizada = true;
							}}
							disabled={isCrearButtonDisabled}>Normalizar</button>
						<button
							type="button"
							class={formState.claveNormalizada ? 'btn' : 'btn btn-warning'}
							onclick={createFamilia}
							disabled={isCrearButtonDisabled}
						>
							{formState.claveNormalizada ? 'Crear' : 'Crear sin normalizar'}
						</button>
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
							<th class="top-0 z-10 w-1/4 bg-base-200 text-left">Nombre sugerido (Click para copiar)</th>
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
								>
									{familia.clave}-{formatNumber(familia.B)}{formatNumber(familia.H)
										? `x${formatNumber(familia.H)}cm`
										: `cm`}
								</td>
								<td>{familia.clave}</td>
								<td>{familia.familia_rvt}</td>
								<td>{formatNumber(familia.B)}</td>
								<td>{formatNumber(familia.H)}</td>
								<td>{familia.nivel_desplante}</td>
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
										onclick={() => openEditModal(familia)}
										aria-label="Actualizar"
									>
										<svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
											<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15.232 5.232l3.536 3.536m-2.036-5.036a2.5 2.5 0 113.536 3.536L6.5 21.036H3v-3.5L14.732 3.732z"/>
										</svg>
									</button>
									<button
										class="btn btn-ghost btn-xs"
										onclick={() => deleteFamilia(familia.id!)}
										aria-label="Eliminar"
									>
										<svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
											<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"/>
										</svg>
									</button>
								</td>
							</tr>
						{/each}
					</tbody>
				</table>
			</div>
		</main>
	</div>
	
	{#if editState.visible && editState.familia}
		<dialog class="modal-open modal">
			<div class="modal-box max-w-3xl">
				<h3 class="mb-4 text-lg font-bold">Editar Familia</h3>
				<form onsubmit={(e) => {
					e.preventDefault();
					updateFamilia();
				}}>
					<div class="space-y-4">
						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editClave">Clave</label>
							<input
								type="text"
								id="editClave"
								bind:value={editState.familia.clave}
								class="input-bordered input flex-1"
							/>
						</div>
						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editFamiliaRvt">Categoría Revit</label>
							<select
								id="editFamiliaRvt"
								bind:value={editState.familia.familia_rvt}
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
								bind:value={editState.familia.tipo}
								class="input-bordered input flex-1"
							/>
						</div>
						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editEstado">Estado</label>
							<select
								id="editEstado"
								bind:value={editState.familia.estado}
								class="select-bordered select flex-1"
							>
								{#each ESTADOS_PERMITIDOS as estado}
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
								bind:value={editState.familia.B}
								class="input-bordered input flex-1"
							/>
						</div>
						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editAlto">H (cm)</label>
							<input
								type="number"
								step="0.1"
								id="editAlto"
								bind:value={editState.familia.H}
								class="input-bordered input flex-1"
							/>
						</div>
						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editLargo">L (cm)</label>
							<input
								type="number"
								step="0.1"
								id="editLargo"
								bind:value={editState.familia.L}
								class="input-bordered input flex-1"
							/>
						</div>
						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editPosicion">Nivel Desplante</label>
							<input
								type="number"
								id="editPosicion"
								step="any"
								bind:value={editState.familia.nivel_desplante}
								class="input-bordered input flex-1"
							/>
						</div>
						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editEdificio">Edificio</label>
							<div class="flex flex-wrap gap-2 flex-1" id="opcionesActualizarEdificio">
								{#each EDIFICIO_OPCIONES as opt}
									<label class="inline-flex items-center gap-2">
										<input
											type="checkbox"
											class="checkbox"
											checked={Array.isArray(editState.familia.edificio_localiza) 
												? editState.familia.edificio_localiza.includes(opt) 
												: false}
											onchange={() => toggleUbicacionEdicion(opt)}
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
								bind:value={editState.familia.edificio_modelo}
								class="input-bordered input flex-1"
							/>
						</div>
						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editParametros">Parámetros</label>
							<input
								type="text"
								id="editParametros"
								bind:value={editState.familia.parametros}
								class="input-bordered input flex-1"
							/>
						</div>
						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editReferencia">Referencia</label>
							<input
								type="text"
								id="editReferencia"
								bind:value={editState.familia.referencia}
								class="input-bordered input flex-1"
							/>
						</div>
						<div class="form-control flex flex-row items-center gap-4">
							<label class="label w-1/3 text-left" for="editRevisor">Revisor</label>
							<input
								type="text"
								id="editRevisor"
								bind:value={editState.familia.revisor}
								class="input-bordered input flex-1"
							/>
						</div>
					</div>
					<div class="modal-action">
						<button
							type="button"
							class="btn"
							onclick={() => {
								if (editState.familia) {
									editState.familia.clave = normalizeKey(String(editState.familia.clave || ''));
								}
								editState.claveNormalizada = true;
							}}
						>Normalizar</button>
						<button
							type="submit"
							class={editState.claveNormalizada ? 'btn btn-primary' : 'btn btn-warning'}
						>
							{editState.claveNormalizada ? 'Actualizar' : 'Actualizar sin normalizar'}
						</button>
						<button type="button" class="btn" onclick={() => editState.visible = false}>Cancelar</button>
					</div>
				</form>
			</div>
		</dialog>
	{/if}
	
	<footer class="footer-center fixed right-0 bottom-0 left-0 z-40 footer bg-base-300 p-4 text-base-content">
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