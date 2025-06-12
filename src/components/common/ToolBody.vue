<template>
  <div class="card p-3">
    <div class="d-flex justify-content-between align-items-center">
      <div>
        <h3>Rescheduler</h3>
      </div>
      <div v-if="tooltipContent.length > 0" v-tooltip="{
        content: tooltipContent,
        html: true,
        hideTriggers: ['click'],
        autoHide: true
      }" class="clickable">
        <h3>ⓘ</h3>
      </div>
    </div>
    <TendoRescheduler />
  </div>
</template>

<script>
import TendoRescheduler from '../TendoRescheduler.vue'

export default {
  components: {
    TendoRescheduler
  },
  data() {
    return {
      tooltip: {
        old: '',
        guide: ''
      }
    }
  },
  computed: {
    tooltipContent() {
      return (this.tooltip.guide ? `Guide: <a href="${this.tooltip.guide}" target="_blank">Here</a><br>` : '') + (this.tooltip.old ? `Legacy: <a href="${this.tooltip.old}" target="_blank">Here</a>` : '')
    }
  }
}
</script>

<style lang="scss" scoped>
.card {
  -webkit-box-shadow: 5px 5px 15px 5px rgba(0, 0, 0, 0.3);
  box-shadow: 5px 5px 15px 5px rgba(0, 0, 0, 0.3);
}
</style>
