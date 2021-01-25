<template>
  <div id="app">
    <div class="day-counter">
      Day {{ daysPassed }}
    </div>
    <div class="stats-box">
      <div class="stats-unaffected">
        <span>Unaffected</span>
        <div>{{ unaffected }}</div>
      </div>
      <div class="stats-infected">
        <span>Infected</span>
        <div>{{ stats.infected }}
        {{ stats.infected > 300 ? 'deadliness x3' :
          stats.infected > 100 ? 'deadliness x2' : ''
          }}
        </div>
      </div>
      <div class="stats-dead">
        <span>Dead</span>
        <div>{{ stats.dead }}</div>
      </div>
      <div class="stats-cured">
        <span>Cured</span>
        <div>{{ stats.cured }}</div>
      </div>
      <div class="stats-vaccinated">
        <span>Vaccinated</span>
        <div>{{ stats.vaccinated }}</div>
      </div>

    </div>

    <div class="control-box">
      <div class="control-item">
        <div class="control-label">Infected start amount (flat number)</div>
        <input class="control-input" type="number" v-model="settings.infectedStartAmount">
      </div>

      <div class="control-item">
        <div class="control-label">Daily infection chance (0-100)</div>
        <input class="control-input" type="number" v-model="settings.dailyInfectionChance">
      </div>

      <div class="control-item">
        <div class="control-label">Daily death chance (0-100)</div>
        <input class="control-input" type="number" v-model="settings.dailyDeathChance">
      </div>

      <div class="control-item">
        <div class="control-label">Infection duration (flat number)</div>
        <input class="control-input" type="number" v-model="settings.infectedForDays">
      </div>

      <div class="control-item">
        <div class="control-label">Starting percentage vaccinated (0-100)</div>
        <input class="control-input" type="number" max="100" v-model="settings.vaccinatedPercentage">
      </div>

      <div class="control-item">
        <div class="control-label">Chance of being vaccined per day (0-100)</div>
        <input class="control-input" type="number" max="100" v-model="settings.dailyVaccineChance">
      </div>

      <div class="control-item">
        <div class="control-label">Stop after days (0 for infinite)</div>
        <input class="control-input" type="number" v-model="settings.daysToRun">
      </div>

      <div class="control-item">
        <button @click="resetConfig">Reset config</button>
      </div>
    </div>

    <div class="game-box">
      <div class="row" v-for="row in blocks">
        <div class="block"
             v-for="block in row"
             :class="block.state === 'infected' ? 'block-infected' :
             block.state === 'cured' ? 'block-cured' :
             block.state === 'vaccinated' ? 'block-vaccinated' :
             block.state === 'dead' ? 'block-dead' : ''"
        >

        </div>
      </div>
    </div>

    <div v-if="!simulationRunning" class="start-game-button" @click="startGame">
      Start!
    </div>
    <div v-else class="start-game-button" @click="stopSimulation">
      Stop
    </div>
  </div>
</template>

<script>
export default {
  name: 'App',
  data() {
    return {
      stats: {
        totalAffected: 0,
        cured: 0,
        infected: 0,
        dead: 0,
        vaccinated: 0,
      },
      simulationInterval: null,
      simulationRunning: false,
      blocks: [],
      daysPassed: 0,
      settings: {
        gridSize: 100,
        infectedStartAmount: 5,
        dailyInfectionChance: 5,
        dailyDeathChance: 0.3,
        dailyVaccineChance: 0,
        infectedForDays: 14,
        daysToRun: 0,
        vaccinatedPercentage: 30,
      }
    }
  },
  computed: {
    unaffected() {
      return (this.settings.gridSize * this.settings.gridSize) - this.stats.totalAffected;
    }
  },
  mounted() {
    this.generateBlocks();
  },
  methods: {
    generateBlocks() {
      let id = 0;
      this.blocks = [];
      const allIds = [];
      for (let i = 0; i < this.settings.gridSize; i++) {
        const blocks = [];
        for (let i2 = 0; i2 < this.settings.gridSize; i2++) {
          blocks.push({
            id,
            x: i2,
            y: i,
            state: 'normal',
            daysLeftInfected: 0,
          })
          allIds.push({
            id,
            x: i2,
            y: i,
          });
          id++;
        }
        this.blocks.push(blocks);
      }
      return allIds;
    },
    startGame() {
      this.simulationRunning = true;
      let healthyIds = this.generateBlocks();
      this.daysPassed = 0;
      this.stats = { totalAffected: 0,
          cured: 0,
          infected: 0,
          dead: 0,
          vaccinated: 0
      }
      let infectedIds = [];
      console.log('starting');
      for (let infected = 0; infected < this.settings.infectedStartAmount; infected++) {
        let infectId = Math.floor(Math.random() * (this.settings.gridSize * this.settings.gridSize));
        while (infectedIds.includes(infectId)) {
          infectId = Math.floor(Math.random() * (this.settings.gridSize * this.settings.gridSize));
        }
        let infectX;
        let infectY;
        this.blocks.forEach((row) => {
          const target = row.find((block) => block.id === infectId);
          if (target) {
            infectX = target.x;
            infectY = target.y;
            infectedIds.push(infectId);
          }
        })

        this.infectBlock(infectX, infectY, true);
      }

      healthyIds = healthyIds.filter((block) => !infectedIds.includes(block.id));

      const vacinateAmount = Math.floor((this.settings.gridSize * this.settings.gridSize) / 100 * this.settings.vaccinatedPercentage);
      for (let vaccinated = 0; vaccinated < vacinateAmount; vaccinated++) {
        const vaccinateIndex = Math.floor(Math.random() * (healthyIds.length - 1));
        this.vaccinateBlock(healthyIds[vaccinateIndex]);
        healthyIds.splice(vaccinateIndex, 1);
      }

      this.simulationInterval = setInterval(() => {
        if (((this.daysPassed < this.settings.daysToRun) || !Number(this.settings.daysToRun)) && this.stats.infected) {
          this.playRound();
        } else {
          clearInterval(this.simulationInterval);
          this.simulationRunning = false;
        }
      }, 200);
    },
    stopSimulation() {
      this.simulationRunning = false;
      if (this.simulationInterval) {
        clearInterval(this.simulationInterval);
        this.simulationInterval = null;
      }
      this.generateBlocks();
      this.daysPassed = 0;
      this.stats = { totalAffected: 0,
        cured: 0,
        infected: 0,
        dead: 0,
        vaccinated: 0
      }
    },
    playRound() {
      console.log('starting day');
      this.blocks.forEach(row => {
        row.forEach(block => {
          if (block.state === 'infected') {

            if(this.willInfectedDie(block)) {
              return;
            }
            let target;
            let targetRow;

            // above row
            targetRow = this.blocks[block.y - 1] ? this.blocks[block.y - 1] : this.blocks[this.blocks.length - 1];
            target = targetRow[block.x - 1] ? targetRow[block.x - 1] : targetRow[this.settings.gridSize - 1];
            this.maybeInfect(block, target);

            target = targetRow[block.x] ;
            this.maybeInfect(block, target);

            target = targetRow[block.x + 1] ? targetRow[block.x + 1] : targetRow[0];
            this.maybeInfect(block, target);

            // same row
            targetRow = this.blocks[block.y];
            target = targetRow[block.x - 1] ? targetRow[block.x - 1] : targetRow[this.settings.gridSize - 1];
            this.maybeInfect(block, target);

            target = targetRow[block.x + 1] ? targetRow[block.x + 1] : targetRow[0];
            this.maybeInfect(block, target);

            // below row
            targetRow = this.blocks[block.y + 1] ? this.blocks[block.y + 1] : this.blocks[0];
            target = targetRow[block.x - 1] ? targetRow[block.x - 1] : targetRow[this.settings.gridSize - 1];
            this.maybeInfect(block, target);

            target = targetRow[block.x];
            this.maybeInfect(block, target);

            target = targetRow[block.x + 1] ? targetRow[block.x + 1] : targetRow[0];
            this.maybeInfect(block, target);

            if (block.daysLeftInfected > 0) {
              block.daysLeftInfected -= 1;
            } else {
              this.$set(this.blocks[block.y], block.x, {
                ...this.blocks[block.y][block.x],
                state: 'cured'
              });
              this.stats.cured++;
              this.stats.infected--;
            }
          }

          if (block.state === 'normal') {
            if (Math.random() * 100 < this.settings.dailyVaccineChance) {
              this.vaccinateBlock(block);
            }
          }

        })
      });
      this.daysPassed++;
    },
    maybeInfect(attacker, target) {
      if (target.state === 'normal' && (attacker.infectedOnDay !== this.daysPassed)) {
        if (Math.random() * 100 < this.settings.dailyInfectionChance) {
          this.infectBlock(target.x, target.y);
        }
      }
    },
    infectBlock(x, y, instant = false) {
      this.$set(this.blocks[y], x, {
        ...this.blocks[y][x],
        state: 'infected',
        daysLeftInfected: this.settings.infectedForDays,
        infectedOnDay: instant ? -1 : this.daysPassed,
      });
      this.stats.infected++;
      this.stats.totalAffected++;
    },
    vaccinateBlock(block) {
      this.$set(this.blocks[block.y], block.x, {
        ...this.blocks[block.y][block.x],
        state: 'vaccinated',
      });
      this.stats.vaccinated++;
      this.stats.totalAffected++;
    },
    willInfectedDie(block) {
      if(Math.random() * 100 < (this.stats.infected > 300 ? this.settings.dailyDeathChance * 3 :
        this.stats.infected > 100 ? this.settings.dailyDeathChance * 2 :
        this.settings.dailyDeathChance
      )) {
        this.$set(this.blocks[block.y], block.x, {
          ...this.blocks[block.y][block.x],
          state: 'dead',
          daysLeftInfected: this.settings.infectedForDays,
        });
        this.stats.infected--;
        this.stats.dead++;
        return true;
      }
      return false;
    },
    resetConfig() {
      this.settings = {
        gridSize: 100,
        infectedStartAmount: 5,
        dailyInfectionChance: 5,
        dailyDeathChance: 0.3,
        dailyVaccineChance: 0,
        infectedForDays: 14,
        daysToRun: 0,
        vaccinatedPercentage: 30,
      }
    }
  }
}
</script>

<style>

body {
  margin: 0;
  padding: 0;
}
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
  min-height: 100vh;
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 0;
  padding-bottom: 30px;
  margin: 0;
  flex-direction: column;
}

.game-box {
}

.row {
  display: flex;
}

.block {
  width: 5px;
  height: 5px;
  border-radius: 50%;
  margin: 1px;
  background-color: #3232c4;
}

.block-infected {
  background-color: red;
}

.block-cured {
  background-color: #9b3e9b;
}

.block-vaccinated {
  background-color: #00ff00;
}

.block-dead {
  background-color: #191919;
}

.start-game-button {
  padding: 6px 12px;
  background-color: red;
  color: #fff;
  text-align: center;
  margin-top: 24px;
  cursor: pointer;
}

.day-counter {
  font-size: 36px;
  color: white;
  margin-bottom: 24px;
}

.stats-box {
  display: flex;
  justify-content: center;
  margin-bottom: 10px;
}

.stats-infected {
  color: red;
  margin-right: 30px;
}

.stats-unaffected {
  color: #3232c4;
  margin-right: 30px;
}

.stats-dead {
  color: #191919;
  margin-right: 30px;
}

.stats-cured {
  color: #9b3e9b;
  margin-right: 30px;
}

.stats-vaccinated {
  color: #00ff00;
}

.control-box {
  position: fixed;
  width: 400px;
  height: 100vh;
  left: 0;
  top: 0;
  display: flex;
  flex-direction: column;
  justify-content: center;
}

.control-item {
  padding: 8px;
  text-align: left;
  color: #fff;
  font-size: 16px;
}

.control-label {
  margin-bottom: 4px;
  color: #191919;
}

</style>
