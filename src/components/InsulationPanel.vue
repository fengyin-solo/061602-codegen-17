<script setup lang="ts">
import { computed } from 'vue'
import type { InsulationStrategy, GameState } from '@/types/game'
import {
  INSULATION_EFFECTS,
  INSULATION_NAMES,
  INSULATION_EMOJI,
  HARSH_WEATHER_PENALTY,
  HARSH_WEATHERS,
  INSULATION_CONSUMPTION_INTERVAL,
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
    const consumptionPerMinute = effect.foodConsumptionPerMinute

    let effectiveMod = 100
    let netEffect = 0
    if (props.isHarshWeather) {
      const penalty = HARSH_WEATHER_PENALTY[props.state.currentWeather]
      effectiveMod = Math.round(penalty * effect.eggProgressMod * 100)
      netEffect = effectiveMod - Math.round(penalty * 100)
    } else {
      effectiveMod = Math.round(effect.eggProgressMod * 100)
      netEffect = effectiveMod - 100
    }

    const consumptionPer10s = (consumptionPerMinute * (INSULATION_CONSUMPTION_INTERVAL / 60000))
    const consumptionPerHour = consumptionPerMinute * 60

    const consumptionForOneEgg = props.eggCount > 0
      ? Math.ceil(consumptionPerMinute / props.eggCount)
      : 0

    const canAfford = strategy === 'natural' || props.state.foodStock >= consumptionPerMinute

    const costToHatch = computed(() => {
      if (strategy === 'natural' || props.eggCount === 0) return { min: 0, max: 0 }

      const eggs = props.state.birds.filter(b => b.stage === 'egg' && !b.isDead)
      if (eggs.length === 0) return { min: 0, max: 0 }

      const maxTimeLeft = Math.max(...eggs.map(e => e.hatchTimeLeft)) / 1000 / 60
      const minTimeLeft = Math.min(...eggs.map(e => e.hatchTimeLeft)) / 1000 / 60

      return {
        min: Math.ceil(minTimeLeft * consumptionPerMinute / effectiveMod * 100),
        max: Math.ceil(maxTimeLeft * consumptionPerMinute / effectiveMod * 100),
      }
    }).value

    return {
      strategy,
      name: INSULATION_NAMES[strategy],
      emoji: INSULATION_EMOJI[strategy],
      description: effect.description,
      consumptionPerMinute,
      consumptionPer10s: consumptionPer10s.toFixed(1),
      consumptionPerHour,
      consumptionForOneEgg,
      eggProgressMod: Math.round(effect.eggProgressMod * 100),
      effectiveMod,
      netEffect,
      selected,
      canAfford,
      disabled: !canAfford && !selected,
      costToHatch,
    }
  })
})

const weatherPenaltyPercent = computed(() => {
  if (!props.isHarshWeather) return 0
  return Math.round((1 - HARSH_WEATHER_PENALTY[props.state.currentWeather]) * 100)
})

const baseEfficiency = computed(() => {
  if (!props.isHarshWeather) return 100
  return Math.round(HARSH_WEATHER_PENALTY[props.state.currentWeather] * 100)
})

const currentStrategyConsumption = computed(() => {
  const effect = INSULATION_EFFECTS[props.state.insulationStrategy]
  return effect.foodConsumptionPerMinute
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

const efficiencyColorClass = (mod: number) => {
  if (mod >= 120) return 'text-green-400'
  if (mod >= 100) return 'text-emerald-300'
  if (mod >= 80) return 'text-yellow-400'
  if (mod >= 50) return 'text-orange-400'
  return 'text-red-400'
}

const handleStrategyClick = (strategy: InsulationStrategy, disabled: boolean) => {
  if (!disabled) {
    emit('setStrategy', strategy)
  }
}
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
        <span class="text-xs text-red-200">⚠️ 寒冷 -{{ weatherPenaltyPercent }}%</span>
      </div>
      <div v-else class="flex items-center gap-1.5 px-2 py-1 bg-green-500/30 rounded-lg">
        <span class="text-xs text-green-200">☀️ 适宜</span>
      </div>
    </div>

    <div class="px-4 py-3 border-b border-white/10">
      <div class="flex items-center justify-between text-xs mb-1.5">
        <span class="text-white/70">
          {{ isHarshWeather ? '抵消惩罚后实际效率' : '当前孵化效率' }}
        </span>
        <span class="font-bold" :class="efficiencyColorClass(currentEggProgressMod)">
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
      <div class="flex items-center justify-between mt-2 text-[10px] text-white/50">
        <span>🥚 待孵化: {{ eggCount }} 颗</span>
        <span v-if="currentStrategyConsumption > 0">
          🔥 消耗: {{ currentStrategyConsumption }}/分钟
        </span>
        <span v-else>🌿 自然保温</span>
      </div>
      <div v-if="isHarshWeather" class="mt-1.5 text-[10px] text-orange-300/80">
        💡 基础效率 {{ baseEfficiency }}%，保温可抵消惩罚并额外加速
      </div>
    </div>

    <div class="p-3 grid grid-cols-2 gap-2">
      <button
        v-for="card in strategyCards"
        :key="card.strategy"
        class="relative p-2.5 rounded-xl border transition-all text-left group"
        :class="[
          card.selected
            ? 'bg-amber-400/20 border-amber-400/60 ring-1 ring-amber-300/40'
            : card.disabled
              ? 'bg-black/20 border-white/10 opacity-50 cursor-not-allowed'
              : 'bg-white/5 border-white/15 hover:bg-white/10 hover:border-white/25 cursor-pointer',
        ]"
        :disabled="card.disabled"
        @click="handleStrategyClick(card.strategy, card.disabled)"
      >
        <div class="flex items-center gap-1.5 mb-1.5">
          <span class="text-base">{{ card.emoji }}</span>
          <span
            class="text-xs font-medium truncate"
            :class="card.selected ? 'text-amber-200' : 'text-white/90'"
          >
            {{ card.name.split(' ')[1] }}
          </span>
        </div>

        <div class="space-y-0.5">
          <div class="flex items-center justify-between text-[9px]">
            <span class="text-white/50">基础加速</span>
            <span
              class="font-medium"
              :class="card.eggProgressMod > 100 ? 'text-green-400' : 'text-white/70'"
            >
              +{{ card.eggProgressMod - 100 }}%
            </span>
          </div>

          <div v-if="isHarshWeather" class="flex items-center justify-between text-[9px]">
            <span class="text-white/50">实际效率</span>
            <span
              class="font-medium"
              :class="efficiencyColorClass(card.effectiveMod)"
            >
              {{ card.effectiveMod }}%
              <span v-if="card.netEffect !== 0" :class="card.netEffect > 0 ? 'text-green-400' : 'text-red-400'">
                ({{ card.netEffect > 0 ? '+' : '' }}{{ card.netEffect }}%)
              </span>
            </span>
          </div>

          <div class="flex items-center justify-between text-[9px]">
            <span class="text-white/50">食物消耗</span>
            <span
              class="font-medium"
              :class="card.consumptionPerMinute === 0 ? 'text-teal-400' : card.canAfford ? 'text-amber-300' : 'text-red-400'"
            >
              {{ card.consumptionPerMinute === 0 ? '无' : `${card.consumptionPerMinute}/分` }}
            </span>
          </div>

          <div v-if="card.consumptionPerMinute > 0" class="flex items-center justify-between text-[8px] text-white/40">
            <span>每10秒: {{ card.consumptionPer10s }}</span>
            <span>每小时: {{ card.consumptionPerHour }}</span>
          </div>

          <div
            v-if="card.consumptionForOneEgg > 0 && eggCount > 0"
            class="mt-1 pt-1 border-t border-white/5 text-[8px] text-white/40"
          >
            🥚 单蛋成本: ~{{ card.consumptionForOneEgg }}/分钟·颗
          </div>

          <div
            v-if="card.costToHatch.max > 0 && eggCount > 0"
            class="text-[8px] text-white/40"
          >
            💰 孵完全部预计: {{ card.costToHatch.min }}-{{ card.costToHatch.max }}
          </div>
        </div>

        <div
          v-if="card.selected"
          class="absolute top-1.5 right-1.5 w-2 h-2 rounded-full bg-amber-400 animate-pulse"
        />

        <div
          v-if="card.disabled && !card.selected"
          class="absolute inset-0 flex items-center justify-center bg-black/40 rounded-xl"
        >
          <span class="text-[10px] text-red-300">食物不足</span>
        </div>
      </button>
    </div>

    <div class="px-4 py-2 border-t border-white/10 text-[10px] text-white/50 leading-relaxed space-y-0.5">
      <div>💡 消耗按实际时间精确计算，每10秒结算一次，小数部分累计到下次。</div>
      <div v-if="isHarshWeather">
        📊 当前: 惩罚{{ weatherPenaltyPercent }}% × 保温加速 = 实际效率{{ currentEggProgressMod }}%
      </div>
      <div v-else>
        📊 晴天无惩罚，保温直接提供 +{{ currentEggProgressMod - 100 }}% 加速
      </div>
    </div>
  </div>
</template>
