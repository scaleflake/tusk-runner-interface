<template>
<main>
  <div>
    <h1>Tusk-Runner GUI</h1>
  </div>

  <select v-model="selectedScript">
    <option :value="null"> - </option>
    <option
      v-for="script in scripts"
      :key="script"
      :value="script"
    > {{ script }} </option>
  </select>
  <button
    style="margin-left: 5px"
    :disabled="selectedScript === null"
    @click="runScript"
  > Run </button>

  <br>

  <button
    class="toggleAll"
    v-if="scriptRuns.some(st => st.isOpened)"
    @click="scriptRuns.forEach(sr => { sr.isOpened = false; })"
  > Collapse all </button>
  <button
    class="toggleAll"
    v-else
    @click="scriptRuns.forEach(sr => { sr.isOpened = true; })"
  > Open all </button>

  <div
    class="scriptRun"
    v-for="scriptRun of scriptRuns"
    :key="scriptRun.webhooksDelivery"
    :class="{
      'scriptRun-error': scriptRun.error || scriptRun.errorStack,
      'scriptRun-active': typeof scriptRun.duration !== `number`,
    }"
  >
    <span class="scriptRun_header">
      {{ formatDatetime(scriptRun.start) }} {{ scriptRun.repoName || `-` }}
    </span>

    <button
      class="scriptRun_toggleButton"
      @click="scriptRun.isOpened = !scriptRun.isOpened"
    > {{ scriptRun.isOpened ? 'V' : '&lt;' }} </button>

    <div v-if="scriptRun.isOpened">
      <h4> Duration: {{ scriptRun.duration / 1000 }} seconds </h4>

      <div
        class="scriptRun_item scriptRun_item-error"
        v-if="scriptRun.error"
      >
        <label>
          <b> error </b>
        </label>
        <pre> {{ JSON.stringify(scriptRun.error, null, 2) }} </pre>
      </div>

      <div
        class="scriptRun_item.scriptRun_item-error"
        v-if="scriptRun.errorStack"
      >
        <label>
          <b> errorStack </b>
        </label>
        <pre> {{ scriptRun.errorStack }} </pre>
      </div>

      <div
        class="scriptRun_item"
        v-for="prop of ['scriptContent', 'stdout', 'stderr'].filter(p => p in scriptRun)"
        :key="prop"
      >
        <label>
          <b> {{ prop }} </b>
        </label>
        <pre> {{ scriptRun[prop] }} </pre>
      </div>
    </div>
  </div>
</main>
</template>

<script>
import axios from 'axios';
import ReconnectingWebSocket from 'reconnecting-websocket';

const months = [
  `jan`, `feb`,
  `mar`, `apr`,
  `may`, `jun`,
  `jul`, `aug`,
  `sep`, `oct`,
  `nov`, `dec`,
];
function formatDatetime(timestamp) {
  const date = new Date(timestamp);
  const y = date.getFullYear();
  const m = date.getMonth();
  const d = date.getDate();
  const h = date.getHours();
  const min = date.getMinutes();
  const dateString = `${d} ${months[m]} ${y}, ${h}:${min < 10 ? `0${min}` : min}`;
  const seconds = Math.round((Date.now() - timestamp) / 1000);
  if (seconds === 0) {
    return dateString;
  } else
  if (seconds < 60) {
    return `${dateString} (${seconds} sec ago)`;
  } else
  if (seconds < 3600) {
    const m = ~~(seconds / 60);
    const s = seconds - m * 60;
    const ms = `${m} min`;
    const ss = s !== 0 ? `${s} sec` : ``;
    return `${dateString} (${ms}${ms && ss ? `, ` : ``}${ss} ago)`;
  } else
  if (seconds < 3600 * 24) {
    const h = ~~(seconds / 3600);
    const m = ~~(seconds / 60) - h * 60;
    const hs = `${h} h`;
    const ms = m !== 0 ? `${m} min` : ``;
    return `${dateString} (${hs}${hs && ms ? `, ` : ``}${ms} ago)`;
  } else
  if (seconds < 3600 * 24 * 3) {
    const d = ~~(seconds / 86400);
    const h = ~~(seconds / 3600) - d * 24;
    const ds = `${d} d`;
    const hs = h !== 0 ? `${h} h` : ``;
    return `${dateString} (${ds}${ds && hs ? `, ` : ``}${hs} ago)`;
  } else
  {
    const d = ~~(seconds / 86400);
    const ds = `${d} d`;
    return `${dateString} (${ds} ago)`;
  }
}

const apiBaseUrl = `//0.narandev.ru/tusk-runner`;

export default {
  data() {
    return {
      scriptRuns: [],

      selectedScript: null,
      scripts: [],
    };
  },
  async created() {
    const ws = new ReconnectingWebSocket(`wss:${apiBaseUrl}/ws`);
    ws.onmessage = (message) => {
      const { method, params } = JSON.parse(message);

      switch (method) {
        case `scriptRun.start`: {
          this.scriptRuns.unshift(params.scriptRun);
        } break;
        case `scriptRun.end`: {
          const i = this.scriptRuns
            .findIndex(sr => sr.webhookDelivery === params.scriptRun.webhookDelivery);
          if (i !== -1) {
            this.scriptRuns[i] = params.scripts;
          }
        } break;
      }
    };
    ws.onopen = () => {
      console.log('WebSocket opened!');
    };
    ws.onclose = () => {
      console.log('WebSocket closed!');
    };


    await Promise.all([
      this.fetchScriptRuns(),
      this.fetchScripts(),
    ]);
  },
  methods: {
    formatDatetime,
    async fetchScriptRuns() {
      const { data: scriptRuns } = await axios.get(
        `https:${apiBaseUrl}/scriptRuns?limit=10&secret=${localStorage.getItem(`secret`)}`
      );
      scriptRuns.forEach(sr => { sr.isOpened = true; });
      this.scriptRuns = scriptRuns;
    },
    async fetchScripts() {
      const { data: scripts } = await axios.get(
        `https:${apiBaseUrl}/scripts`,
      );
      this.scripts = scripts;
    },

    runScript() {
      console.log(this.selectedScript);
    }
  }
}
</script>

<style>
.toggleAll {
  margin-bottom: 15px;
}

.scriptRun {
  background-color: #d2f3fa;
  margin-bottom: 60px;
}
.scriptRun.scriptRun-error {
  background-color: #f8dfdf;
}
.scriptRun.scriptRun-active {
  background-color: #dff8df;
}

.scriptRun_item {
  padding: 10px;
  margin-bottom: 10px;
  background-color: #d5e9ed
}
.scriptRun_item.scriptRun_item-error {
  background-color: #edd5d5
}

.scriptRun_header {
  font-size: 1.5em;
  font-weight: bold;
}

.scriptRun_toggleButton {
  font-size: 1.5em;
  font-family: monospace;
  margin-left: 10px;
  cursor: pointer;
}
.scriptRun_toggleButton:hover {
  font-weight: bold;
}
</style>