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
      distance: 0,
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
          mapId: "5216cd334b4588dc",
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
          let pos = {lat:event.latLng.toJSON().lat,lng:event.latLng.toJSON().lng}
          if (
            that.routePoints.length !== 0 &&
            that.routeSettingFlag &&
            that.step === 1
          ) {
            that.routePoints.push(pos);
            that.routing(that.map);
            that.addPoint(pos, that.map, false);
            
          }
        });
        this.map.addListener("rightclick", function (event) {
          let pos = {lat:event.latLng.toJSON().lat,lng:event.latLng.toJSON().lng}
          if (that.routeSettingFlag && that.step === 1) {
            that.routePoints.push(pos);
            that.routing(that.map);
            that.addPoint(pos, that.map, true);
            
          }
        });
        this.directionsRenderer = new window.google.maps.DirectionsRenderer();
      });
    },

    addPoint(pos, map, startEnd) {
      let icon;

      //icon判別
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


      if (
        this.routePoints.indexOf(pos) === this.routePoints.length - 1 &&
        startEnd &&
        this.routePoints.length > 1
      ) {
        console.log(this.distance+'<---');
        const infowindow = new window.google.maps.InfoWindow({
          content: `<div>${this.distance} 公尺</div>`,
        });
        infowindow.open({ anchor: marker, map });
      }


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
      const that = this;
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
          //unitSystem: window.google.maps.UnitSystem.METRIC
        };
        directionsService.route(directionReq, function (res) {
          that.distance = 0;
          res.routes[0].legs.forEach((leg) => {
            that.distance += leg.distance.value;
          });
          console.log(that.distance);
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
