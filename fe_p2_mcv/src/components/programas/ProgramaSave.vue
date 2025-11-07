<script setup lang="ts">
import type { Programa } from '../../models/programa'
import http from '../../plugins/axios'
import Button from 'primevue/button'
import Dialog from 'primevue/dialog'
import InputText from 'primevue/inputtext'
import InputNumber from 'primevue/inputnumber'
import DatePicker from 'primevue/datepicker'
import Textarea from 'primevue/textarea'
import { computed, ref, watch, onMounted } from 'vue'
import Select from 'primevue/select'

const ENDPOINT = 'programas'
const ENDPOINT_NIVELES = 'niveles-academicos'
const props = defineProps({
  mostrar: Boolean,
  programa: {
    type: Object as () => Programa,
    default: () => ({}) as Programa
  },
  modoEdicion: Boolean
})
const emit = defineEmits(['guardar', 'close'])

const dialogVisible = computed({
  get: () => props.mostrar,
  set: (value) => {
    if (!value) emit('close')
  }
})

const estadosDisponibles = [
  { label: 'En Planificación', value: 'En Planificación' },
  { label: 'En curso', value: 'En curso' },
  { label: 'Finalizado', value: 'Finalizado' }
]

const modalidadesDisponibles = [
  { label: 'Presencial', value: 'Presencial' },
  { label: 'Virtual', value: 'Virtual' },
  { label: 'Mixto', value: 'Mixto' }
]

const programa = ref<Programa>({ ...props.programa })
const nivelesAcademicos = ref<any[]>([])
const fechaInicioDate = ref<Date | null>(null)

watch(
  () => props.programa,
  (newVal) => {
    if (newVal) {
      programa.value = { ...newVal }
      // Convertir la fecha a Date si es string
      if (newVal.fechaInicio) {
        fechaInicioDate.value =
          typeof newVal.fechaInicio === 'string' ? new Date(newVal.fechaInicio) : newVal.fechaInicio
      } else {
        fechaInicioDate.value = null
      }
    }
  },
  { immediate: true }
)

watch(
  () => props.mostrar,
  (newVal) => {
    if (newVal && !props.modoEdicion) {
      // Resetear el formulario cuando se abre en modo creación
      programa.value = {} as Programa
      fechaInicioDate.value = null
    }
  }
)

async function fetchNivelesAcademicos() {
  try {
    const response = await http.get(ENDPOINT_NIVELES)
    nivelesAcademicos.value = response.data
  } catch (error) {
    console.error('Error fetching niveles académicos:', error)
    nivelesAcademicos.value = []
  }
}

onMounted(() => {
  fetchNivelesAcademicos()
})

async function handleSave() {
  try {
    // Convertir la fecha al formato correcto para el backend
    let fechaInicioFormatted = fechaInicioDate.value
    if (fechaInicioDate.value) {
      const date = new Date(fechaInicioDate.value)
      fechaInicioFormatted = date.toISOString().split('T')[0] as any
    }

    const body = {
      idNivelAcademico: programa.value.idNivelAcademico,
      nombre: programa.value.nombre,
      descripcion: programa.value.descripcion,
      modalidadClases: programa.value.modalidadClases,
      version: programa.value.version,
      duracionMeses: programa.value.duracionMeses,
      costo: programa.value.costo,
      fechaInicio: fechaInicioFormatted,
      estado: programa.value.estado
    }
    if (props.modoEdicion) {
      await http.patch(`${ENDPOINT}/${programa.value.id}`, body)
    } else {
      await http.post(ENDPOINT, body)
    }
    emit('guardar')
    programa.value = {} as Programa
    fechaInicioDate.value = null
    dialogVisible.value = false
  } catch (error: any) {
    alert(error?.response?.data?.message)
  }
}
</script>

<template>
  <div class="card flex justify-center">
    <Dialog
      v-model:visible="dialogVisible"
      :header="props.modoEdicion ? 'Editar Programa' : 'Crear Programa'"
      style="width: 40rem"
    >
      <!-- Nuevo orden: Nombre, Nivel Académico, Descripción, Versión, Duración, Costo, Fecha de Inicio, Estado -->
      <div class="mb-4">
        <label for="nombre" class="font-semibold block mb-1">Nombre</label>
        <InputText id="nombre" v-model="programa.nombre" class="w-full" autocomplete="off" />
      </div>

      <div class="mb-4">
        <label for="idNivelAcademico" class="font-semibold block mb-1">Nivel Académico</label>
        <Select
          id="idNivelAcademico"
          v-model="programa.idNivelAcademico"
          :options="nivelesAcademicos"
          optionLabel="nombre"
          optionValue="id"
          class="w-full"
        />
      </div>

      <div class="mb-4">
        <label for="descripcion" class="font-semibold block mb-1">Descripción</label>
        <Textarea id="descripcion" v-model="programa.descripcion" rows="3" class="w-full" autocomplete="off" />
      </div>

      <div class="mb-4">
        <label for="version" class="font-semibold block mb-1">Versión</label>
        <InputNumber id="version" v-model="programa.version" class="w-full" />
      </div>

      <div class="mb-4">
        <label for="duracionMeses" class="font-semibold block mb-1">Duración (meses)</label>
        <InputNumber id="duracionMeses" v-model="programa.duracionMeses" class="w-full" />
      </div>

      <div class="mb-4">
        <label for="costo" class="font-semibold block mb-1">Costo</label>
        <InputNumber
          id="costo"
          v-model="programa.costo"
          :minFractionDigits="2"
          :maxFractionDigits="2"
          class="w-full"
        />
      </div>

      <div class="mb-4">
        <label for="fechaInicio" class="font-semibold block mb-1">Fecha de Inicio</label>
        <DatePicker id="fechaInicio" v-model="fechaInicioDate" class="w-full" :showIcon="true" dateFormat="yy-mm-dd" />
      </div>

      <div class="mb-4">
        <label for="estado" class="font-semibold block mb-1">Estado</label>
        <Select id="estado" v-model="programa.estado" :options="estadosDisponibles" optionLabel="label" optionValue="value" class="w-full" />
      </div>

      <div class="mb-4">
        <label for="modalidadClases" class="font-semibold block mb-1">Modalidad de Clases</label>
        <Select
          id="modalidadClases"
          v-model="programa.modalidadClases"
          :options="modalidadesDisponibles"
          optionLabel="label"
          optionValue="value"
          class="w-full"
        />
      </div>
      <div class="flex justify-end gap-2">
        <Button
          type="button"
          label="Cancelar"
          icon="pi pi-times"
          severity="secondary"
          @click="dialogVisible = false"
        ></Button>
        <Button type="button" label="Guardar" icon="pi pi-save" @click="handleSave"></Button>
      </div>
    </Dialog>
  </div>
</template>

<style scoped></style>
