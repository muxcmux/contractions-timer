<template>
  <div class="timer">
    <h1>Contraction Timer</h1>
    <input type="text" class="counter" v-model="formattedTimer" readonly>
    <button class="button stop" v-if="isRunning" @click="stopContraction()">Stop Contraction</button>
    <button class="button" v-if="!isRunning" @click="startContraction()">Start Contraction</button>

    <table class="table" v-if="anyContractions">
      <thead>
        <tr>
          <th>Duration</th>
          <th>Time apart</th>
          <th>Start &amp; End time</th>
          <th>&nbsp;</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(con, index) in contractions" v-if="con">
          <td><span v-if="con.duration">{{ con.duration }}</span></td>
          <td>
            <span v-if="con.timeSinceLast">{{ con.timeSinceLast }}</span>
            <span v-else>-</span>
          </td>
          <td>{{ con.startedAt | moment('HH:MM') }} - {{ con.finishedAt | moment('HH:MM') }}</td>
          <td><button @click="remove(con.id, index)" >&#9747;</button></td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
import moment from 'moment';

function getDuration(one, two) {
  const ms = moment(one).diff(moment(two));
  return formatDuration(ms);
}

function formatDuration(ms, hours = true) {
  const d = moment.duration(ms);
  if (hours) return `${d.hours()}h ${d.minutes()}m ${d.seconds()}s`;
  return `${d.minutes()}m ${d.seconds()}s`;
}

if (!window.indexedDB) {
  window.alert("Your browser doesn't support a stable version of IndexedDB.")
}

const contractionData = []
let db;
let request = window.indexedDB.open("contractionDb", 4);

request.onerror = function(event) {
  console.log("error: ");
};

request.onsuccess = function(event) {
  db = request.result;
  console.log("success: "+ db);
};

request.onupgradeneeded = function(event) {
  let db = event.target.result;
  let objectStore = db.createObjectStore("contraction", {keyPath: "id"});

  for (let i in contractionData) {
    objectStore.add(contractionData[i]);
  }
}

export default {
  name: 'ContractionTimer',
  data() {
    return {
      timeout:   null,
      isRunning: false,
      timer:     0,
      contraction: {
        id:            null,
        duration:      null,
        startedAt:     null,
        finishedAt:    null,
        timeSinceLast: null,
      },
      contractions: [],
      retryCount: 0,
    }
  },

  computed: {
    lastContraction() {
      const c = this.contractions[this.contractions.length - 1];
      if (c === undefined) {
        return this.contractions[this.contractions.length - 2];
      }
      return c;
    },

    formattedTimer() {
      return formatDuration(this.timer * 1000);
    },

    anyContractions() {
      return !!this.contractions.filter((c) => !!c).length;
    },
  },

  mounted() {
    this.readAll();
  },

  methods: {
    startContraction() {
      this.isRunning = true;
      this.contraction.id = this.contraction.id != null ? this.contraction.id : this.contractions.length++
      this.contraction.startedAt = new Date();
      if (this.lastContraction && this.lastContraction.finishedAt) {
        this.contraction.timeSinceLast = getDuration(new Date(), this.lastContraction.finishedAt);
      }
      this.count();
    },

    stopContraction() {
      this.contraction.finishedAt = new Date();
      this.contraction.duration = formatDuration(this.timer * 1000, false);
      this.add();
    },

    count() {
      this.timeout = setTimeout(() => {
        this.timer += 1;
        if (this.isRunning) this.count();
      }, 1000);
    },

    add() {
      clearTimeout(this.timeout);
      let request = db.transaction(["contraction"], "readwrite")
      .objectStore("contraction").add(this.contraction);

      request.onsuccess = (event) => {
        this.contractions.push(this.contraction);
        this.contraction = {
          id:            null,
          duration:      null,
          startedAt:     null,
          finishedAt:    null,
          timeSinceLast: null,
        }
        this.isRunning = false;
        this.timer = 0;
      }

      request.onerror = (event) => {
        alert("Unable to add data");
        this.contraction = {
          id:            null,
          duration:      null,
          startedAt:     null,
          finishedAt:    null,
          timeSinceLast: null,
        }
        this.isRunning = false;
        this.timer = 0;
      }
    },

    clear() {
      let request = db.transaction(["contraction"], "readwrite")
      .objectStore("contraction")
      .clear();
      this.contractions = [];
    },

    edit(contraction, index) {
      new Promise((resolve, reject) => {
        this.remove(contraction.id, index)
        resolve(this.add(contraction));
      });
    },

    readAll() {
      try {
        this.contractions = []
        let objectStore = db.transaction("contraction").objectStore("contraction");
        if (objectStore) {
          objectStore.openCursor().onsuccess = (event) => {
            let cursor = event.target.result;

            if (cursor) {
              if (this.contractions) {
                this.contractions.push({
                  id: cursor.key,
                  duration:      cursor.value.duration,
                  startedAt:     cursor.value.startedAt,
                  finishedAt:    cursor.value.finishedAt,
                  timeSinceLast: cursor.value.timeSinceLast,
                });
              }
              cursor.continue();
            }
          };
        }
      } catch(e) {
        this.retryDisp();
      }
    },

    remove(id, index) {
      let request = db.transaction(["contraction"], "readwrite")
      .objectStore("contraction")
      .delete(id);
      this.contractions.splice(index, 1)
    },

    retryDisp() {
      if (++this.retryCount > 5) {
        console.log('Cannot open the database after 5 retries');
        this.readAll();
      }
      setTimeout(() => {
        this.readAll();
      }, 100);
    }
  }
}

</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
  .timer {
    margin: 0 auto;
    padding: 20px;
    max-width: 640px;
  }

  .button {
    border: 0;
    display: block;
    margin: 20px auto;
    padding: .5em;
    background: #42b983;
    width: 100%;
    font-size: 1.3em;
    color: white;
  }

  .counter {
    font-size: 1.3em;
    padding: .5em;
    display: block;
    width: 100%;
    border: 1px solid #dfdfdf;
    text-align: center;
  }

  .table {
    border-collapse: collapse;
    width: 100%;
  }

  .table th {
    background: #2a2a2a;
    color: white;
    border: 1px solid white;
    text-align: center;
    padding: .1em .2em;
  }

  .button.stop {
    background: #b94252;
  }

  a {
    color: #42b983;
  }
</style>
