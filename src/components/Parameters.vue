<template>
    <q-page class="flex flex-center column">

    <div class="q-gutter-sm">
        <q-radio v-model="shotType" val="Standard shots" label="Standard shots" />
        <q-radio v-model="shotType" val="Last shots" label="Last shots" />
    </div>

    <div class="q-gutter-sm">
        <q-radio v-model="gunModel" val="G17" label="G17" />
        <q-radio v-model="gunModel" val="G19" label="G19" />
        <q-radio v-model="gunModel" val="G45" label="G45" />
    </div>
    <span class="row">
        <q-input outlined v-model="textInput" label="Paste rows from excel" />
            <q-btn color="white" text-color="black" label="Parse text input" @click="parseTextInput">
        </q-btn>
    </span>
    <span class="row">
        <q-input outlined v-model="newEpoch.start" label="Range start" />
        <q-input outlined v-model="newEpoch.minX" label="Minimum X Accum" />
        <q-input outlined v-model="newEpoch.minY" label="Minimum Y Accum" />
        <q-input outlined v-model="newEpoch.minZ" label="Minimum Z Accum" />
        <q-input outlined v-model="newEpoch.minTotal" label="Minimum Total Accum" />
    </span>
    <span class="row">
        <q-input outlined v-model="newEpoch.end" label="Range end" />
        <q-input outlined v-model="newEpoch.maxX" label="Maximum X Accum" />
        <q-input outlined v-model="newEpoch.maxY" label="Maximum Y Accum" />
        <q-input outlined v-model="newEpoch.maxZ" label="Maximum Z Accum" />
        <q-input outlined v-model="newEpoch.maxTotal" label="Maximum Total Accum" />
    </span>
    <span class="row">
        <q-btn color="white" text-color="black" label="Add ROI" @click="addEpoch"/>
    </span>
    <q-btn color="white" text-color="black" label="Clear ROIs" @click="clearEpochs" />
    <q-table 
      title="ROIs"
      :data="epochs ? epochs : null"
      :columns="roiColumns"
      row-key="name"
    />
    </q-page>
</template>

<script>
export default {
    props: ["epochs"],
    data() {
        return {
            textInput: "",
            shotType:"",
            gunModel: "",
            newEpoch: {
                start: null,
                end: null,
                minX: null,
                maxX: null,
                minY: null,
                maxY: null,
                minZ: null,
                maxZ: null,
                minTotal: null,
                maxTotal: null,
            },
            roiColumns: [
                { name: 'start', label: 'Start of range', field: 'start' },
                { name: 'end', label: 'End of range', field: 'end' },
                { name: 'minX', label: 'Min X sum', field: 'minX' },
                { name: 'minY', label: 'Min Y sum', field: 'minY' },
                { name: 'minZ', label: 'Min Z sum', field: 'minZ' },
                { name: 'minSum', label: 'Min total sum', field: 'minTotal' },
                { name: 'maxX', label: 'Max X sum', field: 'maxX' },
                { name: 'maxY', label: 'Max Y sum', field: 'maxY' },
                { name: 'maxZ', label: 'Max Z sum', field: 'maxZ' },
                { name: 'maxTotal', label: 'Max total sum', field: 'maxTotal' },
                { name: 'shotType', label: 'Shot type', field: 'shotType' },
                { name: 'gunModel', label: 'Gun model', field: 'gunModel' },
            ],
        }
    },
    methods: {
        clearEpochs(){
            this.$emit('clear-epochs');
        },
        addEpoch(){
            if (!(this.newEpoch.start && this.newEpoch.end && this.newEpoch.minX && this.newEpoch.maxX && this.newEpoch.minY && this.newEpoch.maxY && this.newEpoch.minZ && this.newEpoch.maxZ && this.shotType && this.gunModel)){
                return;
            }
            this._addEpoch({
                ...this.newEpoch,
                shotType: this.shotType,
                gunModel: this.gunModel,
            });
            this.newEpoch = {
                start: null,
                end: null,
                minX: null,
                maxX: null,
                minY: null,
                maxY: null,
                minZ: null,
                maxZ: null,
                minTotal: null,
                maxTotal: null,
            };

        },
        _addEpoch(epoch){
            this.$emit('add-epoch', {
                ...epoch,
                shotType: this.shotType,
                gunModel: this.gunModel,
            });
        },
        parseTextInput(){
            var arr = this.textInput.split(/[\s]+/);
            
            // get ranges
            var endOfRanges = arr.indexOf("X");
            var rangesRaw = arr.slice(0, endOfRanges);

            // get accums
            var endOfHeaders = arr.lastIndexOf("SUM");
            var accumsRaw = arr.slice(endOfHeaders + 1);

            var rowSize = rangesRaw.length * 2;
            var num = rangesRaw.length / 2;

            for (let i = 0; i < num; i++){
                var ep = {
                start: rangesRaw[i * 2],
                end: rangesRaw[(i * 2) + 1],
                minX: accumsRaw[(i * 4)],
                minY: accumsRaw[(i * 4) + 1],
                minZ: accumsRaw[(i * 4) + 2],
                minTotal: accumsRaw[(i * 4) + 3],
                maxX: accumsRaw[(i * 4) + (rowSize)],
                maxY: accumsRaw[(i * 4) + 1 + (rowSize)],
                maxZ: accumsRaw[(i * 4) + 2 + (rowSize)],
                maxTotal: accumsRaw[(i * 4) + 3 + (rowSize)],
                };
                this._addEpoch(ep);
            }
        }
    }
}
</script>