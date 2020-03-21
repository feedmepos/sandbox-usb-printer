<template>
  <q-page>
    <div>
      <h6>USB Printer</h6>
      <q-btn v-if="hasChromePrinter" @click="getDeviceClicked('chrome')">
        Get Devices (Chrome)
      </q-btn>
      <div v-if="!hasChromePrinter">No Driver detected</div>
      <div v-for="(d, i) in devices" :key="i">
        {{ d }}
        <q-btn @click="sendText(d)">sendText</q-btn>
        <q-btn @click="sendImage(d)">sendImage</q-btn>
        <q-btn @click="sendImageV2(d)">sendImageV2</q-btn>
      </div>
      <q-input v-model.number="height" type="number" @input="onHeightInput" />
      <canvas ref="canvas" width="200" height="200" />
    </div>
  </q-page>
</template>

<script>
import Vue from "vue";
import Printer from "escpos/printer";
import Console from "escpos/adapter/console";
import Image from "escpos/image";

export default Vue.extend({
  data() {
    return {
      height: 200,
      result: "",
      devices: [],
      printer: null
    };
  },
  computed: {
    hasChromePrinter() {
      return chrome && chrome.usb;
    },
    canvas() {
      return this.$refs.canvas;
    },
    ctx() {
      return this.canvas.getContext("2d");
    }
  },
  mounted() {
    this.$q.loading.hide();
    console.log(chrome);
    if (this.canvas) {
      this.draw();
    }
  },
  methods: {
    draw() {
      this.ctx.beginPath();
      this.ctx.rect(20, 20, 200 - 40, this.height - 40);
      this.ctx.stroke();
    },
    onHeightInput(height) {
      this.canvas.height = Number(height);
      this.$nextTick(() => {
        this.draw();
      });
    },
    getDeviceClicked(plugin) {
      this.$q.loading.show();
      if (plugin === "chrome") {
        chrome.usb.getDevices(
          {},
          result => {
            this.$q.loading.hide();
            this.devices = result;
            console.log(result);
          },
          err => {
            this.$q.loading.hide();
            console.error(err);
          }
        );
      }
    },
    sendText(device) {
      console.log(device);
      const printer = this.getPrinter(device, this.handler);
      printer.text("This is test print from feedme pos android usb tester");
      printer.cut();
      printer.close();
    },
    getPrinter(device, handler) {
      return new Printer(new Console(handler(device)));
    },
    sendImage(device) {
      const printer = this.getPrinter(device, this.handler);
      Image.load(this.canvas.toDataURL(), image => {
        printer.image(image);
        printer.cut();
        printer.close();
      });
    },
    sendImageV2(device) {
      const printer = this.getPrinter(device, this.handlerV2);
      Image.load(this.canvas.toDataURL(), image => {
        printer.image(image);
        printer.cut();
        printer.close();
      });
    },
    handlerV2(device) {
      return data => {
        console.log("received data", data);
        chrome.usb.openDevice(device, function(handle) {
          console.log("received handle", handle);
          chrome.usb.listInterfaces(handle, function(descriptors) {
            console.log("receivve descriptor", descriptors);
            let inEndpoint = null;
            let outEndpoint = null;
            for (var index = 0; index < descriptors.length; index++) {
              const printerInterface = descriptors[index];
              console.log("working on interface", printerInterface);
              for (var i = 0; i < printerInterface.endpoints.length; i++) {
                var endpointDescriptor = printerInterface.endpoints[i];
                console.log("working on endpoint", endpointDescriptor);
                if (endpointDescriptor.type == "bulk") {
                  if (endpointDescriptor.direction == "out") {
                    outEndpoint = endpointDescriptor;
                  } else if (endpointDescriptor.direction == "in") {
                    inEndpoint = endpointDescriptor;
                  }
                }
                if (inEndpoint != null && outEndpoint != null) {
                  console.log("has both endpoint", inEndpoint, outEndpoint);
                  chrome.usb.claimInterface(
                    handle,
                    printerInterface.interfaceNumber,
                    function() {
                      console.log("interface claimed");
                      const content = {
                        direction: "out",
                        endpoint: outEndpoint.address,
                        data: data.buffer,
                        timeout: 5000
                      };
                      console.log("starting bulk transfer", content);
                      chrome.usb.bulkTransfer(handle, content, function(info) {
                        console.log("completed bulk transfer", info);
                        chrome.usb.releaseInterface(
                          handle,
                          printerInterface.interfaceNumber
                        );
                      });
                    }
                  );
                }
              }
            }
          });
        });
      };
    },
    handler(device) {
      return data => {
        console.log("received data", data);
        chrome.usb.openDevice(device, function(handle) {
          console.log("received handle", handle);
          chrome.usb.listInterfaces(handle, function(descriptors) {
            console.log("receivve descriptor", descriptors);
            let inEndpoint = null;
            let outEndpoint = null;
            for (var index = 0; index < descriptors.length; index++) {
              const printerInterface = descriptors[index];
              console.log("working on interface", printerInterface);
              for (var i = 0; i < printerInterface.endpoints.length; i++) {
                var endpointDescriptor = printerInterface.endpoints[i];
                console.log("working on endpoint", endpointDescriptor);
                if (endpointDescriptor.type == "bulk") {
                  if (endpointDescriptor.direction == "out") {
                    outEndpoint = endpointDescriptor;
                  } else if (endpointDescriptor.direction == "in") {
                    inEndpoint = endpointDescriptor;
                  }
                }
                if (inEndpoint != null && outEndpoint != null) {
                  console.log("has both endpoint", inEndpoint, outEndpoint);
                  chrome.usb.claimInterface(
                    handle,
                    printerInterface.interfaceNumber,
                    function() {
                      console.log("interface claimed");
                      const content = {
                        direction: "out",
                        endpoint: outEndpoint.address,
                        data: data.buffer,
                        timeout: 5000
                      };
                      console.log("starting bulk transfer", content);
                      chrome.usb.bulkTransfer(handle, content, function(info) {
                        console.log("completed bulk transfer", info);
                        chrome.usb.releaseInterface(
                          handle,
                          printerInterface.interfaceNumber,
                          function(result) {
                            chrome.usb.closeDevice(handle);
                            console.log("interface released", result);
                          }
                        );
                      });
                    }
                  );
                }
              }
            }
          });
        });
      };
    }
  }
});
</script>
