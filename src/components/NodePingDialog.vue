<script setup lang="ts">
import { Icon } from '@iconify/vue'
import { computed, defineAsyncComponent, ref, watch } from 'vue'
import { Button } from '@/components/ui/button'
import {
  Dialog,
  DialogClose,
  DialogContent,
  DialogDescription,
  DialogTitle,
} from '@/components/ui/dialog'

type NodePingDialogMetric = 'latency' | 'loss'

const PingChart = defineAsyncComponent(() => import('@/components/PingChart.vue'))

const props = withDefaults(defineProps<{
  open: boolean
  uuid: string
  nodeName: string
  metric?: NodePingDialogMetric
}>(), {
  metric: 'latency',
})

const emit = defineEmits<{
  'update:open': [value: boolean]
}>()

const dialogOpen = computed({
  get: () => props.open,
  set: value => emit('update:open', value),
})

const chartRenderKey = ref(0)
const metricLabel = computed(() => props.metric === 'loss' ? '丢包' : '延迟')

watch(() => props.open, (open) => {
  if (open) {
    chartRenderKey.value += 1
  }
})

watch(() => props.metric, () => {
  if (props.open) {
    chartRenderKey.value += 1
  }
})
</script>

<template>
  <Dialog v-model:open="dialogOpen">
    <DialogContent class="gap-0">
      <div class="flex items-center gap-3 border-b px-4 py-3">
        <div class="flex size-8 shrink-0 items-center justify-center rounded-md bg-primary/10 text-primary">
          <Icon icon="tabler:activity-heartbeat" :width="16" :height="16" />
        </div>
        <div class="min-w-0 flex-1">
          <DialogTitle class="truncate">
            {{ nodeName }} · {{ metricLabel }}
          </DialogTitle>
          <DialogDescription class="sr-only">
            {{ nodeName }} 的{{ metricLabel }}记录
          </DialogDescription>
        </div>
        <DialogClose as-child>
          <Button variant="ghost" size="icon-sm" :aria-label="`关闭${metricLabel}弹窗`">
            <Icon icon="lucide:x" :width="16" :height="16" />
          </Button>
        </DialogClose>
      </div>
      <div class="max-h-[calc(100vh-7rem)] overflow-y-auto p-4">
        <PingChart :key="chartRenderKey" :uuid="uuid" :initial-metric="metric" />
      </div>
    </DialogContent>
  </Dialog>
</template>
