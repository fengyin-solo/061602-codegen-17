<script setup lang="ts">
import { computed } from 'vue'
import type { InsulationStrategy, GameState } from '@/types/game'
import {
  INSULATION_EFFECTS,
  INSULATION_NAMES,
  INSULATION_EMOJI,
  HARSH_WEATHER_PENALTY,
  HARSH_WEATHERS,
} from '@/utils/constants'

const props = defineProps<{
  state: GameState
  eggCount: number
  isHarshWeather: boolean
  currentEggProgressMod: number
}>()

const emit = defineEmits<{
  (e: 'setStrategy', strategy: InsulationStrategy): void
}>()

const strategies: InsulationStrategy[] = ['natural', 'straw', 'blanket', 'heater']

const strategyCards = computed(() => {
  return strategies.map(strategy => {
    const effect = INSULATION_EFFECTS[strategy]
    const selected = props.state.insulationStrategy === strategy

    let effectiveMod = 100
    if (props.isHarshWeather) {
      const penalty = HARSH_WEATHER_PENALTY[props.state.currentWeather]
      effectiveMod = Math.round(penalty * effect.eggProgressMod * 100)
    }

    const canAfford = props.state.foodStock >= effect.foodConsumptionPerMinute || strategy === 'natural'

    return {
      strategy,
      name: INSULATION_NAMES[strategy],
      emoji: INSULATION_EMOJI[strategy],
      description: effect.description,
      consumption: effect.foodConsumptionPerMinute,
      eggProgressMod: Math.round(effect.eggProgressMod * 100),
      effectiveMod,
      selected,
      canAfford,
      disabled: !canAfford && !selected,
    }
  })
})

const weatherPenaltyPercent = computed(() => {
  if (!props.isHarshWeather) return 0
  return Math.round((1 - HARSH_WEATHER_PENALTY[props.state.currentWeather]) * 100)
})

const panelStyle = computed(() => {
  if (!props.isHarshWeather) return 'from-teal-500/20 to-emerald-500/10 border-teal-400/30'
  switch (props.state.currentWeather) {
    case 'rainy': return 'from-blue-600/30 to-indigo-700/20 border-blue-400/40'
    case 'snowy': return 'from-cyan-400/30 to-blue-300/20 border-cyan-300/40'
    case 'stormy': return 'from-purple-700/40 to-gray-800/30 border-purple-500/40'
    default: return 'from-teal-500/20 to-emerald-500/10 border-teal-400/30'
  }
})
</script>

<template>
  <div
    class="rounded-2xl border backdrop-blur-sm overflow-hidden transition-all"
    :class="panelStyle"
  >
    <div class="px-4 py-3 border-b border-white/10 flex items-center justify-between">
      <div class="flex items-center gap-2">
        <span class="text-xl">{{ isHarshWeather ? '🏠' : '🌿' }}</span>
        <span class="font-display text-amber-200 font-medium">孵化保温</span>
      </div>
      <div v-if="isHarshWeather" class="flex items-center gap-1.5 px-2 py-1 bg-red-500/30 rounded-lg">
        <span class="text-xs text-red-200">⚠️ 寒冷惩罚 -{{ weatherPenaltyPercent }}%</span>
      </div>
      <div v-else class="flex items-center gap-1.5 px-2 py-1 bg-green-500/30 rounded-lg">
        <span class="text-xs text-green-200">☀️ 温度适宜</span>
      </div>
    </div>

    <div class="px-4 py-3 border-b border-white/10">
      <div class="flex items-center justify-between text-xs mb-1.5">
        <span class="text-white/70">当前孵化效率</span>
        <span
          class="font-bold"
          :class="{
            'text-green-400': currentEggProgressMod >= 100,
            'text-yellow-400': currentEggProgressMod >= 70 && currentEggProgressMod < 100,
            'text-orange-400': currentEggProgressMod >= 40 && currentEggProgressMod < 70,
            'text-red-400': currentEggProgressMod < 40,
          }"
        >
          {{ currentEggProgressMod }}%
        </span>
      </div>
      <div class="w-full h-2 bg-black/30 rounded-full overflow-hidden">
        <div
          class="h-full transition-all duration-500 rounded-full"
          :class="{
            'bg-gradient-to-r from-green-400 to-emerald-500': currentEggProgressMod >= 100,
            'bg-gradient-to-r from-yellow-400 to-amber-500': currentEggProgressMod >= 70 && currentEggProgressMod < 100,
            'bg-gradient-to-r from-orange-400 to-orange-500': currentEggProgressMod >= 40 && currentEggProgressMod < 70,
            'bg-gradient-to-r from-red-400 to-red-500': currentEggProgressMod < 40,
          }"
          :style="{ width: `${Math.min(currentEggProgressMod, 150)}%` }"
        />
      </div>
      <div class="flex items-center justify-between mt-2 text-[11px] text-white/50">
        <span>🥚 待孵化蛋: {{ eggCount }} 颗</span>
        <span>当前: {{ INSULATION_NAMES[state.insulationStrategy] }}</span>
      </div>
    </div>

    <div class="p-3 grid grid-cols-2 gap-2">
      <button
        v-for="card in strategyCards"
        :key="card.strategy"
        class="relative p-3 rounded-xl border transition-all text-left"
        :class="[
          card.selected
            ? 'bg-amber-400/20 border-amber-400/60 ring-1 ring-amber-300/40'
            : card.disabled
              ? 'bg-black/20 border-white/10 opacity-50 cursor-not-allowed'
              : 'bg-white/5 border-white/15 hover:bg-white/10 hover:border-white/25 cursor-pointer',
        ]"
        :disabled="card.disabled"
        @click="!card.disabled && emit('setStrategy', card.strategy)"
      >
        <div class="flex items-center gap-2 mb-1.5">
          <span class="text-lg">{{ card.emoji }}</span>
          <span
            class="text-sm font-medium"
            :class="card.selected ? 'text-amber-200' : 'text-white/90'"
          >
            {{ card.name.split(' ')[1] }}
          </span>
        </div>

        <div class="space-y-1">
          <div class="flex items-center justify-between text-[11px]">
            <span class="text-white/50">孵化加速</span>
            <span
              class="font-medium"
              :class="card.eggProgressMod > 100 ? 'text-green-400' : 'text-white/70'"
            >
              +{{ card.eggProgressMod - 100 }}%
            </span>
          </div>

          <div v-if="isHarshWeather" class="flex items-center justify-between text-[11px]">
            <span class="text-white/50">实际效率</span>
            <span
              class="font-medium"
              :class="{
                'text-green-400': card.effectiveMod >= 100,
                'text-yellow-400': card.effectiveMod >= 70 && card.effectiveMod < 100,
                'text-orange-400': card.effectiveMod >= 40 && card.effectiveMod < 70,
                'text-red-400': card.effectiveMod < 40,
              }"
            >
              {{ card.effectiveMod }}%
            </span>
          </div>

          <div class="flex items-center justify-between text-[11px]">
            <span class="text-white/50">食物消耗</span>
            <span
              class="font-medium"
              :class="card.consumption === 0 ? 'text-teal-400' : card.canAfford ? 'text-amber-300' : 'text-red-400'"
            >
              {{ card.consumption === 0 ? '无' : `${card.consumption}/分钟` }}
            </span>
          </div>
        </div>

        <div
          v-if="card.selected"
          class="absolute top-1.5 right-1.5 w-2 h-2 rounded-full bg-amber-400 animate-pulse"
        />
      </button>
    </div>

    <div class="px-4 py-2 border-t border-white/10 text-[11px] text-white/50 leading-relaxed">
      💡 恶劣天气下蛋孵化会变慢，切换保温策略可抵消惩罚并加速孵化，但会消耗食物。
    </div>
  </div>
</template>
