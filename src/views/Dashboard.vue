<template>
  <div class="dashboard">
    <div class="dashboard__view">
      <MapWorld></MapWorld>
    </div>
    <div class="dashboard__side"></div>
  </div>
</template>

<script>
import MapWorld from '@/components/MapWorld.vue';

export default {
  name: 'Dashboard',
  components: { MapWorld },
};
</script>
