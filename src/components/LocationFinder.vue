<template>
  <div>
    <h1>Location Finder</h1>

    <!-- Search Input -->
    <input v-model="searchQuery" @keyup.enter="searchLocation" placeholder="Search for a location" class="search-input"/>
    <button @click="searchLocation">Search</button>

    <!-- Get Current Location Button -->
    <button @click="getCurrentLocation">Get Current Location</button>

    <!-- Map Component -->
    <div id="map">
      <LocationMap :userLocation="userLocation" :locations="locations" ref="locationMap" />
    </div>

    <!-- Add Location Button -->
    <button @click="addLocation">Add Location</button>

    <!-- Location Table -->
    <table>
      <thead>
        <tr>
          <th><input type="checkbox" v-model="selectAll" @change="selectAllLocations" /></th>
          <th>Location</th>
          <th>Time Zone</th>
          <th>Local Time</th>
        </tr>
      </thead>
      <tbody>
        <!-- Display paginated location records here -->
        <tr v-for="location in paginatedLocations" :key="location.id" @click="handleLocationRowClick(location)">
          <td><input type="checkbox" v-model="selectedLocations" :value="location" @click.stop /></td>
          <td>{{ location.name }}</td>
          <td>{{ location.timezone }}</td>
          <td>{{ location.localTime }}</td>
        </tr>
      </tbody>
    </table>

    <!-- Pagination Controls -->
    <div>
      <button @click="changePage(-1)" :disabled="currentPage === 1">Previous</button>
      <button @click="changePage(1)" :disabled="currentPage === totalPages || locations && (locations.length <= 10)">Next</button>
    </div>

    <!-- Delete Selected Button -->
    <button @click="deleteSelectedLocations" id="deleteSelected">Delete Selected</button>
  </div>
</template>

<script>
import LocationMap from './LocationMap.vue'

let googleMaps = null;

export default {
  name: 'LocationFinder',
  components: {
    LocationMap
  },
  data() {
    return {
      searchQuery: '',
      userLocation: {lat:0, lng:0},
      locations: [], // Store searched locations
      searchResults: [], // Store search results
      selectedLocations: [], // Store selected locations
      currentPage: 1,
      perPage: 10,
    };
  },
  computed: {
    // Calculate the paginated locations based on currentPage and perPage
    paginatedLocations() {
      const start = (this.currentPage - 1) * this.perPage;
      const end = start + this.perPage;
      return this.locations.slice(start, end);
    },
    totalPages() {
      return Math.ceil(this.locations.length / this.perPage);
    },
  },
  methods: {
    // Method to get the current location
    getCurrentLocation() {
      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition(
          (position) => {
            this.userLocation = {
              lat: position.coords.latitude,
              lng: position.coords.longitude,
            };
          },
          (error) => {
            console.error(`Error getting current location: ${error.message}`);
          }
        );
      } else {
        console.error('Geolocation is not supported by this browser.');
      }
    },

    // Method to search for a location
    searchLocation() {
      if (this.searchQuery) {
        const placesService = new googleMaps.maps.places.PlacesService(this.$refs.locationMap.map);
        const request = {
          query: this.searchQuery,
          fields: ['geometry', 'name', 'formatted_address'],
        };

        placesService.findPlaceFromQuery(request, (results, status) => {
          if (status === googleMaps.maps.places.PlacesServiceStatus.OK) {
            // Extract the latitude and longitude of locations
            const latLngArray = results.map(result => {
              this.userLocation = {
                lat: result.geometry.location.lat(),
                lng: result.geometry.location.lng(),
              }
            });
            console.log('Latitude and Longitude of locations from findPlaceFromQuery:', latLngArray);
          }
        });
      }
    },

    addLocation() {
      const geocoder = new googleMaps.maps.Geocoder();
      const namelatlng = new googleMaps.maps.LatLng(this.userLocation.lat, this.userLocation.lng);

      let locationName = 'User Location';

      geocoder.geocode({ location: namelatlng }, (results, status) => {
        if (status === googleMaps.maps.GeocoderStatus.OK) {
          locationName = results[0].formatted_address;
        }

        const timezonelatlng = `${this.userLocation.lat},${this.userLocation.lng}`;
        const timestamp = Math.floor(Date.now() / 1000); // Current timestamp in seconds

        // Make a request to the Google Time Zone API
        fetch(`https://maps.googleapis.com/maps/api/timezone/json?location=${timezonelatlng}&timestamp=${timestamp}&key=AIzaSyCTBz-qB7IR8XP-QL74XM07HQqVC0tX36E`)
          .then(response => response.json())
          .then(timeZoneData => {
            if (timeZoneData.status === 'OK') {
              const timezone = timeZoneData.timeZoneId;

              // Create a new location object with the retrieved information
              const newLocation = {
                lat: this.userLocation.lat,
                lng: this.userLocation.lng,
                name: locationName,
                timezone: timezone,
              };

              // Calculate the local time using the time zone
              const localTime = new Date().toLocaleString('en-US', { timeZone: timezone });
              newLocation.localTime = localTime;

              // Add the newLocation to the locations array
              this.locations.push(newLocation);

              this.userLocation = { lat: 0, lng: 0 };

              // Log the updated locations array
              console.log('Updated Locations:', this.locations);
            }
          })
          .catch(error => {
            console.error('Error fetching time zone data:', error);
          });
        });
    },

    handleLocationRowClick(location) {
      this.showLocationMarker(location);
    },

    // Method to show the marker on the map for a selected location
    showLocationMarker(location) {
      this.userLocation = { lat: location.lat, lng: location.lng };
    },

    // Method to delete selected locations
    deleteSelectedLocations() {
      if (this.selectedLocations.length > 0) {
        // Iterate through the selected locations
        this.selectedLocations.forEach(selectedLocation => {
          // Find the index of the selected location in the locations array
          const index = this.locations.findIndex(location => location.id === selectedLocation.id);

          // If the selected location is found, remove it from the locations array
          if (index !== -1) {
            this.locations.splice(index, 1);
          }
        });

        // Clear the selectedLocations array
        this.selectedLocations = [];
        if(this.selectAll){
          this.selectAll = false;
        }

        this.userLocation = { lat: 0, lng: 0 };

        console.log('Selected locations deleted. Updated locations:', this.locations);
      } else {
        console.warn('No selected locations to delete.');
      }
    },

    // Method to change the current page for pagination
    changePage(direction) {
      if (direction === 1) {
        if (this.currentPage < this.totalPages) {
          this.currentPage++;
        }
      } else if (direction === -1) {
        if (this.currentPage > 1) {
          this.currentPage--;
        }
      }
    },

    // Method to select or deselect all locations
    selectAllLocations() {
      if (this.selectAll) {
        this.selectedLocations = this.paginatedLocations.slice(); // Select all on the current page
      } else {
        this.selectedLocations = [];
      }
    },
    initializeGoogleMaps() {
      googleMaps = window.google;
      this.getCurrentLocation();
    },
  },
  mounted() {
    if (!googleMaps) {
      const script = document.createElement('script');
      script.src = `https://maps.googleapis.com/maps/api/js?key=` + process.env.VUE_APP_GOOGLE_MAPS_API_KEY + `&libraries=places`;
      script.async = true;
      script.defer = true;

      script.onload = () => {
        this.initializeGoogleMaps();
      };

      document.head.appendChild(script);
    } else {
      this.initializeGoogleMaps();
    }
  },
};
</script>

<style>
body, h1, h2, p, ul, li, table {
  margin: 0;
  padding: 0;
}

body {
  font-family: Arial, sans-serif;
  background-color: #f0f0f0;
  margin: 20px;
  padding: 20px;
}

h1 {
  text-align: center;
  margin-bottom: 20px;
  color: #333;
}

button {
  background-color: #007BFF;
  color: #fff;
  border: none;
  padding: 10px 20px;
  margin-bottom: 10px;
  cursor: pointer;
}

input[type="text"] {
  padding: 10px;
  width: 100%;
  border: 1px solid #ccc;
  border-radius: 5px;
  margin-bottom: 10px;
}

.search-input {
  padding: 10px;
  width: 50%;
  border: 1px solid #ccc;
  border-radius: 5px;
  margin-bottom: 10px;
  font-size: 16px;
  background-color: #f7f7f7;
  color: #333;
  transition: border 0.3s, background-color 0.3s;
}

.search-input:focus {
  border: 1px solid #007BFF;
  background-color: #fff;
}

#map {
  width: 100%;
  height: 400px;
  margin-bottom: 20px;
}

table {
  width: 100%;
  border-collapse: collapse;
  background-color: #fff;
}

th, td {
  border: 1px solid #ccc;
  padding: 10px;
  text-align: left;
}

th {
  background-color: #007BFF;
  color: #fff;
}

tbody tr:nth-child(even) {
  background-color: #f2f2f2;
}

tbody tr:hover {
  background-color: #d9edf7;
}

button[disabled] {
  background-color: #ccc;
  cursor: not-allowed;
}

/* Pagination Controls */
div button {
  margin: 10px;
}

/* Checkbox Styling */
input[type="checkbox"] {
  transform: scale(1.5); /* Increase the checkbox size */
}

/* Delete Selected Button */
#deleteSelected {
  background-color: #FF6347; /* Tomato red */
  color: #fff;
}
</style>
