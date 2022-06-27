<template>
  <div id="map"></div>
</template>

<script>
import { Loader } from "@googlemaps/js-api-loader";
export default {
  name: "HelloWorld",
  data() {
    return {
      routePoints: [],
      directionsRenderer: null,
    };
  },
  computed: {
    wayPoints() {
      let wayPoints = [];
      this.routePoints.slice(1, this.routePoints.length - 1).forEach((e) => {
        wayPoints.push({
          location: e,
          stopover: true,
        });
      });
      return wayPoints;
    },
  },
  created() {
    const loader = new Loader({
      apiKey: "AIzaSyDkxHGQuiLxyyVaRkvKrJRKb1-La-7T6nY",
      version: "weekly",
    });
    loader.load().then(() => {
      const map = new window.google.maps.Map(document.getElementById("map"), {
        center: { lat: -34.397, lng: 150.644 },
        zoom: 18,
        mapTypeControl: false,
        mapTypeId: "roadmap",
        clickableIcons: false,
      });
      this.initMap(map);
      this.directionsRenderer = new window.google.maps.DirectionsRenderer()
    });
  },
  
  methods: {
    initMap(map) {
      console.log('init')
      const that = this;
      map.setOptions({ draggableCursor: "crosshair" });
      navigator.geolocation.getCurrentPosition((position) => {
        const pos = {
          lat: position.coords.latitude,
          lng: position.coords.longitude,
        };
        map.setCenter(pos);
      });
      map.addListener("click", function (event) {
        that.addPoint(event.latLng.toJSON(), map);
        that.routing(map);
      });
    },
    addPoint(pos, map) {
      this.routePoints.push(pos);
      const marker = new window.google.maps.Marker({
        position: pos,
        map: map,
        zIndex: 1100,
      });
      marker.addListener("dblclick", () => {
        marker.setMap(null);
        this.routePoints = this.routePoints.filter((point) => point !== pos);
        this.routing(map);
      });
    },
    routing(map) {
      console.log("draw line");
      const directionsService = new window.google.maps.DirectionsService();
      //const directionsRenderer = new window.google.maps.DirectionsRenderer();
      let directionsDisplay = this.directionsRenderer;
      if (this.routePoints.length > 1) {
        let pointA = this.routePoints[0];
        let pointB = this.routePoints[this.routePoints.length - 1];

        let directionReq = {
          origin: pointA,
          destination: pointB,
          waypoints: this.wayPoints,
          travelMode: "WALKING",
        };
        directionsService.route(directionReq, function (res) {
          directionsDisplay.setDirections(res);
          directionsDisplay.setOptions({
            map: map,
            directions: res,
            preserveViewport: true,
            suppressMarkers: true,
          });
          directionsDisplay.setMap(map);
        });
      } else {
        directionsDisplay.setMap(null);
      }
    },
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
#map {
  width: 90vw;
  height: 90vh;
}
a {
  color: #42b983;
}
</style>
