<script setup>
import { Peer } from 'peerjs'
import { onMounted, computed, ref, h } from 'vue'
import { NButton, NIcon, NInput, NTag, NCollapseTransition, NSpace, useDialog, useMessage } from 'naive-ui'
import { Copy } from '@vicons/ionicons5'
import GamePlay from '@/components/GamePlay.vue'

const PEER_STATE = {
  UNAVAILABLE: 'UNAVAILABLE',
  READY: 'READY',
  PAIRING: 'PAIRING',
  WAITING: 'WAITING',
  GAMING: 'GAMING',
  UNPAIRING: 'UNPAIRING',
}

const MESSAGE_TYPE = {
  UNAVAILABLE: 'UNAVAILABLE',
  GAMING: 'GAMING',
  ACCEPTED: 'ACCEPTED',
  REJECTED: 'REJECTED',
  ACKNOWLEDGED: 'ACKNOWLEDGED',
  UNPAIRED: 'UNPAIRED',
  GAMEPLAY: 'GAMEPLAY',
}

let peer
const peerId = ref('')
const peerName = ref('')
const peerState = ref(PEER_STATE.UNAVAILABLE)
const peerChoice = ref('')
const peerScore = ref({ win: 0, draw: 0, lose: 0 })

let opponent
const opponentId = ref('')
const opponentName = ref('')
const opponentIdStatus = ref('')
const opponentChoice = ref('')
const opponentDecided = ref(false)

const gamePlayRef = ref(null)
const count = ref(0)
const inGame = computed(() => [PEER_STATE.WAITING, PEER_STATE.GAMING, PEER_STATE.UNPAIRING].includes(peerState.value))
const dialog = useDialog()
const message = useMessage()

const delay = (ms) => {
  return new Promise((resolve) => {
    setTimeout(resolve, ms)
  })
}

const initPeer = () => {
  restorePeerName()

  peer = new Peer

  // connected to the peer server
  peer.on('open', () => {
    if (peer.id === null) {
      // reconnection do not have `peer.id`, restore from `peerId.value`
      peer.id = peerId.value
    } else {
      peerId.value = peer.id
    }
    peerState.value = PEER_STATE.READY
    console.log('peer ready')
  })

  // new connection from a remote peer
  peer.on('connection', (conn) => {
    // already paried with a remote peer, reject new connection request
    if (opponent && opponent.open) {
      sendMessageSafely(conn, { type: MESSAGE_TYPE.GAMING })
      return
    }

    console.log('incoming connection established')
    dialog.warning({
      title: 'Invitation',
      content: () => h('div', { innerHTML: `Player <strong>${conn.peer}</strong> (win: ${conn.metadata.win}, draw: ${conn.metadata.draw}, lose: ${conn.metadata.lose}) invites you for a game.` }),
      positiveText: 'Accept',
      negativeText: 'Reject',
      closable: false,
      maskClosable: false,
      onPositiveClick: () => acceptInvitation(conn),
      onNegativeClick: () => rejectInvitation(conn),
    })
  })

  // lost connection to the peer server
  peer.on('disconnected', () => {
    peerState.value = PEER_STATE.UNAVAILABLE
    message.warning('Connection has lost. Reconnecting...')
    // Workaround for peer.reconnect deleting previous id
    peer.id = peerId.value
    peer._lastServerId = peerId.value
    peer.reconnect()
  })

  // peer destroyed
  peer.on('close', () => {
    peerState.value = PEER_STATE.UNAVAILABLE
    if (opponent) {
      opponent.close()
      opponent = null
    }
  })

  peer.on('error', (err) => {
    peerState.value = PEER_STATE.UNAVAILABLE
    message.error(`Error: ${err.type}`)
    if (opponent) {
      opponent.close()
      opponent = null
    }
    opponentId.value = ''
    opponentIdStatus.value = ''
    initPeer()
  })
}

const initOpponent = (type) => {
  opponent.on('open', () => {
    console.log(`${type} connection established`)
  })

  opponent.on('data', (data) => {
    switch (data.type) {
      case MESSAGE_TYPE.ACKNOWLEDGED:
        peerState.value = PEER_STATE.GAMING
        break
      case MESSAGE_TYPE.ACCEPTED:
        opponentName.value = data.display
        peerState.value = PEER_STATE.GAMING
        sendMessageSafely(opponent, { type: MESSAGE_TYPE.ACKNOWLEDGED })
        break
      case MESSAGE_TYPE.REJECTED:
        wastedGame('The player has rejected your invitation.')
        break
      case MESSAGE_TYPE.GAMING:
        wastedGame('The player is in another game.')
        break
      case MESSAGE_TYPE.UNPAIRED:
        wastedGame(() => h('div', { innerHTML: `Player <strong>${opponentName.value}</strong> has quit the game.` }))
        break
      case MESSAGE_TYPE.GAMEPLAY:
        opponentDecided.value = data.decided
        if (data.decided && data.choice && !opponentChoice.value) {
          opponentChoice.value = data.choice
          decide(peerChoice.value)
          refresh(3)
        }
        break
      default:
        break
    }
  })

  opponent.on('close', () => {
    peerState.value = PEER_STATE.READY
    opponent = null
    console.log(`${type} connection closed`)
  })
}

const acceptInvitation = (conn) => {
  opponent = conn
  opponentId.value = conn.peer
  opponentName.value = conn.metadata.display
  initOpponent('incoming')
  sendMessageSafely(conn, { type: MESSAGE_TYPE.ACCEPTED })
  peerState.value = PEER_STATE.WAITING
}

const rejectInvitation = (conn) => {
  conn.on('close', () => {
    console.log('incoming connection closed')
  })
  sendMessageSafely(conn, { type: MESSAGE_TYPE.REJECTED })
}

const sendMessage = async (conn, data) => {
  const unusableTypes = [
    MESSAGE_TYPE.UNAVAILABLE,
    MESSAGE_TYPE.GAMING,
    MESSAGE_TYPE.REJECTED,
    MESSAGE_TYPE.UNPAIRED,
  ]
  conn.send({...data, peer: peerId.value, display: peerName.value || 'Nameless' })
  if (unusableTypes.includes(data.type)) {
    await delay(500)
    conn.close()
    // conn = null
  }
}

const sendMessageSafely = (conn, data) => {
  if (conn && conn.open) {
    sendMessage(conn, data)
  } else {
    conn.on('open', () => {
      sendMessage(conn, data)
    })
  }
}

const restorePeerName = () => {
  const previousName = localStorage.getItem('peerName')
  if (previousName) {
    peerName.value = previousName
  }
}

const storePeerName = () => {
  if (peerName.value) {
    localStorage.setItem('peerName', peerName.value)
  }
}

const copyPeerId = async () => {
  await navigator.clipboard.writeText(peerId.value)
  message.info('Your peer ID has been copied to the clipboard.')
}

const wastedGame = (reason) => {
  dialog.warning({
    title: 'Wasted',
    content: reason,
    positiveText: 'OK',
    closable: false,
    maskClosable: false,
    onPositiveClick: () => {
      peerState.value = PEER_STATE.READY
      opponent = null
    },
  })
}

const startGame = () => {
  if (opponentId.value === '') {
    opponentIdStatus.value = 'error'
    message.warning('Please fill in the opponent’s connection ID.')
    return
  }

  if (opponentId.value === peerId.value) {
    opponentIdStatus.value = 'error'
    message.warning('You cannot play with yourself.')
    return
  }

  // close previous opponent connection
  if (opponent) {
    opponent.close()
    opponent = null
  }

  peerState.value = PEER_STATE.PAIRING
  opponent = peer.connect(opponentId.value, {
    reliable: true,
    metadata: {
      display: peerName.value || 'Nameless',
      ...peerScore.value
    }
  })

  initOpponent('outcoming')
}

const endGame = () => {
  peerState.value = PEER_STATE.UNPAIRING
  sendMessageSafely(opponent, { type: MESSAGE_TYPE.UNPAIRED })
  // peerState.value = PEER_STATE.READY
  // opponent = null
}

const refresh = async (s) => {
  for (let i = s; i > 0; i--) {
    count.value = i
    await delay(1000)
  }
  count.value = 0

  gamePlayRef.value.reset()
  peerChoice.value = ''
  opponentChoice.value = ''
  opponentDecided.value = false
}

const decide = (choice) => {
  if (choice) {
    peerChoice.value = choice
    sendMessageSafely(opponent, {
      type: MESSAGE_TYPE.GAMEPLAY,
      decided: !!choice,
      choice: opponentDecided.value ? choice : ''
    })
  }
}

const update = (score) => {
  const { win, draw, lose } = score
  peerScore.value.win += win
  peerScore.value.draw += draw
  peerScore.value.lose += lose
}

const checkCompatibility = () => {
  if (!!window.RTCPeerConnection && 'createDataChannel' in RTCPeerConnection.prototype) {
    initPeer()
  } else {
    dialog.error({
      title: 'Incompatible',
      content: () => h('div', { innerHTML: 'Your browser does not support <strong>WebRTC</strong>. Please upgrade and try again. Visit <a href="https://caniuse.com/rtcpeerconnection" target="_blank">Can I use</a> for more information.' }),
      positiveText: 'OK',
      closable: false,
      maskClosable: false,
    })
  }
}

onMounted(checkCompatibility)
</script>

<template>
  <n-collapse-transition :show="!inGame">
    <n-space vertical :size="16" class="bottom-gap">
      <div>
        <span>Your Display Name</span>
        <n-input
          v-model:value="peerName"
          :disabled="peerState !== PEER_STATE.READY"
          @blur="storePeerName"
        />
      </div>
      <div>
        <span>Your Peer ID</span>
        <n-input
          placeholder="Establishing connection to the peer server..."
          v-model:value="peerId"
          :loading="peerState === PEER_STATE.UNAVAILABLE"
          disabled
        >
          <template #suffix>
            <n-icon
              v-if="peerState !== PEER_STATE.UNAVAILABLE"
              style="cursor: pointer"
              :component="Copy"
              @click="copyPeerId"
            />
          </template>
        </n-input>
      </div>
      <div>
        <span>Opponent Connection ID</span>
        <n-input
          v-model:value="opponentId"
          :disabled="peerState !== PEER_STATE.READY"
          :status="opponentIdStatus"
          @input="opponentIdStatus = ''"
        />
      </div>
    </n-space>
  </n-collapse-transition>

  <n-space v-if="inGame" justify="space-between" align="center" class="bottom-gap">
    <span>You are now playing with <strong>{{ opponentName }}</strong></span>
    <n-button
      type="primary"
      :loading="peerState === PEER_STATE.UNPAIRING"
      :disabled="peerState === PEER_STATE.UNPAIRING"
      @click="endGame"
    >
      Quit
    </n-button>
  </n-space>

  <n-space v-if="!inGame" justify="flex-end" align="center" class="bottom-gap">
    <n-button
      type="primary"
      :loading="peerState === PEER_STATE.PAIRING"
      :disabled="peerState === PEER_STATE.UNAVAILABLE || peerState === PEER_STATE.PAIRING"
      @click="startGame"
    >
      Invite
    </n-button>
  </n-space>

  <n-space v-if="!inGame" justify="space-around" align="center" :size="24" vertical>
    <n-tag :bordered="false" size="large" type="success">Win: {{ peerScore.win }}</n-tag>
    <n-tag :bordered="false" size="large" type="warning">Draw: {{ peerScore.draw }}</n-tag>
    <n-tag :bordered="false" size="large" type="error">Lose: {{ peerScore.lose }}</n-tag>
  </n-space>

  <game-play
    v-if="peerState === PEER_STATE.WAITING || peerState === PEER_STATE.GAMING"
    ref="gamePlayRef"
    :waiting="peerState === PEER_STATE.WAITING"
    :opponentId="opponentId"
    :opponentName="opponentName"
    :opponentChoice="opponentChoice"
    :opponentDecided="opponentDecided"
    :count="count"
    @decide="decide"
    @update="update"
  ></game-play>
</template>

<style scoped>
.bottom-gap {
  margin-bottom: 48px;
}
</style>
