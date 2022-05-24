<template>
  <div class="hello">
    <canvas id="canvas"></canvas>
    <audio
      id="audio"
      :src="songFile"
      @canplay="ready"
      @ended="end"
      @timeupdate="updateTime"
      ref="audio"
      crossOrigin="anonymous"
    ></audio>
    <p>{{ formatTime(currentTime) }}</p>
    <button @click="play">播放</button>&emsp;
    <button @click="end">暂停</button>&emsp;
    <select @change="changeSong">
      <option v-for="item in songArray" :key="item.name" :value="item.file">
        {{ item.name }}
      </option>
    </select>
    <select @change="changeTheme">
      <option v-for="item in themeArray" :key="item" :value="item">
        {{ item }}
      </option>
    </select>
  </div>
</template>

<script>
import { audioTheme, formatTime } from "../utils";
export default {
  data() {
    return {
      themeArray: [
        "apple",
        "aurora",
        "borealis",
        "candy",
        "cool",
        "dusk",
        "outrun",
        "summer",
        "tiedye",
        "rainbow",
      ],
      songArray: [
        {
          name: "夏日漱石",
          file: require("../assets/夏日漱石.mp3"),
        },
        {
          name: "群青",
          file: require("../assets/群青.mp3"),
        },
        {
          name: "Bleeding Love",
          file: require("../assets/Bleeding Love.mp3"),
        },
        {
          name: "sold out",
          file: require("../assets/sold out.mp3"),
        },
      ],
      songFile: null,
      formatTime,
      songReady: false,
      currentTime: 0,

      contextAudio: null,
      analyserAudio: null,
      sourceAudio: null,
      theme: "apple",

      requestId: null,
    };
  },
  computed: {
    audio() {
      return this.$refs.audio;
    },
  },
  mounted() {
    this.songFile = this.songArray[0].file;
  },
  methods: {
    ready() {
      this.songReady = true;
      console.log("ready");
    },
    play() {
      if (this.songReady) {
        this.audio.play();
        this.onLoadAudio();
      }
    },
    end() {
      this.audio.pause();
      window.cancelAnimationFrame(this.requestId);
    },
    updateTime(e) {
      this.currentTime = e.target.currentTime;
    },
    changeSong(e) {
      this.songFile = e.target.value;
      this.audio.load();
    },
    changeTheme(e) {
      this.theme = e.target.value;
    },
    onLoadAudio() {
      // debugger
      if (!this.contextAudio) {
        // 创建AudioContext，关联音频输入，进行解码、控制音频播放和暂停
        this.contextAudio = new (window.AudioContext ||
          window.webkitAudioContext)();
      }
      if (!this.analyserAudio) {
        // 创建analyser，获取音频的频率数据（FrequencyData）和时域数据（TimeDomainData）
        this.analyserAudio = this.contextAudio.createAnalyser();
        // fftSize：快速傅里叶变换，信号样本的窗口大小，区间为32-32768，默认2048
        this.analyserAudio.fftSize = 512;
      }

      if (!this.sourceAudio) {
        // 创建音频源
        this.sourceAudio = this.contextAudio.createMediaElementSource(
          this.audio
        );
        // 音频源关联到分析器
        this.sourceAudio.connect(this.analyserAudio);
        // 分析器关联到输出设备（耳机、扬声器等）
        this.analyserAudio.connect(this.contextAudio.destination);
      }

      // 获取频率数组，即要渲染的数据，frequencyBinCount 是 fftsize 的一半
      const bufferLength = this.analyserAudio.frequencyBinCount;

      const dataArray = new Uint8Array(bufferLength);
 
      const canvas = document.getElementById("canvas");
      canvas.width = 480;
      canvas.height = 250;
      const ctx = canvas.getContext("2d");
      const WIDTH = canvas.width;
      const HEIGHT = canvas.height;

      const barWidth = WIDTH / bufferLength;
      let barHeight;

      const renderFrame = () => {
        this.requestId = requestAnimationFrame(renderFrame);

        // 将当前频率数据复制到传入的Uint8Array，更新频率数据
        this.analyserAudio.getByteFrequencyData(dataArray);
        ctx.clearRect(0, 0, WIDTH, HEIGHT);
        // bufferLength表示柱形图中矩形的个数，当前是128个
        for (let i = 0, x = 0; i < bufferLength; i++) {
          barHeight = dataArray[i];
          // 创建线性渐变对象
          const gradient = ctx.createLinearGradient(0, 0, 0, 250);
          // 渲染颜色
          this.colorPick(gradient);
          ctx.fillStyle = gradient;
          // 左上X，左上Y，width，height
          ctx.fillRect(x, 250 - barHeight, barWidth, barHeight);
          x += barWidth + 2;
        }
      };

      renderFrame();
    },
    colorPick(gradient) {
      audioTheme[this.theme].forEach((item) => {
        const { pos, color } = item;
        gradient.addColorStop(pos, color);
      });
    },
  },
};
</script>