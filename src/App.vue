<template lang="pug">
  main
    h1 Tusk-Runner GUI
    button.toggleAll(
      v-if="scriptRuns.some(st => st.isOpened)"
      @click="scriptRuns.forEach(sr => { sr.isOpened = false; })"
    ) Collapse all
    button.toggleAll(
      v-else
      @click="scriptRuns.forEach(sr => { sr.isOpened = true; })"
    ) Open all

    .scriptRun(
      v-for="scriptRun of scriptRuns"
      :class="{ \
        'scriptRun-error': scriptRun.error || scriptRun.errorStack, \
        'scriptRun-active': typeof scriptRun.duration !== `number`, \
      }"
    )
      span.scriptRun_header {{ formatDatetime(scriptRun.start) }} {{ scriptRun.repoName || `-` }}

      button.scriptRun_toggleButton(
        @click="scriptRun.isOpened = !scriptRun.isOpened"
      ) {{ scriptRun.isOpened ? 'V' : '<' }}

      div(v-if="scriptRun.isOpened")
        h4 Duration: {{ scriptRun.duration / 1000 }} seconds
        .scriptRun_item.scriptRun_item-error(v-if="scriptRun.error")
          label
            b error
          pre {{ JSON.stringify(scriptRun.error, null, 2) }}
        .scriptRun_item.scriptRun_item-error(v-if="scriptRun.errorStack")
          label
            b errorStack
          pre {{ scriptRun.errorStack }}
        .scriptRun_item(
          v-for="prop of ['scriptContent', 'stdout', 'stderr']"
          v-if="prop in scriptRun"
        )
            label
              b {{ prop }}
            pre {{ scriptRun[prop] }}
</template>

<script>
import axios from 'axios';

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

export default {
  data() {
    return {
      scriptRuns: [],
    };
  },
  async created() {
    const { data: scriptRuns } = await axios.get(
      `https://0.narandev.ru/tusk-runner/lastScriptRuns?limit=10&secret=${
        localStorage.getItem(`secret`)
      }`,
    );
    scriptRuns.forEach(sr => { sr.isOpened = true; });
    this.scriptRuns = scriptRuns;
  },
  methods: {
    formatDatetime,
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