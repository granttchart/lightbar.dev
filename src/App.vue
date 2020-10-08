<template>
  <main
    class="container"
    id="emdr-app"
    style="{ '--interval': intervalSeconds + 's' }"
    @keyup.space.prevent="toggleStart"
    tabindex="-1"
  >
    <form
      :class="{ dim: started }"
      class="controls card-body row justify-content-end align-items-center mt-3 rounded"
    >
      <div
        class="col fade-in time-elapsed hide-on-dim"
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

      <div class="col-md-3 hide-on-dim">
        <div class="form-check">
          <input
            v-model="audio.enabled"
            type="checkbox"
            class="form-check-input"
            id="audio-control"
          />
          <label for="audio-control" class="form-check-label"
            >Enable audio</label
          >
        </div>
      </div>

      <div class="col-md-3 hide-on-dim">
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

      <div class="col-md-3">
        <button
          type="button"
          title="Click or press the spacebar to begin"
          class="w-100 font-weight-bold d-block btn btn-lg btn-outline-light"
          v-on:click="toggleStart()"
        >
          <span v-if="started">Stop</span>
          <span v-if="!started">Start</span>
        </button>
      </div>
    </form>

    <article class="color-ball-parent" v-if="started">
      <div class="col-12">
        <div class="color-ball"></div>
      </div>
    </article>
  </main>
</template>

<script type="typescript">
import { defineComponent, watch, toRefs, reactive } from "vue";

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
        context: new (window.AudioContext || window.webkitAudioContext)(),
        oscillator: undefined,
      },
    });

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
      state.audio.oscillator = state.audio.context.createOscillator();

      state.audio.oscillator.type = "sine";
      // value in hertz
      state.audio.oscillator.frequency.setValueAtTime(
        130,
        state.audio.context.currentTime
      );
      state.audio.oscillator.connect(state.audio.context.destination);
      state.audio.oscillator.start();
      state.audio.active = true;
    };

    const stopAudio = () => {
      if (state.audio.oscillator) {
        state.audio.oscillator.stop();
        state.audio.active = false;
      }
    };

    // Pad numbers less than 10 with a leading zero.
    const applyLeadingZero = (value) => (value < 10 ? `0${value}` : value);

    watch("state.audio", (val) => {
      if (!val) {
        stopAudio();
      } else if (state.started && !state.audio.active) {
        startAudio();
      }
    });

    return {
      ...toRefs(state),
      toggleStart,
      startTimer,
      clearTimer,
      startAudio,
      stopAudio,
      applyLeadingZero,
    };
  },
});
</script>

<style lang="sass" scoped>
@import '../node_modules/bootstrap/scss/bootstrap.scss'

$animation-length: .5s

body
  background-color: #222
  color: #fff
  font-size: 14px
  font-family: Helvetica, sans-serif

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

.color-ball
  position: absolute
  background-color: #68d4f8
  width: 12em
  height: 12em
  border-radius: 50%
  animation: slide var(--interval) infinite ease-in-out
</style>
