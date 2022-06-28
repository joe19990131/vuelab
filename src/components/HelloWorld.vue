<template>
  <div>
    <div id="map"></div>
    <button @click="saveRoute" :disabled="routeSettingFlag">save</button>
  </div>
</template>

<script>
import { Loader } from "@googlemaps/js-api-loader";
export default {
  name: "HelloWorld",
  data() {
    return {
      routePoints: [],
      directionsRenderer: null,
      map: null,
      loader: null,
      startPointIcon: require("../assets/startPoint@0.25x.png"),
      endPointIcon: require("../assets/endPoint@0.25x.png"),
      midPointIcon: require("../assets/midPoint@0.5x.png"),
      routeSettingFlag: true,
      step: 1,
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
    this.loader = new Loader({
      apiKey: "AIzaSyDkxHGQuiLxyyVaRkvKrJRKb1-La-7T6nY",
      version: "weekly",
    });
    this.initMap();
  },

  methods: {
    initMap() {
      const that = this;
      this.loader.load().then(() => {
        this.map = new window.google.maps.Map(document.getElementById("map"), {
          center: { lat: -34.397, lng: 150.644 },
          zoom: 18,
          mapTypeControl: false,
          mapTypeId: "roadmap",
          clickableIcons: false,
        });
        //設置預設游標樣式
        this.map.setOptions({ draggableCursor: "crosshair" });

        //地圖初始定位
        navigator.geolocation.getCurrentPosition((position) => {
          const pos = {
            lat: position.coords.latitude,
            lng: position.coords.longitude,
          };
          this.map.setCenter(pos);
        });

        //顯示保存路線
        if (
          this.routePoints.length !== 0 &&
          !this.routeSettingFlag &&
          this.step === 2
        ) {
          this.routing(this.map);
          const startMarker = new window.google.maps.Marker();
          startMarker.setOptions({
            position: this.routePoints[0],
            map: this.map,
            zIndex: 1100,
            icon: {
              url: this.startPointIcon,
              size: new window.google.maps.Size(40, 40),
            },
          });

          const endMarker = new window.google.maps.Marker();
          endMarker.setOptions({
            position: this.routePoints[this.routePoints.length - 1],
            map: this.map,
            zIndex: 1100,
            icon: {
              url: this.endPointIcon,
              size: new window.google.maps.Size(40, 40),
            },
          });
        }

        //click event
        this.map.addListener("click", function (event) {
          if (
            that.routePoints.length !== 0 &&
            that.routeSettingFlag &&
            that.step === 1
          ) {
            that.addPoint(event.latLng.toJSON(), that.map, false);
            that.routing(that.map);
          }
        });
        this.map.addListener("rightclick", function (event) {
          if (that.routeSettingFlag && that.step === 1) {
            that.addPoint(event.latLng.toJSON(), that.map, true);
            that.routing(that.map);
          }
        });
        this.directionsRenderer = new window.google.maps.DirectionsRenderer();
      });
    },

    addPoint(pos, map, startEnd) {
      this.routePoints.push(pos);
      let icon;
      if (this.routePoints.indexOf(pos) === 0 && startEnd) {
        icon = {
          url: this.startPointIcon,
          size: new window.google.maps.Size(40, 40),
        };
      } else if (
        this.routePoints.indexOf(pos) === this.routePoints.length - 1 &&
        startEnd &&
        this.routePoints.length > 1
      ) {
        icon = {
          url: this.endPointIcon,
          size: new window.google.maps.Size(40, 40),
        };
        this.routeSettingFlag = false;
      } else {
        icon = {
          url: this.step === 1 ? this.midPointIcon : "",
          anchor: new window.google.maps.Point(0, 10),
        };
      }
      const marker = new window.google.maps.Marker({
        position: pos,
        map: map,
        zIndex: 1100,
        icon: icon,
      });
      marker.addListener("dblclick", () => {
        if (this.step === 1) {
          if (
            //完成後中途點刪除
            !this.routeSettingFlag &&
            pos !== this.routePoints[0] &&
            pos !== this.routePoints[this.routePoints.length - 1]
          ) {
            this.routeSettingFlag = true;
            marker.setMap(null);
            this.routePoints = this.routePoints.filter(
              (point) => point !== pos
            );
            this.routing(map);
            this.routeSettingFlag = false;
          } else if (
            //完成後終點刪除
            pos === this.routePoints[this.routePoints.length - 1]
          ) {
            this.routeSettingFlag = true;
            marker.setMap(null);
            this.routePoints = this.routePoints.filter(
              (point) => point !== pos
            );
            this.routing(map);
          } else if (this.routePoints.length < 2) {
            this.routeSettingFlag = true;
            marker.setMap(null);
            this.routePoints = this.routePoints.filter(
              (point) => point !== pos
            );
            this.routing(map);
          }
        }
      });
    },

    routing(map) {
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

    saveRoute() {
      this.initMap();

      //road save api here

      this.step = 2;
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
