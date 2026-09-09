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

        <q-card v-if="!servicios.length" class="q-pa-xl text-center">
          <q-icon name="build" size="70px" color="grey-5" />
          <div class="text-h6 q-mt-md">No hay servicios registrados</div>
          <div class="text-grey-7 q-mb-md">
            Comienza registrando el primer equipo.
          </div>
          <q-btn color="primary" icon="add"
            label="Registrar servicio" @click="abrirFormulario" />
        </q-card>

        <div v-else class="row q-col-gutter-md">
          <div v-for="(servicio, index) in servicios" :key="servicio.id"
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

                  <q-chip v-if="servicio.estadoPago === 'Abono'"
                    color="orange" text-color="white">
                    Abono: ${{ formatearDinero(servicio.valorAbono) }}
                  </q-chip>
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
                    @click="abrirCalificacion(index)" />
                </div>
              </q-card-section>

              <q-separator />

              <q-card-actions align="right">
                <template v-if="servicio.estadoEquipo !== 'Entregado'">
                  <q-btn flat color="primary" icon="edit" label="Editar"
                    @click="editarServicio(index)" />
                  <q-btn flat color="negative" icon="delete" label="Eliminar"
                    @click="confirmarEliminar(index)" />
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
                outlined :options="marcas" :rules="[reglaRequerido]"
                class="col-12 col-sm-6 q-mb-md" />

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

const marcas = [
  'Apple', 'Samsung', 'Xiaomi', 'Motorola', 'Huawei',
  'Honor', 'Oppo', 'Realme', 'Vivo', 'Nokia', 'LG', 'Otra'
]

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
const indiceEditar = ref(-1)
const indiceEliminar = ref(-1)
const indiceCalificacion = ref(-1)
const calificacionTemporal = ref(0)

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
    reparacion: [], // Inicializado como arreglo para selección múltiple
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
  indiceEditar.value = -1
  dialogoFormulario.value = true
}

function cerrarFormulario() {
  dialogoFormulario.value = false
  formulario.value = crearFormulario()
  editando.value = false
  indiceEditar.value = -1
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

  // Validar campos de texto vacíos
  if (obligatorios.some(valor => valor === '' || valor === null || valor === undefined)) {
    return
  }

  // Validar selección múltiple de reparaciones
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
    const servicioActual = servicios.value[indiceEditar.value]
    if (!servicioActual || servicioActual.estadoEquipo === 'Entregado') return
    servicios.value[indiceEditar.value] = {
      ...formulario.value,
      calificacion: servicioActual.calificacion || 0
    }
  } else {
    servicios.value.push({
      ...formulario.value,
      id: Date.now()
    })
  }

  cerrarFormulario()
}

function editarServicio(index) {
  const servicio = servicios.value[index]
  if (!servicio || servicio.estadoEquipo === 'Entregado') return

  // Asegura compatibilidad si la reparación antigua venía guardada como texto plano
  let reparacionFormateada = []
  if (Array.isArray(servicio.reparacion)) {
    reparacionFormateada = [...servicio.reparacion]
  } else if (servicio.reparacion) {
    reparacionFormateada = [servicio.reparacion]
  }

  formulario.value = {
    ...crearFormulario(),
    ...servicio,
    marca: servicio.marca || '',
    reparacion: reparacionFormateada
  }
  editando.value = true
  indiceEditar.value = index
  dialogoFormulario.value = true
}

function confirmarEliminar(index) {
  const servicio = servicios.value[index]
  if (!servicio || servicio.estadoEquipo === 'Entregado') return

  indiceEliminar.value = index
  dialogoEliminar.value = true
}

function eliminarServicio() {
  const index = indiceEliminar.value
  if (index >= 0 && servicios.value[index]?.estadoEquipo !== 'Entregado') {
    servicios.value.splice(index, 1)
  }
  indiceEliminar.value = -1
  dialogoEliminar.value = false
}

function abrirCalificacion(index) {
  const servicio = servicios.value[index]
  if (!servicio || servicio.estadoEquipo !== 'Entregado') return

  indiceCalificacion.value = index
  calificacionTemporal.value = servicio.calificacion || 0
  dialogoCalificacion.value = true
}

function cerrarCalificacion() {
  dialogoCalificacion.value = false
  indiceCalificacion.value = -1
  calificacionTemporal.value = 0
}

function guardarCalificacion() {
  const index = indiceCalificacion.value
  if (index < 0 || servicios.value[index]?.estadoEquipo !== 'Entregado') return

  servicios.value[index] = {
    ...servicios.value[index],
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