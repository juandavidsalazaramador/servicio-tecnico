<template>
  <q-layout view="lHh Lpr lFf" class="bg-grey-2">
    <q-header elevated class="bg-primary">
      <q-toolbar>
        <q-toolbar-title>
          <div class="text-weight-bold">🔧 TecnoFix</div>
          <div class="text-caption">Servicio técnico de celulares</div>
        </q-toolbar-title>
        <q-btn color="white" text-color="primary" icon="add"
          label="Nuevo servicio" @click="abrirFormulario" />
      </q-toolbar>
    </q-header>

    <q-page-container>
      <q-page class="q-pa-md">
        <!-- Resumen de Tarjetas -->
        <div class="row q-col-gutter-md q-mb-lg">
          <div v-for="dato in resumen" :key="dato.texto" class="col-12 col-sm-4">
            <q-card bordered>
              <q-card-section>
                <div class="text-subtitle1 text-grey-7">{{ dato.texto }}</div>
                <div :class="`text-h4 text-${dato.color}`">{{ dato.valor }}</div>
              </q-card-section>
            </q-card>
          </div>
        </div>

        <!-- Seccion de Busqueda y Filtros Rápidos -->
        <q-card class="q-mb-lg q-pa-md" bordered>
          <div class="row q-col-gutter-md items-center">
            <!-- Input de Búsqueda -->
            <div class="col-12 col-md-5">
              <q-input
                v-model="filtroTexto"
                placeholder="Buscar por cliente, marca o modelo..."
                outlined
                dense
                clearable
              >
                <template v-slot:prepend>
                  <q-icon name="search" />
                </template>
              </q-input>
            </div>

            <!-- Filtros por Estado del Equipo -->
            <div class="col-12 col-md-7 row items-center q-gutter-xs">
              <q-btn
                v-for="opcion in opcionesFiltroEstado"
                :key="opcion.valor"
                :label="opcion.etiqueta"
                :color="filtroEstado === opcion.valor ? 'primary' : 'grey-4'"
                :text-color="filtroEstado === opcion.valor ? 'white' : 'dark'"
                size="sm"
                unelevated
                @click="filtroEstado = opcion.valor"
              />

              <!-- Filtro rápido de Pendientes de Pago -->
              <q-btn
                label="Solo Pendientes Pago"
                :color="filtroPendientesPago ? 'negative' : 'grey-4'"
                :text-color="filtroPendientesPago ? 'white' : 'dark'"
                size="sm"
                unelevated
                icon="warning"
                @click="filtroPendientesPago = !filtroPendientesPago"
              />
            </div>
          </div>
        </q-card>

        <!-- Mensaje cuando no hay registros en absoluto -->
        <q-card v-if="!servicios.length" class="q-pa-xl text-center">
          <q-icon name="build" size="70px" color="grey-5" />
          <div class="text-h6 q-mt-md">No hay servicios registrados</div>
          <div class="text-grey-7 q-mb-md">
            Comienza registrando el primer equipo.
          </div>
          <q-btn color="primary" icon="add"
            label="Registrar servicio" @click="abrirFormulario" />
        </q-card>

        <!-- Mensaje cuando la búsqueda/filtro no devuelve resultados -->
        <q-card v-else-if="!serviciosFiltrados.length" class="q-pa-xl text-center">
          <q-icon name="search_off" size="60px" color="grey-5" />
          <div class="text-h6 q-mt-md">No se encontraron coincidencias</div>
          <div class="text-grey-7 q-mb-md">
            Intenta cambiar el texto de búsqueda o los filtros aplicados.
          </div>
          <q-btn outline color="primary" icon="clear" label="Limpiar filtros" @click="limpiarFiltros" />
        </q-card>

        <!-- Lista de Servicios Filtrados -->
        <div v-else class="row q-col-gutter-md">
          <div v-for="(servicio, index) in serviciosFiltrados" :key="servicio.id"
            class="col-12 col-md-6 col-lg-4">
            <q-card bordered
              :class="{ 'border-negative': servicio.estadoPago === 'Pendiente' }">

              <q-card-section class="row items-center">
                <div class="col">
                  <div class="text-h6">{{ servicio.marca }} {{ servicio.modelo }}</div>
                  <div class="text-subtitle1 text-grey-7">{{ servicio.cliente }}</div>
                </div>
                <q-icon :name="iconoEstado(servicio.estadoEquipo)"
                  size="32px" :color="colorEstado(servicio.estadoEquipo)" />
              </q-card-section>

              <q-separator />

              <q-card-section>
                <div class="q-mb-sm">
                  <q-icon name="build" color="primary" />
                  <strong> Reparación:</strong> 
                  {{ Array.isArray(servicio.reparacion) ? servicio.reparacion.join(', ') : servicio.reparacion }}
                </div>
                <div class="q-mb-sm">
                  <q-icon name="engineering" color="grey-7" />
                  <strong> Técnico:</strong> {{ servicio.tecnico }}
                </div>
                <div class="q-mb-sm">
                  <q-icon name="event" color="grey-7" />
                  <strong> Recepción:</strong> {{ servicio.fecha }} {{ servicio.hora }}
                </div>
                <div class="q-mb-sm">
                  <q-icon name="payments" color="grey-7" />
                  <strong> Precio:</strong> ${{ formatearDinero(servicio.precio) }}
                </div>

                <div class="q-mb-sm">
                  <q-chip :color="colorPago(servicio.estadoPago)"
                    text-color="white" :icon="iconoPago(servicio.estadoPago)">
                    {{ servicio.estadoPago }}
                  </q-chip>

                  <template v-if="servicio.estadoPago === 'Abono'">
                    <q-chip color="orange" text-color="white">
                      Abono: ${{ formatearDinero(servicio.valorAbono) }}
                    </q-chip>

                    <q-chip color="negative" text-color="white" icon="pending_actions">
                      Resta: ${{ formatearDinero(calcularSaldoPendiente(servicio.precio, servicio.valorAbono)) }}
                    </q-chip>
                  </template>
                </div>

                <div class="q-mb-sm">
                  <q-chip :color="colorEstado(servicio.estadoEquipo)"
                    text-color="white" :icon="iconoEstado(servicio.estadoEquipo)">
                    {{ servicio.estadoEquipo }}
                  </q-chip>
                </div>

                <div v-if="servicio.observaciones">
                  <div class="text-weight-bold">Observaciones:</div>
                  <div class="text-grey-8">{{ servicio.observaciones }}</div>
                </div>

                <div v-if="servicio.estadoEquipo === 'Entregado'" class="q-mt-md">
                  <div class="text-weight-bold q-mb-sm">Calificación del cliente</div>

                  <div v-if="servicio.calificacion">
                    <q-rating :model-value="servicio.calificacion" :max="5"
                      size="28px" color="orange" readonly />
                  </div>

                  <q-btn v-else outline color="orange" icon="star"
                    label="Calificar servicio"
                    @click="abrirCalificacionPorId(servicio.id)" />
                </div>
              </q-card-section>

              <q-separator />

              <q-card-actions align="right">
                <template v-if="servicio.estadoEquipo !== 'Entregado'">
                  <q-btn flat color="primary" icon="edit" label="Editar"
                    @click="editarServicioPorId(servicio.id)" />
                  <q-btn flat color="negative" icon="delete" label="Eliminar"
                    @click="confirmarEliminarPorId(servicio.id)" />
                </template>

                <q-chip v-else color="grey-7" text-color="white" icon="lock">
                  Registro bloqueado
                </q-chip>
              </q-card-actions>
            </q-card>
          </div>
        </div>
      </q-page>
    </q-page-container>

    <!-- Formulario de registro/edición -->
    <q-dialog v-model="dialogoFormulario" persistent>
      <q-card class="form-card">
        <q-card-section class="bg-primary text-white">
          <div class="text-h6">
            {{ editando ? 'Editar servicio' : 'Nuevo servicio' }}
          </div>
        </q-card-section>

        <q-form @submit.prevent="guardarServicio">
          <q-card-section>
            <q-input v-model="formulario.cliente" label="Nombre del cliente *"
              outlined :rules="[reglaRequerido]" class="q-mb-md" />

            <div class="row q-col-gutter-md">
              <q-select v-model="formulario.marca" label="Marca *"
                outlined use-input fill-input hide-selected input-debounce="0"
                :options="opcionesMarcasFiltradas"
                @filter="filtrarMarcas"
                @new-value="crearMarca"
                :rules="[reglaRequerido]"
                class="col-12 col-sm-6 q-mb-md"
                hint="Escribe para buscar o agregar una nueva marca" />

              <q-input v-model="formulario.modelo" label="Modelo *"
                placeholder="Ej: Galaxy A15" outlined
                :rules="[reglaRequerido]" class="col-12 col-sm-6 q-mb-md" />
            </div>

            <!-- Selección múltiple de reparaciones -->
            <q-select v-model="formulario.reparacion"
              label="Tipo de reparación *" outlined
              multiple
              use-chips
              :options="tiposReparacion" :rules="[reglaRequeridoMultiple]"
              class="q-mb-md" />

            <q-select v-model="formulario.tecnico" label="Técnico *"
              outlined :options="tecnicos" :rules="[reglaRequerido]"
              class="q-mb-md" />

            <div class="row q-col-gutter-md">
              <q-input v-model="formulario.fecha" type="date"
                label="Fecha de recepción" outlined readonly
                hint="Fecha generada automáticamente"
                class="col-12 col-sm-6 q-mb-md" />

              <q-input v-model="formulario.hora" type="time"
                label="Hora de recepción *" outlined
                :rules="[reglaRequerido]" class="col-12 col-sm-6 q-mb-md" />
            </div>

            <q-input v-model="formulario.precio" type="number"
              label="Precio cobrado *" prefix="$" outlined min="1"
              :rules="[reglaPrecio]" class="q-mb-md" />

            <q-select v-model="formulario.metodoPago" label="Método de pago *"
              outlined :options="metodosPago" :rules="[reglaRequerido]"
              class="q-mb-md" />

            <q-select v-model="formulario.estadoPago" label="Estado del pago *"
              outlined :options="estadosPago" :rules="[reglaRequerido]"
              class="q-mb-md" />

            <q-input v-if="formulario.estadoPago === 'Abono'"
              v-model="formulario.valorAbono" type="number"
              label="Valor del abono *" prefix="$" outlined min="1"
              :rules="[reglaAbono]" class="q-mb-md" />

            <q-select v-model="formulario.estadoEquipo"
              label="Estado del equipo *" outlined
              :options="estadosEquipo" :rules="[reglaRequerido]"
              class="q-mb-md" />

            <q-input v-model="formulario.observaciones"
              label="Observaciones" type="textarea" outlined rows="3" />
          </q-card-section>

          <q-card-actions align="right">
            <q-btn flat label="Cancelar" color="grey-7"
              @click="cerrarFormulario" />
            <q-btn type="submit" color="primary" icon="save"
              :label="editando ? 'Guardar cambios' : 'Registrar servicio'" />
          </q-card-actions>
        </q-form>
      </q-card>
    </q-dialog>

    <!-- Confirmación de eliminación -->
    <q-dialog v-model="dialogoEliminar">
      <q-card>
        <q-card-section>
          <div class="text-h6">Confirmar eliminación</div>
        </q-card-section>
        <q-card-section>
          ¿Está seguro de que desea eliminar este servicio?
        </q-card-section>
        <q-card-actions align="right">
          <q-btn flat label="Cancelar" color="grey-7"
            @click="dialogoEliminar = false" />
          <q-btn color="negative" label="Eliminar" icon="delete"
            @click="eliminarServicio" />
        </q-card-actions>
      </q-card>
    </q-dialog>

    <!-- Modal de calificación -->
    <q-dialog v-model="dialogoCalificacion">
      <q-card style="width:420px;max-width:95vw">
        <q-card-section class="bg-orange text-white">
          <div class="text-h6">Calificar servicio</div>
        </q-card-section>

        <q-card-section class="text-center q-pa-lg">
          <div class="text-subtitle1 q-mb-md">
            ¿Cómo califica el servicio recibido?
          </div>
          <q-rating v-model="calificacionTemporal" :max="5"
            size="48px" color="orange" icon="star_border" icon-selected="star" />
          <div class="text-grey-7 q-mt-md">
            {{ textoCalificacion }}
          </div>
        </q-card-section>

        <q-card-actions align="right">
          <q-btn flat label="Cancelar" color="grey-7"
            @click="cerrarCalificacion" />
          <q-btn color="orange" icon="star" label="Guardar calificación"
            :disable="!calificacionTemporal"
            @click="guardarCalificacion" />
        </q-card-actions>
      </q-card>
    </q-dialog>
  </q-layout>
</template>

<script setup>
import { ref, computed } from 'vue'
import { useLocalStorage } from '@vueuse/core'

const servicios = useLocalStorage('tecnofix-servicios', [])

// Variables para búsqueda y filtros
const filtroTexto = ref('')
const filtroEstado = ref('Todos')
const filtroPendientesPago = ref(false)

const opcionesFiltroEstado = [
  { etiqueta: 'Todos', valor: 'Todos' },
  { etiqueta: 'Recibido', valor: 'Recibido' },
  { etiqueta: 'En reparación', valor: 'En reparación' },
  { etiqueta: 'Listo para entregar', valor: 'Listo para entregar' },
  { etiqueta: 'Entregado', valor: 'Entregado' }
]

const marcas = ref([
  'Apple', 'Samsung', 'Xiaomi', 'Motorola', 'Huawei',
  'Honor', 'Oppo', 'Realme', 'Vivo', 'Nokia', 'LG', 'Otra'
])

const opcionesMarcasFiltradas = ref([...marcas.value])

const tiposReparacion = [
  'Cambio de pantalla', 'Cambio de batería', 'Cambio de pin de carga',
  'Liberación', 'Mantenimiento de software', 'Cambio de flex', 'Otros'
]

const tecnicos = ['Don Efraín', 'Carlos', 'Andrés']
const metodosPago = ['Efectivo', 'Transferencia', 'Tarjeta']
const estadosPago = ['Pagado', 'Pendiente', 'Abono']
const estadosEquipo = ['Recibido', 'En reparación', 'Listo para entregar', 'Entregado']

const formulario = ref(crearFormulario())
const dialogoFormulario = ref(false)
const dialogoEliminar = ref(false)
const dialogoCalificacion = ref(false)
const editando = ref(false)
const idSeleccionado = ref(null)
const calificacionTemporal = ref(0)

// Computed Property para filtrar la lista dinámicamente
const serviciosFiltrados = computed(() => {
  return servicios.value.filter(s => {
    // 1. Filtro por texto (cliente, marca o modelo)
    const texto = filtroTexto.value.toLowerCase().trim()
    const coincideTexto = !texto || 
      s.cliente.toLowerCase().includes(texto) ||
      s.marca.toLowerCase().includes(texto) ||
      s.modelo.toLowerCase().includes(texto)

    // 2. Filtro por estado del equipo
    const coincideEstado = filtroEstado.value === 'Todos' || s.estadoEquipo === filtroEstado.value

    // 3. Filtro por pendiente de pago
    const coincidePago = !filtroPendientesPago.value || s.estadoPago === 'Pendiente'

    return coincideTexto && coincideEstado && coincidePago
  })
})

function limpiarFiltros() {
  filtroTexto.value = ''
  filtroEstado.value = 'Todos'
  filtroPendientesPago.value = false
}

function filtrarMarcas(val, update) {
  update(() => {
    if (val === '') {
      opcionesMarcasFiltradas.value = marcas.value
    } else {
      const aguja = val.toLowerCase()
      opcionesMarcasFiltradas.value = marcas.value.filter(
        v => v.toLowerCase().indexOf(aguja) > -1
      )
    }
  })
}

function crearMarca(val, done) {
  if (val.length > 0) {
    if (!marcas.value.includes(val)) {
      marcas.value.push(val)
    }
    done(val, 'toggle')
  }
}

function calcularSaldoPendiente(precio, valorAbono) {
  const total = Number(precio || 0)
  const abono = Number(valorAbono || 0)
  const restante = total - abono
  return restante > 0 ? restante : 0
}

function obtenerFechaActual() {
  const ahora = new Date()
  const year = ahora.getFullYear()
  const month = String(ahora.getMonth() + 1).padStart(2, '0')
  const day = String(ahora.getDate()).padStart(2, '0')
  return `${year}-${month}-${day}`
}

function obtenerHoraActual() {
  const ahora = new Date()
  return `${String(ahora.getHours()).padStart(2, '0')}:${String(ahora.getMinutes()).padStart(2, '0')}`
}

function crearFormulario() {
  return {
    id: Date.now(),
    cliente: '',
    marca: '',
    modelo: '',
    reparacion: [],
    tecnico: '',
    fecha: obtenerFechaActual(),
    hora: obtenerHoraActual(),
    precio: '',
    metodoPago: '',
    estadoPago: '',
    valorAbono: '',
    estadoEquipo: 'Recibido',
    calificacion: 0,
    observaciones: ''
  }
}

const resumen = computed(() => [
  { texto: 'Servicios registrados', valor: servicios.value.length, color: 'primary' },
  { texto: 'Pendientes de pago', valor: servicios.value.filter(s => s.estadoPago === 'Pendiente').length, color: 'negative' },
  { texto: 'Listos para entregar', valor: servicios.value.filter(s => s.estadoEquipo === 'Listo para entregar').length, color: 'positive' }
])

const pago = {
  Pagado: ['positive', 'check_circle'],
  Pendiente: ['negative', 'warning'],
  Abono: ['orange', 'payments']
}

const equipo = {
  Recibido: ['blue', 'inventory_2'],
  'En reparación': ['orange', 'build'],
  'Listo para entregar': ['positive', 'local_shipping'],
  Entregado: ['grey-7', 'check_circle']
}

function colorPago(estado) {
  return pago[estado]?.[0] || 'orange'
}

function iconoPago(estado) {
  return pago[estado]?.[1] || 'payments'
}

function colorEstado(estado) {
  return equipo[estado]?.[0] || 'grey-7'
}

function iconoEstado(estado) {
  return equipo[estado]?.[1] || 'check_circle'
}

function reglaRequerido(valor) {
  return (valor !== null && valor !== undefined && String(valor).trim() !== '')
    || 'Este campo es obligatorio'
}

function reglaRequeridoMultiple(valor) {
  return (Array.isArray(valor) && valor.length > 0)
    || 'Selecciona al menos un tipo de reparación'
}

function reglaPrecio(valor) {
  if (valor === '' || valor === null || valor === undefined) {
    return 'El precio es obligatorio'
  }
  return Number(valor) <= 0 ? 'El precio debe ser mayor que 0' : true
}

function reglaAbono(valor) {
  if (formulario.value.estadoPago !== 'Abono') return true
  if (valor === '' || valor === null || valor === undefined) {
    return 'Debe indicar el valor del abono'
  }
  if (Number(valor) <= 0) return 'El abono debe ser mayor que 0'
  return Number(valor) > Number(formulario.value.precio)
    ? 'El abono no puede superar el precio' : true
}

function formatearDinero(valor) {
  return Number(valor || 0).toLocaleString('es-CO')
}

function abrirFormulario() {
  formulario.value = crearFormulario()
  editando.value = false
  idSeleccionado.value = null
  dialogoFormulario.value = true
}

function cerrarFormulario() {
  dialogoFormulario.value = false
  formulario.value = crearFormulario()
  editando.value = false
  idSeleccionado.value = null
}

function guardarServicio() {
  const obligatorios = [
    formulario.value.cliente,
    formulario.value.marca,
    formulario.value.modelo,
    formulario.value.tecnico,
    formulario.value.fecha,
    formulario.value.hora,
    formulario.value.precio,
    formulario.value.metodoPago,
    formulario.value.estadoPago,
    formulario.value.estadoEquipo
  ]

  if (obligatorios.some(valor => valor === '' || valor === null || valor === undefined)) {
    return
  }

  if (!Array.isArray(formulario.value.reparacion) || formulario.value.reparacion.length === 0) {
    return
  }

  if (formulario.value.estadoPago === 'Abono' &&
      (!formulario.value.valorAbono ||
       Number(formulario.value.valorAbono) <= 0 ||
       Number(formulario.value.valorAbono) > Number(formulario.value.precio))) {
    return
  }

  if (editando.value) {
    const idx = servicios.value.findIndex(s => s.id === idSeleccionado.value)
    if (idx !== -1) {
      const servicioActual = servicios.value[idx]
      if (servicioActual.estadoEquipo === 'Entregado') return
      servicios.value[idx] = {
        ...formulario.value,
        id: idSeleccionado.value,
        calificacion: servicioActual.calificacion || 0
      }
    }
  } else {
    servicios.value.push({
      ...formulario.value,
      id: Date.now()
    })
  }

  cerrarFormulario()
}

function editarServicioPorId(id) {
  const servicio = servicios.value.find(s => s.id === id)
  if (!servicio || servicio.estadoEquipo === 'Entregado') return

  let reparacionFormateada = []
  if (Array.isArray(servicio.reparacion)) {
    reparacionFormateada = [...servicio.reparacion]
  } else if (servicio.reparacion) {
    reparacionFormateada = [servicio.reparacion]
  }

  if (servicio.marca && !marcas.value.includes(servicio.marca)) {
    marcas.value.push(servicio.marca)
  }

  formulario.value = {
    ...crearFormulario(),
    ...servicio,
    marca: servicio.marca || '',
    reparacion: reparacionFormateada
  }
  editando.value = true
  idSeleccionado.value = id
  dialogoFormulario.value = true
}

function confirmarEliminarPorId(id) {
  const servicio = servicios.value.find(s => s.id === id)
  if (!servicio || servicio.estadoEquipo === 'Entregado') return

  idSeleccionado.value = id
  dialogoEliminar.value = true
}

function eliminarServicio() {
  const idx = servicios.value.findIndex(s => s.id === idSeleccionado.value)
  if (idx !== -1 && servicios.value[idx]?.estadoEquipo !== 'Entregado') {
    servicios.value.splice(idx, 1)
  }
  idSeleccionado.value = null
  dialogoEliminar.value = false
}

function abrirCalificacionPorId(id) {
  const servicio = servicios.value.find(s => s.id === id)
  if (!servicio || servicio.estadoEquipo !== 'Entregado') return

  idSeleccionado.value = id
  calificacionTemporal.value = servicio.calificacion || 0
  dialogoCalificacion.value = true
}

function cerrarCalificacion() {
  dialogoCalificacion.value = false
  idSeleccionado.value = null
  calificacionTemporal.value = 0
}

function guardarCalificacion() {
  const idx = servicios.value.findIndex(s => s.id === idSeleccionado.value)
  if (idx === -1 || servicios.value[idx]?.estadoEquipo !== 'Entregado') return

  servicios.value[idx] = {
    ...servicios.value[idx],
    calificacion: calificacionTemporal.value
  }

  cerrarCalificacion()
}

const textoCalificacion = computed(() => {
  const textos = {
    1: 'Muy malo',
    2: 'Malo',
    3: 'Regular',
    4: 'Bueno',
    5: 'Excelente'
  }
  return textos[calificacionTemporal.value] || 'Seleccione una calificación'
})
</script>

<style>
body {
  margin: 0;
}

.form-card {
  width: 700px;
  max-width: 95vw;
  font-size: 16px;
}

.border-negative {
  border: 2px solid #f44336;
}

.q-field__label,
.q-field__native,
.q-field__input,
.q-btn,
.q-chip,
.q-card {
  font-size: 16px;
}
</style>