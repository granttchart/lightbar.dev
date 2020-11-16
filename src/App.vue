<template>
  <main
    class="container m-auto flex flex-col"
    id="emdr-app"
    :style="{ '--interval': intervalSeconds + 's' }"
    @keyup.space.prevent="toggleStart"
    tabindex="-1"
  >
    <div
      :class="{ dim: started }"
      class="controls mt-3 rounded grid align-items p-2 grid grid-flow-col auto-cols-max justify-between"
    >
      <div
        class="fade-in time-elapsed hide-on-dim"
        v-if="started"
        aria-live="polite"
      >
        <div class="form-group">
          <label>Time elapsed</label>
          <div class="text-monospace time">
            <span v-if="sessionTimer.minutesElapsed > 0">{{
              sessionTimer.minutesElapsed
            }}</span
            >:
            <span>{{ sessionTimer.secondsElapsed }}</span>
          </div>
        </div>
      </div>

      <div class="hide-on-dim">
        <div class="form-check">
          <input
            v-model="audio.enabled"
            @change="toggleAudio"
            type="checkbox"
            class="form-check-input"
            id="audio-control"
          />
          <label for="audio-control" class="form-check-label">
            Enable audio
          </label>
        </div>
      </div>

      <div class="hide-on-dim">
        <div class="form-group">
          <label for="speed-control">
            Cycle length ({{ intervalSeconds }}s)
          </label>
          <input
            v-model="intervalSeconds"
            type="range"
            class="form-control"
            id="speed-control"
            min="1"
            max="6"
          />
        </div>
      </div>

      <div class="button-container">
        <button
          type="button"
          title="Click or press the spacebar to begin"
          class="start-stop font-semibold py-2 px-4 rounded"
          v-on:click="toggleStart()"
        >
          <span v-if="started">stop</span>
          <span v-if="!started">start</span>
        </button>
      </div>
    </div>

    <article class="color-ball-parent" v-if="started">
      <div class="color-ball"></div>
    </article>
  </main>
</template>

<script type="typescript">
import { defineComponent, toRefs, reactive } from "vue";

export default defineComponent({
  name: "App",
  setup: () => {
    const state = reactive({
      started: false,
      intervalSeconds: 3, // Animation duration in seconds.
      sessionTimer: {
        secondCounter: undefined, // JavaScript interval for session length calculation.
        secondsElapsed: 0,
        minutesElapsed: 0,
      },
      audio: {
        enabled: true,
        active: false,
      },
    });
    
    const audioService = {
      context: new (window.AudioContext || window.webkitAudioContext)(),
      oscillator: undefined,
    };

    const toggleStart = () => {
      state.started = !state.started;

      if (state.started) {
        startTimer();

        if (state.audio.enabled) {
          startAudio();
        }
      } else {
        clearTimer();

        if (state.audio.enabled) {
          stopAudio();
        }
      }
    };

    const toggleAudio = () => {
      if (state.audio.enabled) {
        stopAudio();
      } else if (state.started && !state.audio.active) {
        startAudio();
      }
    };

    const startTimer = () => {
      // Each second, calculate time elapsed.
      state.sessionTimer.secondCounter = setInterval(() => {
        ++state.sessionTimer.secondsElapsed;

        if (state.sessionTimer.secondsElapsed === 60) {
          state.sessionTimer.secondsElapsed = 0;
          ++state.sessionTimer.minutesElapsed;
        }
      }, 1000);
    };

    const clearTimer = () => {
      // Clear existing timer and reset user-facing time elapsed.
      clearInterval(state.sessionTimer.secondCounter);
      state.sessionTimer.secondsElapsed = state.sessionTimer.minutesElapsed = 0;
    };

    const startAudio = () => {
      // create Oscillator node
      audioService.oscillator = audioService.context.createOscillator();

      audioService.oscillator.type = "sine";
      // value in hertz
      audioService.oscillator.frequency.setValueAtTime(
        130,
        audioService.context.currentTime
      );
      audioService.oscillator.connect(audioService.context.destination);
      audioService.oscillator.start();
      state.audio.active = true;
    };

    const stopAudio = () => {
      if (audioService.oscillator) {
        audioService.oscillator.stop();
        state.audio.active = false;
        delete audioService.oscillator;
      }
    };

    return {
      ...toRefs(state),
      toggleStart,
      startTimer,
      clearTimer,
      toggleAudio,
    };
  },
});
</script>

<style lang="sass" scoped>
@import 'sass/variables'

@keyframes slide
  0%
    left: 0

  50%
    left: calc(100% - 12em)

  100%
    left: 0

@keyframes fadeIn
  0%
    opacity: 0

  100%
    opacity: 100%

.fade-in
  animation: fadeIn $animation-length linear

.dim
  transition: opacity $animation-length
  opacity: .6

main
  min-height: 100vh

  .controls
    background: linear-gradient(320deg, rgba(167,140,233,1) 41%, rgba(175,110,213,1) 100%)
    transition: padding $animation-length

    .button-container
      .start-stop
        background-color: $blush
        transition: background-color $animation-length
        &:hover, &:active, &:focus
          background-color: darken($blush, 10%)

    .time-elapsed .time
      font-size: 1.1em
      line-height: 1.5

    .hide-on-dim
      visibility: hidden
      opacity: 0
      transition: opacity $animation-length

    &:hover
      .hide-on-dim
        visibility: visible
        opacity: 1

    &.dim
      padding-bottom: .25em
      padding-top: .25em
      transition: opacity $animation-length

      &:hover
        opacity: 1

.color-ball-parent
  position: relative
  margin: auto 0
  .color-ball
    position: absolute
    background-color: #68d4f8
    width: 12em
    height: 12em
    border-radius: 50%
    animation: slide var(--interval) infinite ease-in-out
</style>
