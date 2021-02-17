<template>
  <q-page class="flex justify-around items-center row app">
    <q-fab
    style="width: 5vw;"
    v-model="navMenu"
    label="Menu"
    label-position="bottom"
    glossy
    color="purple"
    icon="keyboard_arrow_right"
    direction="right"
    >
      <q-fab-action color="primary" @click="navigateTo('Parameters')" icon="build" label="Parameter settings" />
      <q-fab-action color="orange" @click="navigateTo('Backtest')" icon="bolt" label="Backtesting" />
    </q-fab>
    <div style="width: 90vw;">
      <Parameters v-if="currentPage === 'Parameters'" @add-epoch="addEpoch" @clear-epochs="clearEpochs" :epochs="epochs"/>
      <Backtest v-if="currentPage === 'Backtest'" :epochs="epochs" />
    </div>
  </q-page>
</template>

<script>

import Epoch from 'src/components/Epoch.vue';
import Parameters from 'src/components/Parameters.vue';
import Backtest from 'src/components/Backtest.vue';

export default {
  name: 'PageIndex',
  components: { Epoch, Parameters, Backtest },
  data() {
    return {
      currentPage: "Parameters",
      navMenu: false,
      epochs: [],

    }
  },
  methods: {
    navigateTo(target){
      console.log("navigating to: ", target);
      this.currentPage = target;
    },
    addEpoch(epoch){
      this.epochs.push(epoch);
      this.epochs.sort(function(a, b) {
          return a.start - b.start;
      });
      this.updateStorage();
    },
    clearEpochs(){
      this.epochs = [];
      this.updateStorage();
    },
    updateStorage(){
      localStorage.epochs = JSON.stringify(this.epochs);
      localStorage.dataPath = JSON.stringify(this.dataPath);
    },
  },
  mounted() {
    this.epochs = JSON.parse(localStorage.getItem('epochs'));
    // if there are no saved epochs this ensures it remains an array rather than null
    this.epochs = this.epochs ? this.epochs : [];
    this.dataPath = JSON.parse(localStorage.getItem('dataPath'));
  }
}
</script>

<style scoped>
.app {
  max-height: 100vh;
  max-width: 100vw;
}
</style>