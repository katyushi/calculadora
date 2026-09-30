<script setup lang="ts">

import { computed, ref } from 'vue'

import { evaluate } from 'mathjs'
import { Media } from '@capacitor-community/media'

/* ============================================================

   ENTRADAS

   ============================================================ */

const demandaInput = ref('2900 - 125 * P')

const ofertaInput = ref('1460 + 115 * P')

const equilibrioManual = ref(false)

const precoEquilibrioInput = ref('6')

const quantidadeEquilibrioInput = ref('2150')

/* ============================================================

   ESTADO

   ============================================================ */

const erro = ref('')

/* ============================================================

   FUNÇÕES AUXILIARES

   ============================================================ */

function normalizarFormula(formula: string): string {

  return formula
    .replace(/\s+/g, ' ')
    .trim()

}

function avaliarFormula(formula: string, P: number): number {

  return evaluate(formula, { P })

}

/**

 * Para as funções econômicas deste app:

 *

 * Q = a + bP

 *

 * queremos:

 *

 * P = (Q - a) / b

 *

 * Descobrimos a e b avaliando a função em P=0 e P=1.

 */

function converterParaGrafico(formula: string) {

  const f0 = avaliarFormula(formula, 0)

  const f1 = avaliarFormula(formula, 1)

  const b = f1 - f0

  const a = f0

  if (!Number.isFinite(a) || !Number.isFinite(b)) {

    throw new Error(`Fórmula inválida: ${formula}`)

  }

  if (Math.abs(b) < 1e-12) {

    throw new Error(
      `Não foi possível isolar P na fórmula: ${formula}`
    )

  }

  return {

    a,

    b,

    /**
     * P = (Q - a) / b
     */

    fn: (Q: number) => (Q - a) / b,

    texto: `(Q - ${a}) / ${b}`,

  }

}

/* ============================================================

   FÓRMULAS

   ============================================================ */

const demanda = computed(() => {

  try {

    return converterParaGrafico(
      normalizarFormula(demandaInput.value)
    )

  } catch {

    return null

  }

})

const oferta = computed(() => {

  try {

    return converterParaGrafico(
      normalizarFormula(ofertaInput.value)
    )

  } catch {

    return null

  }

})

/* ============================================================

   EQUILÍBRIO

   ============================================================ */

const equilibrio = computed(() => {

  if (equilibrioManual.value) {

    const P = Number(precoEquilibrioInput.value)

    const Q = Number(quantidadeEquilibrioInput.value)

    if (!Number.isFinite(P) || !Number.isFinite(Q)) {

      return null

    }

    return { P, Q }

  }

  if (!demanda.value || !oferta.value) {

    return null

  }

  /**
   * Demanda:
   * Q = aD + bD P
   *
   * Oferta:
   * Q = aS + bS P
   *
   * Igualando:
   *
   * aD + bD P = aS + bS P
   *
   * P = (aS - aD) / (bD - bS)
   */

  const aD = demanda.value.a

  const bD = demanda.value.b

  const aS = oferta.value.a

  const bS = oferta.value.b

  const denominador = bD - bS

  if (Math.abs(denominador) < 1e-12) {

    return null

  }

  const P = (aS - aD) / denominador

  const Q = aD + bD * P

  if (!Number.isFinite(P) || !Number.isFinite(Q)) {

    return null

  }

  return { P, Q }

})

/* ============================================================

   FÓRMULAS DO GRÁFICO

   ============================================================ */

const formulaDemandaGrafico = computed(() => {

  if (!demanda.value) return '—'

  const { a, b } = demanda.value

  /**
   * P = (Q-a)/b
   *
   * Simplificamos para:
   *
   * P = Q/b - a/b
   */

  const coeficiente = 1 / b

  const constante = -a / b

  return formatFormula(coeficiente, constante)

})

const formulaOfertaGrafico = computed(() => {

  if (!oferta.value) return '—'

  const { a, b } = oferta.value

  const coeficiente = 1 / b

  const constante = -a / b

  return formatFormula(coeficiente, constante)

})

function formatFormula(coeficiente: number, constante: number): string {

  const c = formatNumber(coeficiente)

  const d = formatNumber(Math.abs(constante))

  if (Math.abs(constante) < 1e-12) {

    return `${c}x`

  }

  if (Math.abs(coeficiente) < 1e-12) {

    return `${formatNumber(constante)}`

  }

  const sinal = constante >= 0 ? '+' : '-'

  return `${c}x ${sinal} ${d}`

}

function formatNumber(value: number): string {

  if (Math.abs(value) < 1e-10) {

    return '0'

  }

  return Number(value.toFixed(6)).toString()

}

/* ============================================================

   GRÁFICO

   ============================================================ */

const graphWidth = 800

const graphHeight = 500

const padding = {

  left: 70,

  right: 30,

  top: 30,

  bottom: 55,

}

const graphAreaWidth =
  graphWidth - padding.left - padding.right

const graphAreaHeight =
  graphHeight - padding.top - padding.bottom

const qMin = computed(() => {

  if (!equilibrio.value) return 0

  return Math.max(0, equilibrio.value.Q - 100)

})

const qMax = computed(() => {

  if (!equilibrio.value) return 1000

  return equilibrio.value.Q + 100

})

const priceValues = computed(() => {

  if (!demanda.value || !oferta.value) {

    return [0, 10]

  }

  const values: number[] = []

  for (let i = 0; i <= 20; i++) {

    const q =
      qMin.value +
      (qMax.value - qMin.value) * (i / 20)

    values.push(demanda.value.fn(q))

    values.push(oferta.value.fn(q))

  }

  if (equilibrio.value) {

    values.push(equilibrio.value.P)

  }

  return values.filter(Number.isFinite)

})

const pMin = computed(() => {

  const values = priceValues.value

  if (!values.length) return 0

  const min = Math.min(...values)

  const max = Math.max(...values)

  const margem = Math.max((max - min) * 0.1, 1)

  return Math.min(0, min - margem)

})

const pMax = computed(() => {

  const values = priceValues.value

  if (!values.length) return 10

  const min = Math.min(...values)

  const max = Math.max(...values)

  const margem = Math.max((max - min) * 0.1, 1)

  return max + margem

})

function xPixel(q: number): number {

  return (
    padding.left +
    ((q - qMin.value) /
      (qMax.value - qMin.value)) *
      graphAreaWidth
  )

}

function yPixel(p: number): number {

  return (
    padding.top +
    graphAreaHeight -
    ((p - pMin.value) /
      (pMax.value - pMin.value)) *
      graphAreaHeight
  )

}

function gerarPath(
  fn: (q: number) => number
): string {

  const pontos: string[] = []

  const quantidadePontos = 300

  for (let i = 0; i < quantidadePontos; i++) {

    const q =
      qMin.value +
      (qMax.value - qMin.value) *
        (i / (quantidadePontos - 1))

    const p = fn(q)

    if (!Number.isFinite(p)) {

      continue

    }

    const x = xPixel(q)

    const y = yPixel(p)

    if (
      y < padding.top - 100 ||
      y > graphHeight - padding.bottom + 100
    ) {

      continue

    }

    pontos.push(
      `${pontos.length === 0 ? 'M' : 'L'} ${x} ${y}`
    )

  }

  return pontos.join(' ')

}

const pathDemanda = computed(() => {

  if (!demanda.value) return ''

  return gerarPath(demanda.value.fn)

})

const pathOferta = computed(() => {

  if (!oferta.value) return ''

  return gerarPath(oferta.value.fn)

})

/* ============================================================

   PONTO DE EQUILÍBRIO

   ============================================================ */

const equilibrioX = computed(() => {

  if (!equilibrio.value) return 0

  return xPixel(equilibrio.value.Q)

})

const equilibrioY = computed(() => {

  if (!equilibrio.value) return 0

  return yPixel(equilibrio.value.P)

})

/* ============================================================

   EIXOS

   ============================================================ */

const xTicks = computed(() => {

  const ticks: number[] = []

  for (let i = 0; i <= 5; i++) {

    ticks.push(
      qMin.value +
        (qMax.value - qMin.value) * (i / 5)
    )

  }

  return ticks

})

const yTicks = computed(() => {

  const ticks: number[] = []

  for (let i = 0; i <= 5; i++) {

    ticks.push(
      pMin.value +
        (pMax.value - pMin.value) * (i / 5)
    )

  }

  return ticks

})

/* ============================================================

   SALVAR GRÁFICO

   ============================================================ */

async function salvarGrafico() {

  try {

    const svg = document.querySelector('.graph') as SVGSVGElement | null

    if (!svg) {

      throw new Error('Gráfico não encontrado.')

    }

    const svgClone = svg.cloneNode(true) as SVGSVGElement

    svgClone.setAttribute(
      'xmlns',
      'http://www.w3.org/2000/svg'
    )

    svgClone.setAttribute(
      'xmlns:xlink',
      'http://www.w3.org/1999/xlink'
    )

    const svgData = new XMLSerializer().serializeToString(svgClone)

    const svgBlob = new Blob(
      [svgData],
      { type: 'image/svg+xml;charset=utf-8' }
    )

    const url = URL.createObjectURL(svgBlob)

    const imagem = new Image()

    await new Promise<void>((resolve, reject) => {

      imagem.onload = () => resolve()

      imagem.onerror = () =>
        reject(new Error('Não foi possível converter o gráfico.'))

      imagem.src = url

    })

    const escala = 2

    const canvas = document.createElement('canvas')

    canvas.width = graphWidth * escala
    canvas.height = graphHeight * escala

    const contexto = canvas.getContext('2d')

    if (!contexto) {

      URL.revokeObjectURL(url)

      throw new Error('Não foi possível criar a imagem.')

    }

    contexto.scale(escala, escala)

    contexto.fillStyle = '#ffffff'

    contexto.fillRect(
      0,
      0,
      graphWidth,
      graphHeight
    )

    contexto.drawImage(
      imagem,
      0,
      0,
      graphWidth,
      graphHeight
    )

    URL.revokeObjectURL(url)

    const dataUrl = canvas.toDataURL(
      'image/png'
    )

    const albumName = 'Oferta e Demanda'

    let albums = await Media.getAlbums()

    let album = albums.albums.find(
      item => item.name === albumName
    )

    if (!album) {

      await Media.createAlbum({
        name: albumName
      })

      albums = await Media.getAlbums()

      album = albums.albums.find(
        item => item.name === albumName
      )

    }

    if (!album) {

      throw new Error(
        'Não foi possível criar ou encontrar o álbum.'
      )

    }

    await Media.savePhoto({

      path: dataUrl,

      albumIdentifier: album.identifier,

      fileName: `grafico-oferta-demanda-${Date.now()}`

    })

    alert('Gráfico salvo na galeria!')

  } catch (e) {

    console.error(e)

    alert(
      e instanceof Error
        ? e.message
        : 'Não foi possível salvar o gráfico.'
    )

  }

}

/* ============================================================

   EXECUTAR / VALIDAR

   ============================================================ */

function calcular() {

  erro.value = ''

  try {

    if (!demanda.value) {

      throw new Error(
        'A fórmula da demanda é inválida.'
      )

    }

    if (!oferta.value) {

      throw new Error(
        'A fórmula da oferta é inválida.'
      )

    }

    if (!equilibrio.value) {

      throw new Error(
        'Não foi possível encontrar o equilíbrio.'
      )

    }

  } catch (e) {

    erro.value =
      e instanceof Error
        ? e.message
        : 'Erro desconhecido.'

  }

}

calcular()

</script>

<template>

  <main class="app">

    <header class="header">

      <h1>Oferta e Demanda</h1>

      <p>
        Calculadora gráfica de equilíbrio de mercado
      </p>

    </header>

    <!-- ======================================================
         ENTRADAS
         ====================================================== -->

    <section class="panel">

      <h2>Funções do mercado</h2>

      <div class="inputs">

        <label>

          <span>Demanda — QD</span>

          <input
            v-model="demandaInput"
            type="text"
            spellcheck="false"
            @keyup.enter="calcular"
          >

        </label>

        <label>

          <span>Oferta — QS</span>

          <input
            v-model="ofertaInput"
            type="text"
            spellcheck="false"
            @keyup.enter="calcular"
          >

        </label>

      </div>

      <div class="equilibrium-toggle">

        <label class="checkbox">

          <input
            v-model="equilibrioManual"
            type="checkbox"
          >

          <span>
            Informar equilíbrio manualmente
          </span>

        </label>

      </div>

      <div
        v-if="equilibrioManual"
        class="inputs"
      >

        <label>

          <span>Preço de equilíbrio — P</span>

          <input
            v-model="precoEquilibrioInput"
            type="number"
            step="any"
          >

        </label>

        <label>

          <span>Quantidade de equilíbrio — Q</span>

          <input
            v-model="quantidadeEquilibrioInput"
            type="number"
            step="any"
          >

        </label>

      </div>

      <button
        type="button"
        @click="calcular"
      >

        Calcular

      </button>

      <p
        v-if="erro"
        class="error"
      >

        {{ erro }}

      </p>

    </section>

    <!-- ======================================================
         EQUILÍBRIO
         ====================================================== -->

    <section
      v-if="equilibrio"
      class="panel"
    >

      <h2>Equilíbrio</h2>

      <div class="equilibrium">

        <div>

          <span>Preço</span>

          <strong>
            P = {{ formatNumber(equilibrio.P) }}
          </strong>

        </div>

        <div>

          <span>Quantidade</span>

          <strong>
            Q = {{ formatNumber(equilibrio.Q) }}
          </strong>

        </div>

      </div>

    </section>

    <!-- ======================================================
         FÓRMULAS PARA CALCULADORA GRÁFICA
         ====================================================== -->

    <section class="panel">

      <h2>Fórmulas para calculadora gráfica</h2>

      <div class="formula">

        <span>Demanda</span>

        <code>
          y = {{ formulaDemandaGrafico }}
        </code>

      </div>

      <div class="formula">

        <span>Oferta</span>

        <code>
          y = {{ formulaOfertaGrafico }}
        </code>

      </div>

    </section>

    <!-- ======================================================
         GRÁFICO
         ====================================================== -->

    <section class="panel graph-panel">

      <h2>Gráfico</h2>

      <div class="graph-wrapper">

        <svg
          :viewBox="`0 0 ${graphWidth} ${graphHeight}`"
          class="graph"
          role="img"
          aria-label="Gráfico de oferta e demanda"
        >

          <!-- Grade vertical -->

          <g class="grid">

            <line
              v-for="q in xTicks"
              :key="`x-${q}`"
              :x1="xPixel(q)"
              :x2="xPixel(q)"
              :y1="padding.top"
              :y2="graphHeight - padding.bottom"
            />

          </g>

          <!-- Grade horizontal -->

          <g class="grid">

            <line
              v-for="p in yTicks"
              :key="`y-${p}`"
              :x1="padding.left"
              :x2="graphWidth - padding.right"
              :y1="yPixel(p)"
              :y2="yPixel(p)"
            />

          </g>

          <!-- Eixos -->

          <line
            class="axis"
            :x1="padding.left"
            :x2="graphWidth - padding.right"
            :y1="graphHeight - padding.bottom"
            :y2="graphHeight - padding.bottom"
          />

          <line
            class="axis"
            :x1="padding.left"
            :x2="padding.left"
            :y1="padding.top"
            :y2="graphHeight - padding.bottom"
          />

          <!-- Demanda -->

          <path
            v-if="pathDemanda"
            :d="pathDemanda"
            class="demand-line"
            fill="none"
          />

          <!-- Oferta -->

          <path
            v-if="pathOferta"
            :d="pathOferta"
            class="supply-line"
            fill="none"
          />

          <!-- Linha vertical do equilíbrio -->

          <line
            v-if="equilibrio"
            class="equilibrium-line"
            :x1="equilibrioX"
            :x2="equilibrioX"
            :y1="equilibrioY"
            :y2="graphHeight - padding.bottom"
          />

          <!-- Linha horizontal do equilíbrio -->

          <line
            v-if="equilibrio"
            class="equilibrium-line"
            :x1="padding.left"
            :x2="equilibrioX"
            :y1="equilibrioY"
            :y2="equilibrioY"
          />

          <!-- Ponto de equilíbrio -->

          <circle
            v-if="equilibrio"
            :cx="equilibrioX"
            :cy="equilibrioY"
            r="7"
            class="equilibrium-point"
          />

          <!-- Labels eixo X -->

          <g class="labels">

            <text
              v-for="q in xTicks"
              :key="`xl-${q}`"
              :x="xPixel(q)"
              :y="graphHeight - padding.bottom + 25"
              text-anchor="middle"
            >

              {{ formatNumber(q) }}

            </text>

          </g>

          <!-- Labels eixo Y -->

          <g class="labels">

            <text
              v-for="p in yTicks"
              :key="`yl-${p}`"
              :x="padding.left - 10"
              :y="yPixel(p) + 4"
              text-anchor="end"
            >

              {{ formatNumber(p) }}

            </text>

          </g>

          <!-- Títulos dos eixos -->

          <text
            class="axis-title"
            :x="graphWidth / 2"
            :y="graphHeight - 10"
            text-anchor="middle"
          >

            Quantidade (Q)

          </text>

          <text
            class="axis-title"
            :x="18"
            :y="graphHeight / 2"
            text-anchor="middle"
            transform="rotate(-90 18 250)"
          >

            Preço (P)

          </text>

          <!-- Texto do equilíbrio -->

          <text
            v-if="equilibrio"
            class="equilibrium-label"
            :x="equilibrioX + 12"
            :y="equilibrioY - 12"
          >

            ({{ formatNumber(equilibrio.Q) }},
            {{ formatNumber(equilibrio.P) }})

          </text>

        </svg>

      </div>

      <!-- Salvar gráfico -->

      <button
        type="button"
        class="save-graph-button"
        @click="salvarGrafico"
      >

        Salvar gráfico na galeria

      </button>

      <!-- Legenda -->

      <div class="legend">

        <div>

          <span class="legend-line demand" />

          Demanda

        </div>

        <div>

          <span class="legend-line supply" />

          Oferta

        </div>

        <div>

          <span class="legend-point" />

          Equilíbrio

        </div>

      </div>

    </section>

  </main>

</template>

<style scoped>

/* ============================================================

   LAYOUT

   ============================================================ */

.app {

  max-width: 100vw;

  margin: 0 auto;

  padding: 24px;

  font-family:
    Inter,
    system-ui,
    -apple-system,
    BlinkMacSystemFont,
    "Segoe UI",
    sans-serif;

  color: #1f2937;

}

.header {

  margin-bottom: 24px;

}

.header h1 {

  margin: 0;

  font-size: 2rem;

}

.header p {

  margin: 6px 0 0;

  color: #000;

}

/* ============================================================

   PAINÉIS

   ============================================================ */

.panel {

  margin-bottom: 20px;

  padding: 20px;

  border: 1px solid #e5e7eb;

  border-radius: 12px;

  background: white;

}

.panel h2 {

  margin: 0 0 16px;

  font-size: 1.15rem;

}

/* ============================================================

   INPUTS

   ============================================================ */

.inputs {

  display: grid;

  grid-template-columns:
    repeat(2, minmax(0, 1fr));

  gap: 16px;

  margin-bottom: 16px;

}

label > span {

  display: block;

  margin-bottom: 6px;

  font-size: 0.9rem;

  font-weight: 600;

}

input[type="text"],
input[type="number"] {

  width: 100%;

  box-sizing: border-box;

  padding: 11px 12px;

  border: 1px solid #d1d5db;

  border-radius: 8px;

  font: inherit;

}

input:focus {

  outline: 2px solid #93c5fd;

  outline-offset: 1px;

}

.equilibrium-toggle {

  margin-bottom: 16px;

}

.checkbox {

  display: flex;

  align-items: center;

  gap: 8px;

}

.checkbox input {

  width: 18px;

  height: 18px;

}

button {

  padding: 11px 18px;

  border: 0;

  border-radius: 8px;

  background: #2563eb;

  color: white;

  font: inherit;

  font-weight: 600;

  cursor: pointer;

}

button:hover {

  background: #1d4ed8;

}

/* ============================================================

   EQUILÍBRIO

   ============================================================ */

.equilibrium {

  display: flex;

  gap: 48px;

}

.equilibrium div {

  display: flex;

  flex-direction: column;

  gap: 4px;

}

.equilibrium span {

  color: #6b7280;

  font-size: 0.9rem;

}

.equilibrium strong {

  font-size: 1.4rem;

}

/* ============================================================

   FÓRMULAS

   ============================================================ */

.formula {

  display: flex;

  align-items: center;

  gap: 16px;

  margin: 10px 0;

}

.formula span {

  width: 80px;

  font-weight: 600;

}

.formula code {

  padding: 8px 12px;

  border-radius: 6px;

  background: #f3f4f6;

  font-family: "JetBrains Mono", monospace;

}

/* ============================================================

   GRÁFICO

   ============================================================ */

.graph-panel {

  overflow: hidden;

}

.graph-wrapper {

  width: 100%;

  overflow-x: auto;

}

.graph {

  display: block;

  width: 100%;

  min-width: 600px;

  height: auto;

}

.grid line {

  stroke: #e5e7eb;

  stroke-width: 1;

}

.axis {

  stroke: #374151;

  stroke-width: 1.5;

}

.demand-line {

  stroke: #2563eb;

  stroke-width: 3;

}

.supply-line {

  stroke: #dc2626;

  stroke-width: 3;

}

.equilibrium-line {

  stroke: #6b7280;

  stroke-width: 1.5;

  stroke-dasharray: 7 5;

}

.equilibrium-point {

  fill: #111827;

  stroke: white;

  stroke-width: 3;

}

.labels {

  fill: #4b5563;

  font-size: 12px;

}

.axis-title {

  fill: #111827;

  font-size: 14px;

  font-weight: 600;

}

.equilibrium-label {

  fill: #111827;

  font-size: 13px;

  font-weight: 600;

}

/* ============================================================

   BOTÃO SALVAR GRÁFICO

   ============================================================ */

.save-graph-button {

  display: block;

  width: 100%;

  margin-top: 14px;

  padding: 12px 18px;

  border: 0;

  border-radius: 8px;

  background: #2563eb;

  color: white;

  font: inherit;

  font-weight: 600;

  cursor: pointer;

}

.save-graph-button:hover {

  background: #1d4ed8;

}

.legend {

  display: flex;

  flex-wrap: wrap;

  gap: 20px;

  margin-top: 14px;

  font-size: 0.9rem;

}

.legend > div {

  display: flex;

  align-items: center;

  gap: 7px;

}

.legend-line {

  width: 24px;

  height: 3px;

  display: inline-block;

}

.legend-line.demand {

  background: #2563eb;

}

.legend-line.supply {

  background: #dc2626;

}

.legend-point {

  width: 10px;

  height: 10px;

  border-radius: 50%;

  background: #111827;

  display: inline-block;

}

.error {

  margin-top: 12px;

  color: #b91c1c;

  font-weight: 500;

}

/* ============================================================

   MOBILE

   ============================================================ */

@media (max-width: 700px) {

  .app {

    padding: 16px;

  }

  .inputs {

    grid-template-columns: 1fr;

  }

  .equilibrium {

    gap: 24px;

  }

  .formula {

    align-items: flex-start;

    flex-direction: column;

    gap: 6px;

  }

  .formula span {

    width: auto;

  }

}

</style>
