<template>
  <div class="iq-talk-wrapper">
    <!-- Hidden player to handle the live MSE stream -->
    <audio ref="liveAudioPlayer" autoplay></audio>

    <div v-if="!hasJoined" class="join-wrap">
      <div class="join-logo">IQ<span>Talk</span></div>
      <div class="join-tag">Real-Time Chunking Edition</div>
      <div class="join-card">
        <div class="field-label">Your name</div>
        <input 
          v-model="myName" 
          class="field-input" 
          placeholder="e.g. Priya" 
          maxlength="20" 
          @keydown.enter="joinApp"
        />
        <button class="join-btn" @click="joinApp">Join System</button>
      </div>
    </div>

    <div v-else class="app">
      <div class="hdr">
        <div class="hdr-logo">IQ<span>Talk</span></div>
        <div class="hdr-dot"></div>
        <span class="hdr-name">{{ myName }}</span>
        <span class="hdr-pill">{{ onlineUsers.length + 1 }} online</span>
      </div>

      <div class="tabs">
        <button :class="['tab', { on: activeTab === 'people' }]" @click="activeTab = 'people'">
          People<span class="badge">{{ onlineUsers.length }}</span>
        </button>
        <button :class="['tab', { on: activeTab === 'teams' }]" @click="activeTab = 'teams'">
          Teams<span class="badge">{{ teamsList.length }}</span>
        </button>
      </div>

      <div v-show="activeTab === 'people'" class="pane">
        <div class="sec-label">Direct Communication</div>
        
        <div v-if="onlineUsers.length === 0" class="empty">
          <div class="empty-icon">👥</div>
          <div class="empty-txt">No one else online yet.</div>
        </div>

        <div v-for="u in onlineUsers" :key="u.socketId" :class="['ucard', { talking: isUserTalking(u.socketId) }]">
          <div class="av av-g">{{ getInitials(u.name) }}</div>
          <div class="uinfo">
            <div class="uname">{{ u.name }}</div>
            <div :class="['usub', isUserTalking(u.socketId) ? 'talking' : 'online']">
              {{ isUserTalking(u.socketId) ? '🎙 Speaking…' : '● Online' }}
            </div>
          </div>
          <button 
            :class="['btn', isActiveTarget('user', u.socketId) ? 'btn-g btn-active' : 'btn-b']" 
            @click="setContext('user', u.socketId, u.name)"
          >
            {{ isActiveTarget('user', u.socketId) ? '✓ Active' : '→ Direct PTT' }}
          </button>
        </div>
      </div>

      <div v-show="activeTab === 'teams'" class="pane">
        <div class="sec-label">Create a team</div>
        <div class="create-row">
          <input 
            v-model="newTeamName" 
            class="create-in" 
            placeholder="Team name…" 
            maxlength="30" 
            @keydown.enter="createTeam" 
          />
          <button class="btn btn-g" @click="createTeam">Create</button>
        </div>
        
        <div class="sec-label">All teams</div>
        <div v-if="teamsList.length === 0" class="empty">
          <div class="empty-icon">📡</div>
          <div class="empty-txt">No teams yet.</div>
        </div>

        <div v-for="t in teamsList" :key="t.id" :class="['tcard', { active: isActiveTarget('team', t.id), mine: amIInTeam(t.id) }]">
          <div class="thead">
            <span style="font-size:15px">📡</span>
            <span class="tname">{{ t.name }}</span>
          </div>
          <div class="tchips">
            <span 
              v-for="m in t.members" 
              :key="m.socketId" 
              :class="['chip', { me: m.socketId === mySocketId, tk: isUserTalking(m.socketId) }]"
            >
              {{ m.name }}
            </span>
          </div>
          <div class="tacts">
            <template v-if="amIInTeam(t.id)">
              <span v-if="isActiveTarget('team', t.id)" style="color:#1D9E75; font-size:11px; font-weight:bold;">● Active</span>
              <button v-else class="btn btn-g" @click="setContext('team', t.id, t.name)">Set Active</button>
              <button class="btn btn-r" @click="leaveTeam(t.id)">Leave</button>
            </template>
            <template v-else>
              <button class="btn btn-b" @click="joinTeam(t.id)">Join</button>
            </template>
          </div>
        </div>
      </div>

      <div class="ptt-bar">
        <div class="ptt-ctx">
          <span class="ptt-ctx-label">Target:</span>
          <span :class="['ptt-badge', talkMode === 'user' ? 'direct' : talkMode === 'team' ? 'team' : '']">
            {{ targetBadgeText }}
          </span>
        </div>
        <button 
          :class="['ptt-btn', pttButtonClass]"
          @mousedown="startTalk" 
          @mouseup="stopTalk"
          @touchstart.prevent="startTalk" 
          @touchend.prevent="stopTalk"
        >
          <span>🎙️</span>
          <span v-if="isTalking">Live Transmitting...</span>
          <span v-else-if="talkMode !== 'none'">Hold to Talk</span>
          <span v-else>Select Target</span>
          
          <div v-if="isTalking" class="wave">
            <div class="wb"></div><div class="wb"></div><div class="wb"></div><div class="wb"></div><div class="wb"></div>
          </div>
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { io } from "socket.io-client";

export default {
  name: "WalkieTalkie",
  data() {
    return {
      // Auth & State
      hasJoined: false,
      myName: "",
      newTeamName: "",
      activeTab: "people",
      
      // Socket & Media Sender
      socket: null,
      mySocketId: null,
      localStream: null,
      mediaRecorder: null,
      mimeType: 'audio/webm; codecs="opus"', // Force standard codec for MSE
      
      // Data Stores
      onlineUsers: [],
      teamsList: [],
      myTeamIds: [],
      talkingIds: [], 
      
      // PTT Context
      isTalking: false,
      talkMode: "none",
      targetId: null,
      targetName: null,

      // Receiver Stream Components
      mediaSource: null,
      sourceBuffer: null,
      audioQueue: [] // Queue to hold chunks while the buffer is parsing
    };
  },
  computed: {
    targetBadgeText() {
      if (this.talkMode === 'user') return `→ ${this.targetName}`;
      if (this.talkMode === 'team') return `📡 ${this.targetName}`;
      return "— none selected —";
    },
    pttButtonClass() {
      if (this.isTalking) return 'ptt-live';
      if (this.talkMode !== 'none') return 'ptt-rdy';
      return 'ptt-off';
    }
  },
  methods: {
    getInitials(name) {
      return name ? name.substring(0, 2).toUpperCase() : "??";
    },
    isUserTalking(socketId) {
      return this.talkingIds.includes(socketId);
    },
    isActiveTarget(type, id) {
      return this.talkMode === type && this.targetId === id;
    },
    amIInTeam(teamId) {
      return this.myTeamIds.includes(teamId);
    },

    // ─── INIT & AUTH ──────────────────────────────────────────────────
    async joinApp() {
      this.myName = this.myName.trim() || "User";
      
      const micAccess = await this.setupAudioCapture();
      if (!micAccess) return;

      this.hasJoined = true;
      this.initNetwork();
    },

    async setupAudioCapture() {
      try {
        this.localStream = await navigator.mediaDevices.getUserMedia({ audio: true });
        
        // Ensure Safari compatibility or fallback
        if (!MediaRecorder.isTypeSupported(this.mimeType)) {
            if (MediaRecorder.isTypeSupported('audio/mp4')) this.mimeType = 'audio/mp4';
            else this.mimeType = ''; // Let browser decide, MSE might be tricky here
        }
        return true;
      } catch (e) {
        alert("Microphone access denied. PTT will not work.");
        return false;
      }
    },

    initNetwork() {
      this.socket = io(process.env.VUE_APP_SERVER_URL || "http://localhost:3000");

      this.socket.on('connect', () => {
        this.mySocketId = this.socket.id;
        this.socket.emit('register', { name: this.myName });
      });

      this.socket.on('users-updated', (list) => {
        this.onlineUsers = list.filter(u => u.socketId !== this.mySocketId);
      });

      this.socket.on('teams-updated', (list) => {
        this.teamsList = list;
      });

      this.socket.on('team-joined', ({ teamId, teamName }) => {
        if (!this.myTeamIds.includes(teamId)) {
          this.myTeamIds.push(teamId);
        }
        this.setContext('team', teamId, teamName);
      });

      // ─── STREAM RECEIVING LOGIC ───
      this.socket.on('incoming-stream-start', (payload) => {
        const { senderId, mimeType } = payload;
        
        // Update UI Indicator
        if (!this.talkingIds.includes(senderId)) this.talkingIds.push(senderId);

        // Setup Media Source Extension (Bucket)
        this.mediaSource = new MediaSource();
        this.$refs.liveAudioPlayer.src = URL.createObjectURL(this.mediaSource);
        this.audioQueue = [];

        this.mediaSource.addEventListener('sourceopen', () => {
          try {
            this.sourceBuffer = this.mediaSource.addSourceBuffer(mimeType);
            
            // Consume from the queue continuously as the buffer finishes parsing chunks
            this.sourceBuffer.addEventListener('updateend', () => {
              if (this.audioQueue.length > 0 && !this.sourceBuffer.updating) {
                this.sourceBuffer.appendBuffer(this.audioQueue.shift());
              }
            });
          } catch (e) {
            console.error("MSE Buffer Setup Error (Codec mismatch?):", e);
          }
        });
      });

      this.socket.on('incoming-stream-chunk', (payload) => {
        const { audioChunk } = payload; // Arrives as ArrayBuffer
        
        if (this.sourceBuffer && this.mediaSource.readyState === 'open') {
          if (!this.sourceBuffer.updating) {
            // Buffer is free, parse chunk instantly
            this.sourceBuffer.appendBuffer(audioChunk);
          } else {
            // Buffer is busy parsing previous chunk, queue this one
            this.audioQueue.push(audioChunk);
          }
        } else {
          // Source not fully open yet, store the chunks safely
          this.audioQueue.push(audioChunk);
        }
      });

      this.socket.on('incoming-stream-stop', (payload) => {
        const { senderId } = payload;
        
        // Update UI
        this.talkingIds = this.talkingIds.filter(id => id !== senderId);

        // Close the MSE stream
        if (this.mediaSource && this.mediaSource.readyState === 'open') {
          try {
            this.mediaSource.endOfStream();
          } catch(e) { 
            console.error("Error to play audio: ",e);
            
          }
        }
      });
    },

    // ─── TEAM ACTIONS ──────────────────────────────────────────────────
    createTeam() {
      if (!this.newTeamName.trim()) return;
      this.socket.emit('create-team', { teamName: this.newTeamName.trim() });
      this.newTeamName = "";
    },
    joinTeam(teamId) {
      this.socket.emit('join-team', { teamId });
    },
    leaveTeam(teamId) {
      this.myTeamIds = this.myTeamIds.filter(id => id !== teamId);
      this.socket.emit('leave-team', { teamId });
      if (this.targetId === teamId) this.setContext('none', null, null);
    },

    // ─── PTT LOGIC (SENDER) ────────────────────────────────────────────
    setContext(mode, id, name) {
      this.talkMode = mode;
      this.targetId = id;
      this.targetName = name;
    },

    startTalk() {
      if (this.isTalking || this.talkMode === 'none' || !this.localStream) return;

      this.isTalking = true;
      this.mediaRecorder = new MediaRecorder(this.localStream, { mimeType: this.mimeType });
      
      this.socket.emit('stream-start', { 
        targetType: this.talkMode, 
        targetId: this.targetId,
        mimeType: this.mimeType 
      });
      
      this.mediaRecorder.ondataavailable = (event) => {
        if (event.data.size > 0) {
          // Emit the chunk instantly as binary Blob
          this.socket.emit('stream-chunk', {
            targetType: this.talkMode,
            targetId: this.targetId,
            audioChunk: event.data
          });
        }
      };
      
      // The crucial step for streaming: timeslice! Fire event every 250ms
      this.mediaRecorder.start(250); 
    },

    stopTalk() {
      if (!this.isTalking) return;
      
      this.isTalking = false;
      if (this.mediaRecorder && this.mediaRecorder.state !== "inactive") {
        this.mediaRecorder.stop();
      }
      
      this.socket.emit('stream-stop', { targetType: this.talkMode, targetId: this.targetId });
    }
  },
  beforeDestroy() {
    if (this.socket) this.socket.disconnect();
    if (this.localStream) this.localStream.getTracks().forEach(track => track.stop());
  }
};
</script>

<style scoped>
* { box-sizing: border-box; margin: 0; padding: 0; }
.iq-talk-wrapper { font-family: system-ui, -apple-system, sans-serif; background: #0f1117; color: #e8e8e8; height: 100vh; overflow: hidden; }

.join-wrap { display: flex; flex-direction: column; align-items: center; justify-content: center; min-height: 100vh; padding: 24px; }
.join-logo { font-size: 32px; font-weight: 800; color: #fff; letter-spacing: -1px; margin-bottom: 4px; }
.join-logo span { color: #1D9E75; }
.join-tag { font-size: 12px; color: #555; margin-bottom: 32px; }
.join-card { background: #1a1d27; border: 0.5px solid #2a2d3a; border-radius: 16px; padding: 24px; width: 100%; max-width: 360px; }
.field-label { font-size: 10px; color: #555; text-transform: uppercase; letter-spacing: 0.07em; margin-bottom: 8px; }
.field-input { width: 100%; background: #0f1117; border: 0.5px solid #2a2d3a; border-radius: 8px; color: #e8e8e8; font-size: 15px; padding: 12px 14px; outline: none; font-family: inherit; margin-bottom: 14px; }
.field-input:focus { border-color: #1D9E75; }
.join-btn { width: 100%; padding: 13px; background: #1D9E75; color: #fff; border: none; border-radius: 8px; font-size: 15px; font-weight: 700; cursor: pointer; transition: background 0.15s, transform 0.1s; }
.join-btn:active { background: #19896a; transform: scale(0.98); }

.app { display: flex; flex-direction: column; height: 100vh; max-width: 480px; margin: 0 auto; }
.hdr { background: #1a1d27; border-bottom: 0.5px solid #2a2d3a; padding: 10px 16px; display: flex; align-items: center; gap: 10px; flex-shrink: 0; }
.hdr-logo { font-size: 17px; font-weight: 800; color: #fff; letter-spacing: -0.5px; flex: 1; }
.hdr-logo span { color: #1D9E75; }
.hdr-dot { width: 7px; height: 7px; border-radius: 50%; background: #1D9E75; flex-shrink: 0; animation: pglow 2s infinite; }
@keyframes pglow { 0%,100%{box-shadow:0 0 0 0 rgba(29,158,117,.5)}50%{box-shadow:0 0 0 4px rgba(29,158,117,0)} }
.hdr-name { font-size: 13px; color: #aaa; }
.hdr-pill { font-size: 11px; color: #555; background: #0f1117; border: 0.5px solid #2a2d3a; border-radius: 10px; padding: 2px 8px; }

.tabs { display: flex; background: #1a1d27; border-bottom: 0.5px solid #2a2d3a; flex-shrink: 0; }
.tab { flex: 1; padding: 10px 6px; background: none; border: none; color: #555; font-size: 11px; font-weight: 700; cursor: pointer; border-bottom: 2px solid transparent; text-transform: uppercase; transition: color 0.15s, border-color 0.15s; }
.tab.on { color: #1D9E75; border-bottom-color: #1D9E75; }
.badge { display: inline-block; background: #2a2d3a; color: #888; font-size: 9px; border-radius: 8px; padding: 1px 5px; margin-left: 3px; vertical-align: middle; }
.tab.on .badge { background: #0a2a20; color: #1D9E75; }

.pane { flex: 1; overflow-y: auto; padding: 12px 16px; }
.sec-label { font-size: 10px; color: #555; text-transform: uppercase; margin-bottom: 8px; margin-top: 4px; }

.ucard { background: #1a1d27; border: 0.5px solid #2a2d3a; border-radius: 12px; padding: 10px 14px; margin-bottom: 8px; display: flex; align-items: center; gap: 12px; }
.ucard.talking { border-color: #1D9E75; background: #081a14; }
.av { width: 36px; height: 36px; border-radius: 50%; display: flex; align-items: center; justify-content: center; font-weight: 700; font-size: 13px; flex-shrink: 0; }
.av-g { background: #0a2a20; color: #1D9E75; }
.uinfo { flex: 1; min-width: 0; }
.uname { font-size: 13px; font-weight: 500; color: #e8e8e8; }
.usub { font-size: 11px; color: #555; margin-top: 1px; }
.usub.online { color: #1D9E75; }
.usub.talking { color: #f39c12; }

.btn { padding: 6px 11px; border-radius: 6px; border: none; font-size: 11px; font-weight: 600; cursor: pointer; transition: background 0.12s; }
.btn-g { background: #0a2a20; color: #1D9E75; border: 0.5px solid #1D9E7540; }
.btn-g:active { background: #1D9E75; color: #fff; }
.btn-b { background: #0a1a2e; color: #378ADD; border: 0.5px solid #378ADD40; }
.btn-b:active { background: #378ADD; color: #fff; }
.btn-r { background: #2a1015; color: #e74c3c; border: 0.5px solid #e74c3c40; }
.btn-r:active { background: #e74c3c; color: #fff; }
.btn-active { background: #1D9E75 !important; color: #fff !important; }

.tcard { background: #1a1d27; border: 0.5px solid #2a2d3a; border-radius: 12px; padding: 12px 14px; margin-bottom: 8px; }
.tcard.mine { border-color: #1D9E7530; }
.tcard.active { border-color: #1D9E75; background: #081a14; }
.thead { display: flex; align-items: center; gap: 8px; margin-bottom: 8px; }
.tname { font-size: 13px; font-weight: 600; color: #e8e8e8; flex: 1; }
.tchips { display: flex; flex-wrap: wrap; gap: 4px; margin-bottom: 10px; }
.chip { font-size: 11px; color: #888; background: #0f1117; border: 0.5px solid #2a2d3a; border-radius: 10px; padding: 2px 8px; }
.chip.me { color: #378ADD; border-color: #378ADD40; }
.chip.tk { color: #f39c12; border-color: #f39c1240; }
.tacts { display: flex; gap: 6px; flex-wrap: wrap; align-items: center; }

.create-row { display: flex; gap: 8px; margin-bottom: 16px; }
.create-in { flex: 1; background: #0f1117; border: 0.5px solid #2a2d3a; border-radius: 8px; color: #e8e8e8; font-size: 13px; padding: 10px 12px; outline: none; }

.ptt-bar { flex-shrink: 0; background: #1a1d27; border-top: 0.5px solid #2a2d3a; padding: 10px 16px 14px; }
.ptt-ctx { display: flex; align-items: center; justify-content: center; gap: 6px; margin-bottom: 8px; }
.ptt-ctx-label { font-size: 11px; color: #555; }
.ptt-badge { font-size: 11px; background: #0f1117; border: 0.5px solid #2a2d3a; border-radius: 8px; padding: 3px 10px; color: #666; }
.ptt-badge.direct { color: #378ADD; border-color: #378ADD40; background: #0a1322; }
.ptt-badge.team { color: #1D9E75; border-color: #1D9E7540; background: #081a14; }

.ptt-btn { width: 100%; padding: 20px 16px; border-radius: 14px; border: none; font-size: 15px; font-weight: 700; cursor: pointer; display: flex; align-items: center; justify-content: center; gap: 10px; user-select: none; transition: transform 0.1s; }
.ptt-off { background: #1a1d27; border: 1.5px solid #2a2d3a; color: #383b4a; cursor: default; }
.ptt-rdy { background: #0a1f18; border: 1.5px solid #1D9E75; color: #1D9E75; }
.ptt-live { background: #1D9E75; color: #fff; transform: scale(0.97); }

.wave { display: flex; gap: 3px; align-items: center; height: 18px; margin-left: 10px; }
.wb { width: 3px; border-radius: 2px; background: #fff; animation: wave .7s ease-in-out infinite; }
.wb:nth-child(1){height:4px;animation-delay:0s}
.wb:nth-child(2){height:10px;animation-delay:.1s}
.wb:nth-child(3){height:16px;animation-delay:.2s}
.wb:nth-child(4){height:10px;animation-delay:.3s}
.wb:nth-child(5){height:4px;animation-delay:.4s}
@keyframes wave{0%,100%{transform:scaleY(.4);opacity:.5}50%{transform:scaleY(1);opacity:1}}

.empty { text-align: center; padding: 28px 12px; }
.empty-icon { font-size: 28px; margin-bottom: 8px; }
.empty-txt { font-size: 12px; color: #444; line-height: 1.8; }
</style>