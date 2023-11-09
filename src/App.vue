<template>
  <v-card style="max-width: 400px; margin:auto; margin-top:25px;">
    <div id="preview"
      :style="screenCaptrueStream ? 'height:' + ((400 / screenCaptrueStream.getVideoTracks()[0].getSettings().width) * screenCaptrueStream.getVideoTracks()[0].getSettings().height) + 'px;' : (webcamStream ? 'height:200px' : '')">
      <video id="screen" style="width:400px;" :srcObject="screenCaptrueStream" autoplay muted playsinline></video>
      <video id="camera"
        style="background-color: #FBE1B7;height:0px;width: 100px;position: absolute;top: 10px;right: 10px;"
        :class="webcamStream ? 'webcam-big' : ''" :srcObject="webcamStream" autoplay muted playsinline></video>
    </div>
      <v-card-title >Simple Screen Recorder</v-card-title>
      <v-card-subtitle >Your screen capture companion</v-card-subtitle>
    <ion-card-content id="settings">
      <v-list>
        <v-list-item>
          <template v-slot:prepend>
            <v-icon :class="screenCaptrueStream ? 'icon-blink' : ''" icon="mdi-monitor"></v-icon>
          </template>
          <template v-slot:default>
            Screen <span v-if="!screenCaptrueStream">(Required)</span>
          </template>
          <template v-slot:append>
            <v-btn slot="end" size="default" @click="startScreenCapture" :loading="gettingScreenCapture"> {{ screenText }} </v-btn>
          </template>
        </v-list-item>
        <v-divider inset></v-divider>
        <v-list-item>
          <template v-slot:prepend>
            <v-icon :class="shouldRecordWebcam ? 'icon-blink' : ''" icon="mdi-webcam"></v-icon>
          </template>
          <template v-slot:default>
            Webcam
          </template>
          <template v-slot:append>
            <v-switch slot="end" v-model="shouldRecordWebcam" color="primary" value="primary" hide-details :loading="gettingWebcam"
              @click="startWebcamStream"></v-switch>
          </template>
        </v-list-item>
        <v-divider inset></v-divider>
        <v-list-item>
          <template v-slot:prepend>
            <v-icon :class="shouldRecordMic ? 'icon-blink' : ''" icon="mdi-microphone"></v-icon>
          </template>
          Microphone
          <template v-slot:append>
            <v-switch slot="end" v-model="shouldRecordMic" color="primary" value="primary" hide-details :loading="gettingMic"
              @click="startAudioStream"></v-switch>
          </template>
        </v-list-item>
        <v-list-item>
          <v-select v-if="microphoneDeviceOptions.length" v-model="microphoneDeviceId" :items="microphoneDeviceOptions" item-title="text" item-value="value"
            label="Select" :disabled="!shouldRecordMic" single-line></v-select>
        </v-list-item>
      </v-list>
    </ion-card-content>
    <div style="height: 50px;overflow: hidden;position:relative;">
      <v-btn id="previewbtn" @click="record" class="button-item" style="z-index: 2;"
        :disabled="!screenCaptrueStream">Record</v-btn>
      <v-btn id="svbtn" @click="save" class="button-item " color="secondary">Save</v-btn>
      <v-btn id="cancelbtn" @click="cancel" class="button-item " style="z-index: 1;" color="danger">Cancel</v-btn>
    </div>
  </v-card>
</template>
<script setup>
import { ref, onMounted } from 'vue';
import { VideoStreamMerger } from 'video-stream-merger';
console.log({ VideoStreamMerger })

const audioStream = ref(null);
const microphoneDeviceId = ref(null);
const microphoneDeviceOptions = ref([]);
const webcamStream = ref(null);
const screenCaptrueStream = ref(null);
const shouldRecordWebcam = ref(false);
const shouldRecordMic = ref(false);
const gettingScreenCapture = ref(false);
const gettingWebcam = ref(false);
const gettingMic = ref(false);

const screenText = ref('Select');

let recorder;
let chunks = [];
let merger;
let mediaStream;

async function startWebcamStream() {
  gettingWebcam.value = true;
  try{
    if (webcamStream.value) {
      webcamStream.value.getTracks().forEach(track => track.stop());
      webcamStream.value = null;
    } else {
      webcamStream.value = await navigator.mediaDevices.getUserMedia({ video: true });
    }
  }catch(e){
    console.error(e);
  }finally{
    gettingWebcam.value = false;
  }
}

async function startAudioStream() {
  gettingMic.value = true;
  try{
    if (audioStream.value) {
      audioStream.value.getTracks().forEach(track => track.stop());
      audioStream.value = null;
      return;
    }
    audioStream.value = await navigator.mediaDevices.getUserMedia({
      audio: { deviceId: microphoneDeviceId.value ? { exact: microphoneDeviceId.value } : undefined }
    });

    if(microphoneDeviceOptions.value.length === 0){
      getMicrophones();
    }
  }catch(e){
    console.error(e);
  }finally{
    gettingMic.value = false;
  }
}

async function startScreenCapture() {
  gettingScreenCapture.value = true;
  try{
    if (screenCaptrueStream.value) {
      screenCaptrueStream.value.getTracks().forEach(track => track.stop());
      screenCaptrueStream.value = null;
      screenText.value = 'Select';
      return;
    }
    screenCaptrueStream.value = await navigator.mediaDevices.getDisplayMedia({ video: true, audio: true });

    const video = screenCaptrueStream.value.getVideoTracks()[0];
    console.log({ video })
    const constrains = video.getConstraints();
    const settings = video.getSettings();
    console.log({ constrains })
    console.log({ settings })
    screenText.value = 'Stop';
  }catch(e){
    console.error(e);
  }finally{
    gettingScreenCapture.value = false;
  }


}

onMounted(async () => {
  previewbtn = document.getElementById('previewbtn');
  settingsDiv = document.getElementById('settings');
  svbtn = document.getElementById('svbtn');
  cancelbtn = document.getElementById('cancelbtn');
  svbtn.classList.add('left')
  svbtn.classList.add('down')
  cancelbtn.classList.add('hide')
  cancelbtn.classList.add('center')
});

async function getMicrophones(){
  
  const devices = await navigator.mediaDevices.enumerateDevices();
  const audioInputDevices = devices.filter(device => device.kind === 'audioinput');

  audioInputDevices.forEach(device => {
    console.log({ device });
    if (microphoneDeviceId.value === null) {
      microphoneDeviceId.value = device.deviceId;
    }
    microphoneDeviceOptions.value.push({ value: device.deviceId, text: device.label || `Microphone (${audioInputDevices.indexOf(device) + 1})` });
  });
}

const previewButtonText = ref('Record');
const buttonState = ref('preview');
const recordWebcam = ref(false);
var previewbtn;
var svbtn;
var cancelbtn;
var settingsDiv;

function record() {
  console.log('preview')
  settingsDiv.classList.add('hide-settings');
  previewbtn.classList.add('right');
  previewbtn.classList.add('hide');
  svbtn.classList.remove('down');
  cancelbtn.classList.remove('hide');
  cancelbtn.classList.add('right');
  cancelbtn.classList.remove('center');
  buttonState.value = 'record';

  const options = {
    recordAudio: shouldRecordMic.value,
    recordVideo: shouldRecordWebcam.value,
    recordScreen: true,
    audioDeviceId: shouldRecordMic.value ? microphoneDeviceId.value : null
  };
  console.log({ options })
  try {
    mediaStream = screenCaptrueStream.value;

    if (options.recordVideo) {
      var merger = new VideoStreamMerger();
      const video = screenCaptrueStream.value.getVideoTracks()[0];
      const settings = video.getSettings();
      const width = settings.width;
      const height = settings.height;
      console.log({ width, height })
      merger.setOutputSize(width, height);
      // Add the screen capture. Position it to fill the whole stream (the default)
      merger.addStream(screenCaptrueStream.value, {
        x: 0, // position of the topleft corner
        y: 0,
        width: width,
        height: height,
        mute: true // we don't want sound from the screen (if there is any)
      })

      // Add the webcam stream. Position in the top right corner with a width of 150px or 20% of the total stream width.
      const webCampVideoTrack = webcamStream.value.getVideoTracks()[0];
      const webCamSettings = webCampVideoTrack.getSettings();
      const webcamWidth = width * 0.2 > 150 ? 150 : width * 0.2;
      const webcamHeight = (webCamSettings.height / webCamSettings.width) * webcamWidth;



      merger.addStream(webcamStream.value, {
        x: width - webcamWidth,
        y: 0,
        width: webcamWidth,
        height: webcamHeight,
        mute: true
      })

      // Start the merging. Calling this makes the result available to us
      merger.start()

      // We now have a merged MediaStream!
      mediaStream = merger.result
    }
    if (options.recordAudio) {
      audioStream.value.getAudioTracks().forEach(track => mediaStream.addTrack(track));
    }

    if (!mediaStream || mediaStream.getTracks().length === 0) {
      throw new Error('No media tracks available');
    }
    chunks = [];
    recorder = new MediaRecorder(mediaStream);
    recorder.ondataavailable = event => chunks.push(event.data);
    recorder.onstop = exportRecording;
    recorder.start();
  } catch (e) {
    console.error('Error capturing media', e);
    alert(e.message);
  }
}

function save() {
  retrunToSettings();
  stopRecording();
  startScreenCapture();

  if (webcamStream.value) {
    startWebcamStream();
    shouldRecordWebcam.value = false;
  }
  if (audioStream.value) {
    startAudioStream();
    shouldRecordMic.value = false;
  }
}

function cancel() {
  retrunToSettings();
  recorder.onstop = null;
  recorder = null;
  chunks = [];
  merger = null;
  mediaStream = null;
}


function retrunToSettings() {
  console.log('retrunToSettings')
  if (buttonState.value == 'record') {
    previewbtn.classList.remove('right');
    previewbtn.classList.remove('hide');
    svbtn.classList.add('down');
    cancelbtn.classList.add('center');
    cancelbtn.classList.remove('right');
    cancelbtn.classList.add('hide');
    settingsDiv.classList.remove('hide-settings');
    buttonState.value = 'preview';
  }
}

function stopRecording() {
  recorder.stop();
}
function exportRecording() {
  const blob = new Blob(chunks, { type: 'video/webm' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'capture.webm';
  a.click();
  URL.revokeObjectURL(url);
}
</script>


<style scoped>
#preview {
  transition: height 1.5s ease;
  height: 0px;
  overflow: hidden;
  background-image: url('./assets/preview1.jpeg');
  background-repeat: no-repeat;
  background-attachment: unset;
  background-position: center;
  background-color: #FBE1B7;
  background-size: contain;
}

#settings {
  height: 230px;
  overflow: hidden;
  display:block;
  transition: height 1.5s ease;
}

.hide-settings {
  height: 0px !important;
}


.minimized {
  height: 0;
}

#camera {
  transition: height 1s ease;
  transition-delay: 0.2s;
}

.webcam-big {
  height: 80px !important;
}

.maximized {
  height: unset !important;
}

.button-item {
  position: absolute;
  width: 100px;
  transition: all 0.6s ease;
  top: 00px;
  left: 150px;
}

.center {
  left: 150px;
}

.down {
  top: 260px;
}

.right {
  left: 280px;
}

.left {
  left: 10px;
}

.far-left {
  left: -150px;
}

.hide {
  opacity: 0;
  z-index: 1 !important;
}

.hidepreivew {
  opacity: 0;
}

#preview-screen-only {
  transition: all 1s ease;
  width: 100%;
  height: 100%;
  object-fit: cover;
}



.icon-blink {
  color: red;
  animation: blinker 1.5s cubic-bezier(.5, 0, 1, 1) infinite alternate;
}

@keyframes blinker {
  from {
    opacity: 1;
  }

  to {
    opacity: 0;
  }
}
</style>