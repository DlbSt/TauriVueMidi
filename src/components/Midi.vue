<template>
  <div id="gridContainer">
    <div class="content-block dx-card responsive-paddings cards">
      <h3 class="content-block cards">Input devices</h3>
      <div class="content-block lists">
        <DxList :items="inputDeviceNames" :activeStateEnabled="false" :focusStateEnabled="false" :hoverStateEnabled="false" noDataText=" "  />
      </div>
    </div>
    <div class="content-block dx-card responsive-paddings cards">
      <h3 class="content-block cards">Output devices</h3>
      <div class="content-block lists">
        <DxList :items="outputDeviceNames" :activeStateEnabled="false" :focusStateEnabled="false" :hoverStateEnabled="false" noDataText=" " />
      </div>
    </div>
    <div
      id="messages"
      class="content-block dx-card responsive-paddings messagecard"
    >
      <h3 class="content-block cards">Messages</h3>
    </div>
    <div id="buttonBar">
      <div>
        <DxButton
          :width="200"
          style="margin-bottom: 10px"
          text="Refresh"
          @click="requestMidiDevices"
          type="normal"
          styling-mode="contained"
        />
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref, reactive, resolveDynamicComponent } from "vue";
import DxButton from "devextreme-vue/button";
import DxList from "devextreme-vue/list";

import {
  isPermissionGranted,
  sendNotification,
} from "@tauri-apps/api/notification";
import { register } from "@tauri-apps/api/globalShortcut";

interface MidiMessage {
  timestamp: Date,
  device: String,
  data: String
}

let inputDevices: WebMidi.MIDIInputMap;
let outputDevices: WebMidi.MIDIOutputMap;

let inputDeviceNames = ref<string[]>([]);
let outputDeviceNames = ref<string[]>([]);
let deviceMessages = ref<MidiMessage[]>([]);

let permissionGranted = false;

//https://stackoverflow.com/questions/73834911/webmidi-how-to-call-removelistener

onMounted(async () => {
  permissionGranted = await isPermissionGranted();

  await register("CommandOrControl+R", () => {
    console.log("Devices refreshed!");
  });

  await requestMidiDevices();
});

let requestMidiDevices = async (): Promise<void> => {
  const midi: WebMidi.MIDIAccess = await navigator.requestMIDIAccess();

  inputDevices = midi.inputs;
  outputDevices = midi.outputs;

  inputDeviceNames.value = [];
  outputDeviceNames.value = [];

  console.log("Input devices: ", inputDevices.size);
  console.log("Output devices: ", outputDevices.size);

  for (const input of inputDevices) {
    const [, device] = input;

    inputDeviceNames.value.push(device.name ?? "Undefined name");

    device.onmidimessage = (message) => {
      deviceMessages.value.push({
        data: message.data.toString(),
        device: message.currentTarget.name,
        timestamp: new Date(Date.now())
      } as MidiMessage);
    };

    if (permissionGranted) {
      sendNotification({
        title: "Input device found",
        body:
          device.name
      });
    }

    console.log(
      `Input port [type:'${device.type}']` +
        ` id:'${device.id}'` +
        ` manufacturer:'${device.manufacturer}'` +
        ` name:'${device.name}'` +
        ` version:'${device.version}'`
    );
  }

  for (const output of outputDevices) {
    const [, device] = output;

    outputDeviceNames.value.push(device.name ?? "Undefined name");

    if (permissionGranted) {
      sendNotification({
        title: "Output device found",
        body:
          device.name,
      });
    }

    console.log(
      `Output port [type:'${device.type}']` +
        `id:'${device.id}'` +
        `manufacturer:'${device.manufacturer}'` +
        `name:'${device.name}'` +
        `version:'${device.version}'`
    );
  }
};
</script>

<style scoped>
#gridContainer {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  grid-template-rows: 3fr 7fr auto;
  row-gap: 10px;
  width: 100%;
  height: 100vh;
}

#messages {
  grid-row: 2;
  grid-column: 1/3;
}

#buttonBar {
  grid-row: 3;
  grid-column: 1/3;
  text-align: center;
}

h3 {
  font-size: 24px;
  font-weight: lighter;
}

.cards {
  margin: 10px 10px 0px 10px;
  padding: 5px;
}

.messagecard {
  margin: 0px 10px;
  padding: 10px;
}

.lists {
  height: calc(100% - 50px);
}

.dx-list {
  height: 100% !important;
}
</style>
