<template>
  <v-app style="background: #12141f;">
    <HelloWorld/>
  </v-app>
</template>

<script>
import HelloWorld from './components/HelloWorld';

export default {
  name: 'App',
  components: { HelloWorld }
};
</script>

<style>
html { overflow-y: auto; }

.wave-bar { display: inline-flex; gap: 3px; align-items: center; height: 18px; }
.wb { width: 3px; border-radius: 2px; background: white; animation: wave 0.7s ease-in-out infinite; }
.wb:nth-child(1){height:4px;animation-delay:0s}
.wb:nth-child(2){height:10px;animation-delay:0.1s}
.wb:nth-child(3){height:16px;animation-delay:0.2s}
.wb:nth-child(4){height:10px;animation-delay:0.3s}
.wb:nth-child(5){height:4px;animation-delay:0.4s}
@keyframes wave{0%,100%{transform:scaleY(0.4);opacity:0.5}50%{transform:scaleY(1);opacity:1}}

.pulse-glow { animation: pglow 2s infinite; }
@keyframes pglow { 0%,100%{box-shadow:0 0 0 0 rgba(29,158,117,0.5)}50%{box-shadow:0 0 0 6px rgba(29,158,117,0)} }

.logbox { background: #121212; border: 1px solid #333; border-radius: 8px; padding: 12px; height: 150px; overflow-y: auto; font-family: monospace; font-size: 12px; }
.log-g { color: #1D9E75; } .log-y { color: #f39c12; } .log-r { color: #e74c3c; }

::-webkit-scrollbar { width: 6px; }
::-webkit-scrollbar-track { background: transparent; }
::-webkit-scrollbar-thumb { background: #444; border-radius: 10px; }

.ptt-btn-wrapper { touch-action: none; user-select: none; -webkit-user-select: none; }
</style>
