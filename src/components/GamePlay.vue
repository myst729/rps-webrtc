<script setup>
import { defineEmits, defineExpose, defineProps, computed, ref } from 'vue'
import { NIcon, NSpace } from 'naive-ui'
import { HourglassOutline, BulbOutline } from '@vicons/ionicons5'

import rock from '../assets/rock.png'
import paper from '../assets/paper.png'
import scissors from '../assets/scissors.png'

const results = ['draw', 'win', 'lose']
const options = {
  ROCK: { image: rock, value: 1 },
  PAPER: { image: paper, value: 2 },
  SCISSORS: { image: scissors, value: 3 },
}

const emits = defineEmits(['decide'])
const props = defineProps({
  opponentId: String,
  opponentName: String,
  opponentChoice: String,
  opponentDecided: Boolean,
  wait: Number,
})

const choice = ref('')
const total = ref({ win: 0, draw: 0, lose: 0 })

const result = computed(() => {
  if (choice.value && props.opponentChoice) {
    const diff = options[choice.value].value - options[props.opponentChoice].value
    return results[(diff + 3) % 3]
  }
  return ''
})

const capitalize = (str) => {
  if (str.length) {
    return str[0].toUpperCase() + str.slice(1).toLowerCase()
  }
  return ''
}

const update = () => {
  total.value[result.value]++
  choice.value = ''
}

const decide = (option) => {
  if (choice.value === '') {
    choice.value = option
    emits('decide', option)
  }
}

defineExpose({ update })
</script>

<template>
  <n-space vertical :size="12" justify="center" align="center">
    <div class="choice-field centered">
      <n-icon v-if="!opponentDecided" size="120" :component="HourglassOutline" />
      <n-icon v-if="opponentDecided && !opponentChoice" size="120" :component="BulbOutline" />
      <img
        v-if="opponentDecided && opponentChoice"
        :src="options[opponentChoice].image"
        class="choice opponent-choice"
      />
    </div>
    <div style="position: relative">
      <strong class="self-centered">{{ capitalize(result) }}{{ wait ? ` (${wait})` : ' ' }}</strong>
    </div>
    <div class="choice-field centered">
      <n-icon v-if="!choice" size="120" :component="HourglassOutline" />
      <n-icon v-if="choice && !(opponentDecided && opponentChoice)" size="120" :component="BulbOutline" />
      <img
        v-if="choice"
        :src="options[choice].image"
        :class="{ 'choice': true, blured: !(opponentDecided && opponentChoice) }"
      />
    </div>

    <n-space :size="16" justify="center" align="center">
      <img
        v-for="(option, key) in options"
        draggable="false"
        @click="decide(key)"
        :key="key"
        :src="option.image"
        :class="{ option: true, chosen: choice === key, pending: !choice }"
      />
    </n-space>
  </n-space>
  <pre>{{ JSON.stringify(total, null, 2) }}</pre>
</template>

<style scoped>
.self-centered {
  font-size: 2rem;
  width: 240px;
  text-align: center;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

.choice-field {
  width: 240px;
  height: 240px;
  position: relative;
}

.choice {
  width: 200px;
  height: 200px;
  user-select: none;
  position: absolute;
}

.opponent-choice {
  transform: rotate(180deg);
}

.blured {
  filter: blur(5px);
  opacity: .5;
}

.option {
  width: 80px;
  height: 80px;
  background-color: rgba(255, 255, 255, .25);
  border: 4px solid rgba(255, 255, 255, .25);
  border-radius: 20px;
  cursor: not-allowed;
  user-select: none;
  transition: border-color .3s cubic-bezier(.4, 0, .2, 1);
}

.pending {
  cursor: pointer;
}

.pending:hover {
  border-color: #36ad6a;
}

.pending:active {
  border-color: #0c7a43;
}

.chosen {
  border-color: #18a058;
}
</style>
