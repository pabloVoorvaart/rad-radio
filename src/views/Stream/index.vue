<script setup lang="ts">
import { onMounted, ref } from "vue";
import { useEthers } from "vue-dapp";
import DonateButton from "../../components/DonateButton.vue";
import DonateModal from "../../components/DonateModal.vue";
import LensButton from "../../components/LensButton.vue";
import LensModal from "../../components/LensModal.vue";
import VideoPlayer from "../../components/VideoPlayer.vue";
import LayoutHeader from "../../components/LayoutHeader.vue";
import useLiveDoc from "../../db/useLiveDoc";
import useRealtimeDbValue from "../../db/useRealtimeDbValue";
import { increment } from "@firebase/firestore";
import { increment as rtDbIncrement } from "@firebase/database";
import bus from "./reactionEventBus";
import { Reaction, emojiMap } from "./reactions";
import ReactionFountain from "../../components/ReactionFountain.vue";
import Chill from "../../components/Chill.vue";
import useTokenBalances, { Token } from "./use/tokenBalances";
import { formatEther, parseEther } from "@ethersproject/units";

const modalShown = ref(false);
const lensModalShown = ref(false);

const { isActivated } = useEthers();

const donate = async () => {
  modalShown.value = true;
};

const lens = async () => {
  lensModalShown.value = true;
};

const closeModal = () => {
  modalShown.value = false;
};

const closeLensModal = () => {
  lensModalShown.value = false;
};

type Reactions = {
  [key in Reaction]: number;
};

const reactions = useLiveDoc<Reactions>("reactions", "reactions");

const THROTTLE_PERIOD_SECS = 5;
const THROTTLE_TIMEOUT_SECS = 10;
const throttled = ref(false);
let reactionThrottleCount: number = 0;
let throttleResetInterval: ReturnType<typeof setTimeout> | undefined;

function updateThrottle() {
  reactionThrottleCount++;

  clearInterval(throttleResetInterval);
  throttleResetInterval = setTimeout(() => {
    reactionThrottleCount = 0;
  }, THROTTLE_TIMEOUT_SECS * 1000);

  if (reactionThrottleCount >= 10) {
    throttled.value = true;

    setTimeout(() => {
      reactionThrottleCount = 0;
      throttled.value = false;
    }, THROTTLE_PERIOD_SECS * 1000);
  }
}

function react(key: Reaction) {
  updateThrottle();

  if (throttled.value) return;

  reactions.update({
    [key]: increment(1),
  });
}

reactions.registerListener((doc, prevDoc) => {
  if (!prevDoc) return;

  let increasedReaction: Reaction | undefined = undefined;

  for (const reaction in doc) {
    const prevAmount = prevDoc[reaction as Reaction];

    if (prevAmount < doc[reaction as Reaction]) {
      increasedReaction = reaction as Reaction;
    }
  }

  if (increasedReaction) bus.emit("reaction", increasedReaction);
});

function getReactionCount(reaction: Reaction) {
  return reactions.doc?.value?.[reaction];
}

const viewCounter = useRealtimeDbValue<number>("viewersCount");

/*
  The Alchemy token is domain-restricted, so it's safe to embed
  it client-side.
*/
const { balances: tokenBalances } = useTokenBalances(
  import.meta.env.VITE_ALCHEMY_TOKEN,
  "0x1665f71Ca63cd880A6294a16F4342485bAc12A46"
);

const currentlyVisibleTokenBalance = ref<
  { token: Token; formattedBalance: string } | undefined
>();
let currentlyVisibleTokenIndex = 0;

function cycleCurrentlyVisibleToken() {
  if (!tokenBalances.value || Object.keys(tokenBalances.value).length === 0)
    return;

  const [tokenName, balance] = Object.entries(tokenBalances.value)[
    currentlyVisibleTokenIndex
  ];
  currentlyVisibleTokenBalance.value = {
    token: tokenName as Token,
    formattedBalance: (+formatEther(balance)).toFixed(2),
  };

  currentlyVisibleTokenIndex = Object.keys(tokenBalances.value)[
    currentlyVisibleTokenIndex + 1
  ]
    ? currentlyVisibleTokenIndex + 1
    : 0;
}

onMounted(() => {
  viewCounter.set(rtDbIncrement(1));
  viewCounter.setOnDisconnect(rtDbIncrement(-1));

  setInterval(cycleCurrentlyVisibleToken, 2000);
});
</script>

<template>
  <Chill v-if="throttled" />
  <LayoutHeader />
  <ReactionFountain />
  <div id="spinners">
    <a id="spinner1" href="https://hoerberlin.com/" target="_blank"></a>
    <a
      id="spinner3"
      class="hideOnMobile"
      href="https://livepeer.org/"
      target="_blank"
    ></a>
    <a
      id="spinner4"
      class="hideOnMobile"
      href="https://radicle.xyz/"
      target="_blank"
    ></a>
  </div>

  <div id="tribals">
    <div id="logo">RAD RADIO</div>
    <div id="tribal2"></div>
    <div id="tribal3"></div>
  </div>

  <div class="stats">
    <div class="donated">
      <div v-if="currentlyVisibleTokenBalance">
        <span class="count"
          >{{ currentlyVisibleTokenBalance.formattedBalance }}
          {{ currentlyVisibleTokenBalance.token }}</span
        >
      </div>
      <div v-else>
        <span class="count">---</span>
      </div>
      <span class="label">DONATED</span>
    </div>
    <div class="online">
      <span class="count" v-if="viewCounter.val">{{ viewCounter.val }}</span>
      <span class="label">WATCHING NOW</span>
    </div>
  </div>

  <div class="reaction-buttons">
    <div
      class="reaction"
      @click="() => react(reaction)"
      :class="{ throttled }"
      v-for="(emoji, reaction) in emojiMap"
    >
      <span>{{ emoji }}</span>
      <span class="count" v-if="reactions.doc?.value">{{
        getReactionCount(reaction)
      }}</span>
    </div>
  </div>

  <div class="modal">
    <DonateModal @close="closeModal" :show="modalShown" />
    <LensModal @close="closeLensModal" :show="lensModalShown" />
  </div>

  <div class="stream">
    <VideoPlayer />
    <DonateButton
      class="donate-button"
      @click="donate"
      v-if="isActivated && !modalShown"
    />
    <LensButton
      class="lens-button"
      @click="lens"
      v-if="isActivated && !lensModalShown"
    />
  </div>
</template>

<style scoped>
.stream {
  position: fixed;
  height: 100vh;
  width: 100vw;
  display: flex;
  z-index: -1;
}

.modal {
  z-index: 10;
}

.throttled {
  opacity: 0.5;
}

.stats {
  position: fixed;
  bottom: 128px;
  border: 2px solid red;
  border-radius: 16px;
  left: 50%;
  transform: translateX(-50%);
  color: red;
  display: flex;
  font-feature-settings: "ss02" off;
}

.stats > div {
  padding: 8px;
  display: flex;
  flex-direction: column;
  align-items: center;
  white-space: nowrap;
}

.stats .count {
  font-size: 24px;
  text-transform: uppercase;
}

.stats > div {
  min-width: 170px;
}

.stats > div:not(:last-child) {
  border-right: 2px solid red;
}

.reactions {
  display: flex;
  gap: 8px;
}

.reaction-buttons {
  position: fixed;
  z-index: 100;
  bottom: 2vh;
  left: 50vw;
  transform: translateX(-50%);
  display: flex;
  font-feature-settings: "ss02" off;
}

.reaction {
  display: flex;
  width: 12vw;
  max-width: 64px;
  flex-direction: column;
  gap: 4px;
  align-items: center;
  transition: transform 0.3s, opacity 0.3s;
  transform-origin: 50% 100%;
  user-select: none;
  cursor: pointer;
}

.reaction:not(.throttled):hover {
  transform: scale(1.3);
}

.reaction:not(.throttled):active {
  transform: scale(1.4);
}

.reaction > .count {
  font-size: 16px;
  color: red;
}

.reaction-buttons > div {
  font-size: 32px;
}

@media (max-width: 691px) {
  #logo {
    display: none;
  }

  .hideOnMobile {
    display: none;
  }

  .reaction:not(.throttled):hover {
    transform: scale(1);
  }
}
</style>
