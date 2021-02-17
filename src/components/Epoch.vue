<template>
    <div class="text-body1">
        Start: {{epoch.start}}
        End: {{epoch.end}}
        MinX: {{epoch.minX}}
        MaxX: {{epoch.maxX}}
        MinY: {{epoch.minY}}
        MaxY: {{epoch.maxY}}
        MinZ: {{epoch.minZ}}
        MaxZ: {{epoch.maxZ}}
        MinTotal: {{epoch.minTotal}}
        MaxTotal: {{epoch.maxTotal}}
        <q-btn color="white" text-color="black" label="Edit" @click="$emit('edit-epoch')"/>
        <q-btn color="white" text-color="black" label="Delete" @click="$emit('delete-epoch')"/>
    </div>
</template>

<script>
export default {
    props: ["epoch"],
}
</script>