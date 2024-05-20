<script setup>
import { defineEmits, defineExpose, defineProps, computed, ref } from 'vue'
import { NIcon, NSpace, NTag } from 'naive-ui'
import { HourglassOutline, BulbOutline } from '@vicons/ionicons5'

import rock from '@/assets/rock.png'
import paper from '@/assets/paper.png'
import scissors from '@/assets/scissors.png'

const results = ['draw', 'win', 'lose']
const options = {
  ROCK: { image: rock, value: 1 },
  PAPER: { image: paper, value: 2 },
  SCISSORS: { image: scissors, value: 3 },
}

const emits = defineEmits(['decide', 'update'])
const props = defineProps({
  opponentId: String,
  opponentName: String,
  opponentChoice: String,
  opponentDecided: Boolean,
  waiting: Boolean,
  count: Number,
})

const choice = ref('')
const score = ref({ win: 0, draw: 0, lose: 0 })

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

const reset = () => {
  score.value[result.value]++
  choice.value = ''
  emits('update', score.value)
}

const decide = (option) => {
  if (choice.value === '') {
    choice.value = option
    emits('decide', option)
  }
}

defineExpose({ reset })
</script>

<template>
  <n-space vertical :size="12" justify="center" align="center">
    <div :class="{ 'choice-field': true, 'centered': true, waiting }">
      <n-icon v-if="!opponentDecided" size="48" :component="HourglassOutline" />
      <n-icon v-if="opponentDecided && !opponentChoice" size="48" :component="BulbOutline" />
      <img
        v-if="opponentDecided && opponentChoice"
        :src="options[opponentChoice].image"
        class="choice opponent-choice"
      />
    </div>
    <div style="position: relative">
      <strong v-if="waiting" class="self-centered">Waiting</strong>
      <strong v-if="!waiting" class="self-centered">{{ capitalize(result) }}{{ count ? ` (${count})` : ' ' }}</strong>
    </div>
    <div :class="{ 'choice-field': true, 'centered': true, waiting }">
      <n-icon v-if="!choice" size="48" :component="HourglassOutline" />
      <n-icon v-if="choice && !(opponentDecided && opponentChoice)" size="48" :component="BulbOutline" />
      <img
        v-if="choice"
        :src="options[choice].image"
        :class="{ 'choice': true, blurred: !(opponentDecided && opponentChoice) }"
      />
    </div>

    <n-space :size="16" justify="center" align="center" :class="{ 'options-field': true, waiting }">
      <img
        v-for="(option, key) in options"
        draggable="false"
        @click="decide(key)"
        :key="key"
        :src="option.image"
        :class="{ option: true, chosen: choice === key, pending: !choice }"
      />
    </n-space>

    <n-space :size="12" :class="{ 'score-field': true, waiting }">
      <n-tag :bordered="false" type="success">Win: {{ score.win }}</n-tag>
      <n-tag :bordered="false" type="warning">Draw: {{ score.draw }}</n-tag>
      <n-tag :bordered="false" type="error">Lose: {{ score.lose }}</n-tag>
    </n-space>
  </n-space>
</template>

<style scoped>
.waiting {
  filter: blur(10px);
}

.choice-field,
.options-field,
.score-field {
  transition: filter 1s cubic-bezier(.4, 0, .2, 1);
}

.self-centered {
  font-size: 1.2rem;
  width: 160px;
  text-align: center;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

.choice-field {
  width: 160px;
  height: 160px;
  position: relative;
}

.choice {
  width: 96px;
  height: 96px;
  user-select: none;
  position: absolute;
}

.opponent-choice {
  transform: rotate(180deg);
}

.blurred {
  filter: blur(5px);
  opacity: .5;
}

.option {
  padding: 8px;
  width: 64px;
  height: 64px;
  background-color: rgba(200, 200, 200, .25);
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

.chosen {
  border-color: #18a058;
}
</style>
