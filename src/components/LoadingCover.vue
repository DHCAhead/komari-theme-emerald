<script setup lang="ts">
import { computed } from 'vue'
import { useAppStore } from '@/stores/app'

interface PixelCell {
  key: string
  color: string | null
  revealAt: number
}

const appStore = useAppStore()
const isDark = computed(() => appStore.isDark)
const gridSize = 29

const loadingProgress = computed(() => Math.min(Math.max(appStore.loadingProgress ?? 0, 0), 100))
const revealProgress = computed(() => loadingProgress.value / 100)
const progressStyle = computed(() => ({ width: `${loadingProgress.value}%` }))

function getLandColor(x: number, y: number): string | null {
  const northAmerica = x < -0.18 && y < -0.06 && y > -0.64
    && (
      ((x + 0.55) / 0.34) ** 2 + ((y + 0.34) / 0.26) ** 2 < 1
      || ((x + 0.34) / 0.22) ** 2 + ((y + 0.18) / 0.2) ** 2 < 1
    )
  const southAmerica = x < -0.18 && x > -0.56 && y > 0.1
    && ((x + 0.35) / 0.17) ** 2 + ((y - 0.42) / 0.36) ** 2 < 1
  const eurasia = x > -0.08 && x < 0.82 && y < -0.02 && y > -0.6
    && (
      ((x - 0.28) / 0.52) ** 2 + ((y + 0.3) / 0.28) ** 2 < 1
      || ((x - 0.56) / 0.26) ** 2 + ((y + 0.16) / 0.2) ** 2 < 1
    )
  const africa = x > 0.02 && x < 0.43 && y > -0.02
    && ((x - 0.2) / 0.2) ** 2 + ((y - 0.26) / 0.38) ** 2 < 1
  const australia = x > 0.42 && x < 0.78 && y > 0.38
    && ((x - 0.6) / 0.2) ** 2 + ((y - 0.52) / 0.12) ** 2 < 1

  if (!northAmerica && !southAmerica && !eurasia && !africa && !australia)
    return null
  if (y < -0.32)
    return '#9ad66b'
  if (y > 0.45)
    return '#28a86b'
  return '#43c172'
}

function getEarthColor(row: number, col: number): string | null {
  const center = (gridSize - 1) / 2
  const radius = gridSize * 0.45
  const x = (col - center) / radius
  const y = (row - center) / radius
  const distance = Math.sqrt(x ** 2 + y ** 2)

  if (distance > 1)
    return null

  const landColor = getLandColor(x, y)
  if (landColor)
    return landColor
  if (distance > 0.9)
    return '#1f6bc7'
  if (x < -0.35 && y < -0.28)
    return '#7dd3fc'
  if (x < -0.05 && y < 0.04)
    return '#38bdf8'
  return '#2f8fe8'
}

const pixelCells = computed<PixelCell[]>(() => {
  const cells: Array<PixelCell & { diagonal: number }> = []
  const diagonals: number[] = []

  for (let row = 0; row < gridSize; row++) {
    for (let col = 0; col < gridSize; col++) {
      const color = getEarthColor(row, col)
      const diagonal = row + col
      if (color)
        diagonals.push(diagonal)
      cells.push({
        key: `${row}-${col}`,
        color,
        diagonal,
        revealAt: 1,
      })
    }
  }

  const minDiagonal = Math.min(...diagonals)
  const maxDiagonal = Math.max(...diagonals)
  const span = maxDiagonal - minDiagonal || 1

  return cells.map(({ diagonal, ...cell }) => ({
    ...cell,
    revealAt: cell.color ? (diagonal - minDiagonal) / span : 1,
  }))
})

const visiblePixelCells = computed(() => {
  return pixelCells.value.map((cell) => {
    const visible = cell.color !== null && cell.revealAt <= revealProgress.value
    return {
      ...cell,
      style: {
        backgroundColor: cell.color ?? 'transparent',
        opacity: visible ? '1' : '0',
        transform: visible ? 'scale(1)' : 'scale(0.25)',
      },
    }
  })
})
</script>

<template>
  <div
    class="loading-cover fixed inset-0 z-20 flex items-center justify-center backdrop-blur-sm"
    :class="isDark ? 'bg-black/50' : 'bg-white/80'"
  >
    <div class="flex flex-col items-center gap-4 text-foreground">
      <div
        class="pixel-earth"
        role="progressbar"
        aria-label="加载进度"
        aria-valuemin="0"
        aria-valuemax="100"
        :aria-valuenow="loadingProgress"
      >
        <span
          v-for="pixel in visiblePixelCells"
          :key="pixel.key"
          class="pixel-earth__cell"
          :style="pixel.style"
        />
      </div>
      <div class="flex w-44 flex-col items-center gap-2">
        <div class="h-1 w-full overflow-hidden rounded-full bg-muted/70">
          <div class="h-full rounded-full bg-emerald-500 transition-[width] duration-200 ease-out" :style="progressStyle" />
        </div>
        <div class="text-xs font-medium text-muted-foreground tabular-nums">
          {{ Math.round(loadingProgress) }}%
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.pixel-earth {
  display: grid;
  width: 156px;
  aspect-ratio: 1;
  grid-template-columns: repeat(29, minmax(0, 1fr));
  gap: 2px;
  padding: 2px;
  image-rendering: pixelated;
}

.pixel-earth__cell {
  aspect-ratio: 1;
  border-radius: 1px;
  box-shadow: inset -1px -1px 0 rgb(0 0 0 / 14%);
  transition:
    opacity 160ms ease,
    transform 180ms cubic-bezier(0.22, 1, 0.36, 1);
}
</style>
