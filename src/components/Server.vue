<template>
  <tr :class="trClasses" v-if="server.status >= 0">
    <td>
      <span class="fullAddress">{{ fullAddress }}</span>
      <button class="button margin-left hide--on-mobile" data-tooltip="Copy">
        <img
          alt="Copy"
          class="icon"
          src="../assets/copy.png"
          data-copy-selector=".fullAddress"
          v-on:click="copy"
        />
      </button>
    </td>
    <td :data-tooltip="infos">
      <span v-if="server.data.active >= 0 && server.data.idle >= 0">
        <span v-if="server.data.active >= 0">
          {{ server.data.active }}
          <img alt="Active users" class="icon" src="../assets/active.png" />
        </span>
        <span v-if="server.data.idle >= 0">
          / {{ server.data.idle }}
          <img alt="Idle users" class="icon" src="../assets/idle.png" />
        </span>
      </span>
      <span v-else>
        <span v-if="server.data.online >= 0">
          {{ server.data.online }}
          <img alt="Online users" class="icon" src="../assets/online.png" />
        </span>
      </span>
    </td>
    <td class="hide--on-mobile" :data-tooltip="country">
      <flag :iso="server.flag" :squared="false" />
    </td>
    <CellIcon class="hide--on-mobile" :platform="server.platform" />
    <td>
      <span v-if="server.ping >= 0">{{ server.ping }} ms</span>
      <span v-else>n/a</span>
    </td>
  </tr>
</template>

<script>
import CellIcon from "@/components/CellIcon.vue";

const fetchWithTimeout = function(url, options, timeout = 20000) {
  return Promise.race([
    fetch(url, options),
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error("timeout")), timeout)
    )
  ]);
};

const gqlPing = (server, delay = 0, timeout = 20000) => {
  return new Promise((resolve, reject) => {
    setTimeout(() => reject(new Error("timeout")), timeout);
    const ws = new WebSocket(`ws://${server}`, "graphql-ws");
    let timeoutId = undefined;
    let lastTime = undefined;
    const doPing = () => {
      ws.send(
        `{"id":"1","type":"start","payload":{"variables":{},"extensions":{},"operationName":null,"query":"subscription{serverInfo{online idle version}}"}}`
      );
      lastTime = Date.now();
    };
    ws.onmessage = e => {
      const data = JSON.parse(e.data);
      if (data.type === "data" && data.id === "1") {
        let ping = Date.now() - lastTime;
        resolve({ data: data.payload.data.serverInfo, ping });
        ws.send(`{"id":"1","type":"stop"}`);
        ws.close();
      }
    };
    ws.onclose = () => {
      timeoutId && clearTimeout(timeoutId);
    };
    ws.onerror = e => {
      reject(e);
    };
    ws.onopen = () => {
      ws.send(`{"type":"connection_init","payload":{}}`);
      timeoutId = setTimeout(doPing, delay);
    };
  });
};

export default {
  components: {
    CellIcon
  },
  data: () => {
    return {
      timerServer: undefined
    };
  },
  props: {
    server: Object
  },
  computed: {
    fullAddress() {
      return `${this.server.ip}:${this.server.port}`;
    },
    infos() {
      let infos = `Server type: ${this.server.type}`;
      if (this.server.data.version) {
        infos = `${infos}
(v${this.server.data.version})`;
      }
      return infos;
    },
    country() {
      return this.server.flag.toUpperCase();
    },
    trClasses() {
      return this.server.highlight ? "highlight" : "";
    }
  },
  methods: {
    copy(event) {
      let el = this.$el.querySelector(
        event.target.getAttribute("data-copy-selector")
      );
      let range = document.createRange();
      range.selectNode(el);
      window.getSelection().addRange(range);
      document.execCommand("copy");
      window.getSelection().removeAllRanges();
    },
    async gqlRefresh() {
      let ctx = this;
      try {
        let { data, ping } = await gqlPing(
          `${ctx.server.ip}:${ctx.server.port}`
        );

        ctx.server.ping = ping;
        ctx.server.data = {
          active: data.online - data.idle,
          ...data
        };

        if (ctx.server.status === undefined) {
          ctx.server.status = 0;
          clearInterval(ctx.timerServer);
          ctx.timerServer = setInterval(ctx.gqlRefresh, 5000);
        }
      } catch (e) {
        ctx.server.status = undefined;
        clearInterval(ctx.timerServer);
        ctx.timerServer = setInterval(ctx.gqlRefresh, 300000);
        ctx.server.ping = 0;
        ctx.server.data = {};
      }
    },
    async restRefresh() {
      let ctx = this;
      let url = `${this.server.ip}:${this.server.port}`;
      if (ctx.server.status === -1 || ctx.server.status === 1) {
        url = `/proxy.php?address=${url}`;
      } else {
        url = `//${url}`;
      }
      if (this.server.type === "node" || this.server.type === "rust") {
        url = `${url}/info`;
      }
      let lastTime = Date.now();
      try {
        const response = await fetchWithTimeout(url);
        if (!response.ok) {
          throw new Error(response);
        }

        if (ctx.server.status === undefined || ctx.server.status === 0) {
          ctx.server.ping = Math.ceil((Date.now() - lastTime) * 0.3);
        } else {
          ctx.server.ping = undefined;
        }

        const data = await response.json();

        if (ctx.server.status === undefined) {
          ctx.server.status = 0;
          clearInterval(this.timerServer);
          this.timerServer = setInterval(this.restRefresh, 5000);
        }

        if (ctx.server.status === -1) {
          ctx.server.status = 1;
          clearInterval(this.timerServer);
          this.timerServer = setInterval(this.restRefresh, 30000);
        }

        if (this.server.type === "node") {
          ctx.server.data = {
            ...data
          };
        } else if (this.server.type === "rust") {
          ctx.server.data = {
            active: data.online - data.idle,
            ...data
          };
        } else if (this.server.type === "dotnet") {
          ctx.server.data = { online: data.clientCount };
        }
      } catch (e) {
        if (ctx.server.status === undefined) {
          // Maybe CORS issue, test with proxy
          ctx.server.status = -1;
          this.restRefresh();
        } else {
          // timeout
          ctx.server.status = undefined;
          clearInterval(this.timerServer);
          this.timerServer = setInterval(this.restRefresh, 300000);
        }
        ctx.server.ping = 0;
        ctx.server.data = {};
      }
    }
  },
  created() {
    if (this.server.type === "rust") {
      this.gqlRefresh();
    } else {
      this.restRefresh();
    }
  },
  beforeDestroy() {
    clearInterval(this.timerServer);
  }
};
</script>

<style lang="scss">
tr.highlight {
  td {
    font-weight: bold;
  }
}
.margin-left {
  margin-left: 4px;
}
.button {
  padding: 4px;
  cursor: pointer;
  outline: none;
  border: none;
  background-color: #ffffff00;
  border-radius: 4px;

  &:hover {
    background-color: #52525220;
  }

  &:active {
    background-color: #52525240;
    transform: translateY(1px);
  }
}
</style>
