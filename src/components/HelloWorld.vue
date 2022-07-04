<template>
  <div class="content">
    <div class="control_panel">
      <button
        class="save_btn"
        @click="step === 'routeEdit' ? saveRoute() : startRoute()"
        :disabled="
          (step === 'routeEdit' && routeSettingFlag) || step === 'routrEdit'
        "
      >
        {{ step === "routeEdit" ? "保存路線" : "添加路線" }}
      </button>
      <div class="discription">
        <ul>
          <li>點擊右鍵新增起訖點</li>
          <li>單擊左鍵新增中繼點，雙擊刪除節點</li>
        </ul>
      </div>
    </div>
    <div id="map"></div>
  </div>
</template>

<script>
import { Loader } from "@googlemaps/js-api-loader";
export default {
  name: "HelloWorld",
  data() {
    return {
      routePoints: [],
      savedRoutes: [],
      polyLinePoint: {},
      directionsRenderer: null,
      map: null,
      loader: null,
      distancePopup: null,
      startPointIcon: require("../assets/startPoint@0.25x.png"),
      endPointIcon: require("../assets/endPoint@0.25x.png"),
      midPointIcon: require("../assets/midPoint@0.5x.png"),
      routeSettingFlag: true,
      distance: 0,
      step: "siteEdit",
      tipContent: false,
      //STEP OPTION："routeEdit","siteEdit"
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
      this.routePoints = [];
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
        let div = document.createElement("div");
        let mapDiv = document.getElementById("map")
        div.setAttribute("id", "tip_content");
        let newContent = document.createTextNode("Hi there and greetings!");
        div.appendChild(newContent);
        div.style.backgroundColor = "white";
        div.style.borderRadius = "4px";
        div.style.padding = "4px 8px";
        div.style.boxShadow = "2px 2px 4px black";
        div.style.position = "fixed";
        div.style.zIndex = "9999";
        mapDiv.appendChild(div)
        //click event
        this.map.addListener("mousemove", function (e) {
          div.style.top = `${e.domEvent.pageY-30}px`;
          div.style.left = `${e.domEvent.pageX+5}px`;
        });
        this.map.addListener("click", function (event) {
          console.log(event);
          let pos = {
            lat: event.latLng.toJSON().lat,
            lng: event.latLng.toJSON().lng,
          };
          if (
            that.routePoints.length !== 0 &&
            that.routeSettingFlag &&
            that.step === "routeEdit"
          ) {
            that.routePoints.push(pos);
            that.routing(that.map);
            that.addPoint(pos, that.map, false);
          }
        });
        this.map.addListener("rightclick", function (event) {
          console.log(that.step);
          console.log(that.routeSettingFlag);
          let pos = {
            lat: event.latLng.toJSON().lat,
            lng: event.latLng.toJSON().lng,
          };
          if (that.routeSettingFlag && that.step === "routeEdit") {
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
          url: this.step === "routeEdit" ? this.midPointIcon : "",
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
        console.log("dblc");
        if (this.step === "routeEdit") {
          if (
            //完成後中途點刪除
            !this.routeSettingFlag &&
            pos !== this.routePoints[0] &&
            pos !== this.routePoints[this.routePoints.length - 1]
          ) {
            this.distancePopup.close();
            this.polyLinePoint = {};
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
            this.polyLinePoint = {};
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
          } else {
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

          if (
            that.routePoints.length > 1 &&
            !that.routeSettingFlag &&
            that.step === "routeEdit"
          ) {
            console.log(that.distance + "<---");
            that.distancePopup = new window.google.maps.InfoWindow();

            that.distancePopup.setOptions({
              content: `<div>${that.distance} 公尺</div>`,
              position: that.routePoints[that.routePoints.length - 1],
            });
            that.distancePopup.open(map);
            let points = [];
            res.routes[0].overview_path.forEach((point) => {
              points.push(point);
            });
            that.polyLinePoint = {
              name: `route${that.savedRoutes.length + 1}`,
              points: points,
            };
          }

          console.log(res.routes[0]);
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
    startRoute() {
      this.step = "routeEdit";
      this.routeSettingFlag = true;
      this.initMap();
      console.log("start");
    },
    saveRoute() {
      //road save api here
      this.routeSettingFlag = true;
      this.step = "siteEdit";
      this.savedRoutes.push(this.polyLinePoint);
      console.log(this.savedRoutes);
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
        //定位
        navigator.geolocation.getCurrentPosition((position) => {
          const pos = {
            lat: position.coords.latitude,
            lng: position.coords.longitude,
          };
          this.map.setCenter(pos);
        });
        this.savedRoutes.forEach((route, idx) => {
          console.log(route);
          for (let i = 0; i < route.points.length - 1; i++) {
            let line = [route.points[i], route.points[i + 1]];
            console.log(line);
            console.log(i + " " + (i + 1));
            const polyRoute = new window.google.maps.Polyline({
              path: line,
              geodesic: true,
              strokeColor: this.filterLineColor(idx),
              strokeOpacity: 0.8,
              strokeWeight: 5,
            });

            polyRoute.setMap(this.map);
          }

          const startMarker = new window.google.maps.Marker();
          startMarker.setOptions({
            position: route.points[0],
            map: this.map,
            zIndex: 1100,
            icon: {
              url: this.startPointIcon,
              size: new window.google.maps.Size(40, 40),
            },
          });

          const endMarker = new window.google.maps.Marker();
          endMarker.setOptions({
            position: route.points[route.points.length - 1],
            map: this.map,
            zIndex: 1100,
            icon: {
              url: this.endPointIcon,
              size: new window.google.maps.Size(40, 40),
            },
          });
        });
      });
    },
    filterLineColor(num) {
      // 路线颜色
      let temp = num % 5;
      let color = "";
      switch (temp) {
        case 0:
          color = "#1890FF";
          break;
        case 1:
          color = "#ff8514";
          break;
        case 2:
          color = "#C12D2D";
          break;
        case 3:
          color = "#F52F5F";
          break;
        case 4:
          color = "#B242F6";
          break;
        default:
          color = "#1890FF";
          break;
      }
      return color;
    },
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.content {
  width: 1440px;
  height: 100%;
  padding: 12px 0;
}
#map {
  width: 100%;
  height: 90%;
}
a {
  color: #42b983;
}
.control_panel {
  height: 10%;
  display: flex;
  flex-direction: row;
  justify-content: flex-start;
  align-items: center;
  margin: auto;
}
ul {
  text-align: left;
}
.save_btn {
  height: 48px;
  width: 120px;
  background-color: #1691e9;
  color: #fff;
  border-radius: 8px;
  font-size: 16px;
  font-weight: bold;
  box-shadow: none;
  border: none;
  cursor: pointer;
}
.save_btn:hover {
  background-color: #0069b5;
}
.save_btn:disabled {
  background-color: #a0cbeb;
  cursor: not-allowed;
}
</style>
