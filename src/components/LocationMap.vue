<template>
  <div>
    <div ref="map" style="width: 100%; height: 400px;"></div>
  </div>
</template>

<script>
import { ref, onMounted, watch } from 'vue';

export default {
  props: {
    userLocation: Object,
    locations: Array,
  },
  setup(props) {
    const map = ref(null);
    let googleMaps = null;
    let userLocationMarker = null;
    const locationMarkers = new Map(); // Use a Map to store location markers

    const initializeMap = () => {
      if (!map.value || !window.google) {
        return;
      }

      googleMaps = window.google;

      const center = props.userLocation || { lat: 0, lng: 0 };

      const mapOptions = {
        center,
        zoom: 8,
      };

      const mapInstance = new googleMaps.maps.Map(map.value, mapOptions);

      if (props.userLocation) {
        userLocationMarker = new googleMaps.maps.Marker({
          position: props.userLocation,
          map: mapInstance,
          title: 'User Location',
        });
      }

      updateLocationMarkers(mapInstance, props.locations);
    };

    const updateLocationMarkers = (mapInstance, locations) => {
      // Remove old location markers
      locationMarkers.forEach((marker) => {
        marker.setMap(null);
      });

      locationMarkers.clear(); // Clear all markers from the Map

      // Create new location markers
      if (googleMaps && locations) {
        locations.forEach((location, index) => {
          const locationKey = `${location.lat}_${location.lng}`; // Unique key for each location
          const marker = new googleMaps.maps.Marker({
            position: location,
            map: mapInstance,
            title: `Location ${index + 1}`,
          });

          locationMarkers.set(locationKey, marker); // Add the marker to the Map
        });
      }
    };

    onMounted(() => {
      if (typeof window.google === 'undefined') {
        const script = document.createElement('script');
        script.src = `https://maps.googleapis.com/maps/api/js?key=` + process.env.VUE_APP_GOOGLE_MAPS_API_KEY + `&libraries=places`;
        script.async = true;
        script.defer = true;
        script.onload = initializeMap;
        document.head.appendChild(script);
      } else {
        initializeMap();
      }
    });

    watch(() => props.userLocation, () => {
      if (userLocationMarker) {
        userLocationMarker.setMap(null);
      }
      initializeMap();
    });

    // Watch for changes in the locations prop
    watch(() => props.locations, (newLocations) => {
      // Update the markers for the new locations
      updateLocationMarkers(map.value, newLocations);
    });

    return { map };
  },
};
</script>