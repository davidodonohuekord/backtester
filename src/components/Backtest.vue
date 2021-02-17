<template>
<q-page class="flex flex-center column">
      <p class="text-body1">{{dataPath == null ? "No directory selected." : dataPath}}</p>
<q-btn color="white" text-color="black" :label="dataPath == null? 'Select directory' : 'Change directory'" @click="selectPath"/>
      <q-btn color="white" v-if="(dataPath != null) && !processing" text-color="black" label="Run Backtest" @click="backTest">
      </q-btn>
      <q-spinner v-if="processing"
        color="primary"
        size="3em"
      />

    <div v-if="resultsFailed.length > 0">
                  <q-btn color="white" v-if="resultsFailed.length > 0" text-color="black" label="Clear results" @click="clearResults">
      </q-btn>
      <q-table 
      title="Failed events"
      :data="resultsFailed"
      :columns="columns"
      row-key="name"
    />
    </div>
</q-page>
</template>

<script>

const electron = require('electron');
const dialog = electron.remote.dialog;
const fs = require('fs');
const path = require("path");

export default {
    props: ["epochs"],
    data() {
        return {
            processing: false,
            columns: [
                { name: 'filename', label: 'Filename', field: 'filename' },
                { name: 'index', label: 'Event Index', field: 'index' },
                { name: 'reason', label: 'Reason', field: 'reason' },
            ],
            resultsFailed: [],
            dataPath: null,
        }
    },
    methods: {
        clearResults(){
            this.resultsFailed = [];
        },
        getAllFiles(dir) {
            var files = [];
            fs.readdirSync(dir).forEach(file => {
                const absolute = path.join(dir, file);
                if (fs.statSync(absolute).isDirectory()) {
                    files = files.concat(this.getAllFiles(absolute));
                    }
                else{
                    files.push(absolute);
                }
            });
            return files;
        },
        backTest(){
            if (!(this.dataPath && this.epochs)){
                console.log("Missing information");
                return;
            }
            this.processing = true;
            this.clearResults();
            // get list of the full path of all files in all subfolders
            var allFiles = this.getAllFiles(this.dataPath);
            // loop over each file
            allFiles.forEach(filename => {
                // make sure it's a log file (and not a different file or directory)
                if (filename.split('.').pop() == "log"){
                    var shotType;
                    if (filename.includes("LAST_SHOTS")){
                        shotType = "Last shots"
                    } else if (filename.includes("STANDARD_SHOTS")){
                        shotType = "Standard shots"
                    }
                    var gunModel;
                    if (filename.includes("G17")){
                        gunModel = "G17";
                    } else if (filename.includes("G19")){
                        gunModel = "G19";
                    } else if (filename.includes("G45")){
                        gunModel = "G45";
                    }
                    // get entire contents of file
                    var fileContents = fs.readFileSync(filename, "ascii");
                    // split contents into an array of events
                    var eventArray = fileContents.split(" ..... EVENT DETECTED ..... ");
                    // for each event
                    eventArray.forEach((event, index) => {
                        // check whether it it an invalid event
                        var loggedShot = event.includes("$TERMINAL LOG: Shot");
                        // for later possible use (indicates whether shotdot detected event as a shot)
                        //var detectedShot = !event.includes("Detected Event Type: NOT SHOT");
                        var xAxis = "X Axis Data ";
                        var yAxis = "Y Axis Data ";
                        var zAxis = "Z Axis Data ";
                        var endOfZAxis = "Printing Vibration Sensor Data";
                        var xData = event.substring(event.lastIndexOf(xAxis) + xAxis.length, event.lastIndexOf(yAxis));
                        var xDataArray = xData.split(",").map(elem => Math.abs(parseInt(elem)));
                        var yData = event.substring(event.lastIndexOf(yAxis) + yAxis.length, event.lastIndexOf(zAxis));
                        var yDataArray = yData.split(",").map(elem => Math.abs(parseInt(elem)));
                        var zData = event.substring(event.lastIndexOf(zAxis) + zAxis.length, event.lastIndexOf(endOfZAxis));
                        var zDataArray = zData.split(",");
                        // pop here because the last element in the array is a bunch of hyphens and it's easier to parse that out here
                        zDataArray.pop();
                        zDataArray = zDataArray.map(elem => Math.abs(parseInt(elem)));
                        // if shot was logged on terminal, and event was detected as shot
                        // assume it passes and try to invalidate 
                        var passed = true;
                        // keep track of the reason for failure
                        var reason = "";
                        this.epochs.filter(elem => elem.shotType == shotType && elem.gunModel == gunModel).forEach(epoch => {
                            if (passed) {
                                // map is not mutable, slice is not mutable
                                // checks from start to end, inclusive
                                // var xBoolArray = xDataArray.slice(epoch.start, epoch.end).map(datapoint => datapoint >= epoch.minX && datapoint <= epoch.maxX);
                                // var yBoolArray = yDataArray.slice(epoch.start, epoch.end).map(datapoint => datapoint >= epoch.minY && datapoint <= epoch.maxY);
                                // var zBoolArray = zDataArray.slice(epoch.start, epoch.end).map(datapoint => datapoint >= epoch.minZ && datapoint <= epoch.maxZ);

                                // reduce does not mutate, +1 to be inclusive of end value
                                var xSumArray = xDataArray.slice(parseInt(epoch.start), parseInt(epoch.end) + 1).reduce((a, b) => a + b, 0);
                                var ySumArray = yDataArray.slice(parseInt(epoch.start), parseInt(epoch.end) + 1).reduce((a, b) => a + b, 0);
                                var zSumArray = zDataArray.slice(parseInt(epoch.start), parseInt(epoch.end) + 1).reduce((a, b) => a + b, 0);

                                // shot occurred, validate algorithm
                                if (xSumArray > parseInt(epoch.maxX)){
                                    if (loggedShot){
                                        passed = false;
                                        reason = "Missed shot: Sum of ROI " + epoch.start + " to " + epoch.end + " (" + xSumArray + ") exceeded " + epoch.maxX + " on X axis";
                                    }
                                } else if (ySumArray > parseInt(epoch.maxY)){
                                    if (loggedShot){
                                        passed = false;
                                        reason = "Missed shot: Sum of ROI " + epoch.start + " to " + epoch.end + " (" + ySumArray + ") exceeded " + epoch.maxY + " on Y axis";
                                    }
                                } else if (zSumArray > parseInt(epoch.maxZ)){
                                    if (loggedShot){
                                        passed = false;
                                        reason = "Missed shot: Sum of ROI " + epoch.start + " to " + epoch.end + " (" + zSumArray + ") exceeded " + epoch.maxZ + " on Z axis";
                                    }
                                } else if ((xSumArray + ySumArray + zSumArray) > parseInt(epoch.maxTotal)){
                                    if (loggedShot){
                                        passed = false;
                                        reason = "Missed shot: Sum of ROI " + epoch.start + " to " + epoch.end + " (" + (xSumArray + ySumArray + zSumArray) + ") exceeded " + epoch.maxTotal + " on all axes";
                                    }
                                } else if (xSumArray < parseInt(epoch.minX)){
                                    if (loggedShot){
                                        passed = false;
                                        reason = "Missed shot: Sum of ROI " + epoch.start + " to " + epoch.end + " (" + xSumArray + ") was less than " + epoch.minX + " on X axis";
                                    }
                                } else if (ySumArray < parseInt(epoch.minY)){
                                    if (loggedShot){
                                        passed = false;
                                        reason = "Missed shot: Sum of ROI " + epoch.start + " to " + epoch.end + " (" + ySumArray + ") was less than " + epoch.minY + " on Y axis";
                                    }
                                } else if (zSumArray < parseInt(epoch.minZ)){
                                    if (loggedShot){
                                        passed = false;
                                        reason = "Missed shot: Sum of ROI " + epoch.start + " to " + epoch.end + " (" + zSumArray + ") was less than " + epoch.minZ + " on Z axis";
                                    }
                                } else if ((xSumArray + ySumArray + zSumArray) < parseInt(epoch.minTotal)){
                                    if (loggedShot){
                                        passed = false;
                                        reason = "Missed shot: Sum of ROI " + epoch.start + " to " + epoch.end + " (" + (xSumArray + ySumArray + zSumArray) + ") was less than " + epoch.minTotal + " on all axes";
                                    }
                                } else if (!loggedShot) {
                                    passed = false;
                                    reason = "Event was not logged as a shot but passed all parameters";
                                }
                            }
                        });
                        if (!passed){
                            this.resultsFailed.push({
                                filename,
                                event,
                                index,
                                reason,
                                data: event
                            })
                        }
                    })
                }
            })
            this.processing = false;
        },
        selectPath(){
            const options = {
                properties: ["openDirectory", "dontAddToRecent"]
            }
            const savePath = dialog.showOpenDialogSync(null, options);
            this.dataPath = savePath ? savePath[0] : null;
        },
    }
}
</script>