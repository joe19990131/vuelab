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
      <div class="routes">
        <div class="route_item" v-for="(item, key) in savedRoutes" :key="key">
          {{ item.name }}
        </div>
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
      siteTypeList: [
        { name: "醫療站", icon: require("../assets/1.png") },
        { name: "AED醫療站", icon: require("../assets/2.png") },
        { name: "補給站", icon: require("../assets/3.png") },
      ],
      routeSettingFlag: true,
      distance: 0,
      step: "siteEdit",
      tipContent: "點擊右鍵開始繪製",
      centerLanLng: null,
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
          fullscreenControl: false,
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
          console.log(that.centerLanLng);
          if (that.centerLanLng) {
            that.map.fitBounds(that.centerLanLng);
          } else {
            that.map.setCenter(pos);
          }
        });
        if (this.step === "routeEdit") {
          this.tipEvent();
          this.initAutoComplete();
          this.siteBox();
        }

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
      this.tipContent = "點擊左鍵新增中繼點\n點擊右鍵結束繪製";
      this.tipEvent();
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
          anchor: new window.google.maps.Point(10, 10),
          size: new window.google.maps.Size(40, 40),
        };
      }

      const marker = new window.google.maps.Marker({
        position: pos,
        map: map,
        zIndex: 1100,
        icon: icon,
        //label: { text: "標", color: "#fff" },
      });

      marker.addListener("dblclick", (e) => {
        console.log(e.latLng.toJSON());
        
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
            that.distancePopup.addListener("visible", function (e) {
              console.log(e);
            });
            that.distancePopup.open(map);
            let points = [];
            console.log(res.routes[0].overview_path[0].toJSON());
            res.routes[0].overview_path.forEach((point) => {
              points.push(point.toJSON());
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
      this.tipContent = "點擊右鍵開始繪製";
      this.initMap();
      console.log("start");
    },
    geoCode(latLng){
      let req = {
        location:latLng,
      }
      let geocoder = new window.google.maps.Geocoder()
      geocoder.geocode(req,(results,status)=>{
        if(status === "OK"){
          console.log('geocode result:');
          console.log(results);
          console.log('geocode result city:');
          let addrCountry = results[results.length-1].formatted_address
          let addrCity = results[results.length-2].formatted_address
          console.log(addrCity.slice(addrCountry.length));

        }
      })
    },
    saveRoute() {
      //road save api here
      this.routeSettingFlag = true;
      this.step = "siteEdit";
      this.savedRoutes.push(this.polyLinePoint);
      
      console.log(this.polyLinePoint.points[parseInt(this.polyLinePoint.points.length/2)]);
      let geocodeReq = {
        lat:this.polyLinePoint.points[parseInt(this.polyLinePoint.points.length/2)].lat,
        lng:this.polyLinePoint.points[parseInt(this.polyLinePoint.points.length/2)].lng
      }
      console.log(geocodeReq);
      this.geoCode(geocodeReq)
      this.loader.load().then(() => {
        this.map = new window.google.maps.Map(document.getElementById("map"), {
          mapId: "5216cd334b4588dc",
          center: { lat: -34.397, lng: 150.644 },
          zoom: 18,
          mapTypeControl: false,
          fullscreenControl: false,
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
          if (this.centerLanLng) {
            this.map.fitBounds(this.centerLanLng);
          } else {
            this.map.setCenter(pos);
          }
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
      this.initAutoComplete();
    },

    initAutoComplete() {
      let map = this.map;
      //const input = document.getElementById("pac_input");
      const container = document.createElement("div");
      container.style.padding = "8px 12px";
      container.style.margin = "10px";
      container.style.borderRadius = "8px";
      container.style.backgroundColor = "white";
      container.style.boxShadow = "rgb(27 142 236 / 50%) 0px 2px 6px 0px";
      container.style.display = "flex";
      container.style.flexDirection = "row";
      container.style.justifyContent = "flex-start";
      container.style.alignItems = "center";
      container.appendChild(document.createTextNode("地點搜索："));
      const input = document.createElement("input");
      container.appendChild(input);
      input.setAttribute("id", "pac_input");
      input.type = "text";
      input.placeholder = "請輸入地點";

      input.style.padding = "8px 12px";
      input.style.border = "1px  solid rgb(200,200,200)";
      input.style.borderRadius = "4px";
      input.style.fontSize = "16px";
      //input.style.boxShadow = "inset 0 0 2px #808080";

      const searchBox = new window.google.maps.places.SearchBox(input);
      this.map.controls[window.google.maps.ControlPosition.TOP_RIGHT].push(
        container
      );
      map.addListener("bounds_changed", () => {
        searchBox.setBounds(map.getBounds());
      });

      // Listen for the event fired when the user selects a prediction and retrieve
      // more details for that place.
      searchBox.addListener("places_changed", () => {
        const places = searchBox.getPlaces();
        if (places.length == 0) {
          return;
        }
        // For each place, get the icon, name and location.
        const bounds = new window.google.maps.LatLngBounds();

        places.forEach((place) => {
          if (!place.geometry || !place.geometry.location) {
            console.log("Returned place contains no geometry");
            return;
          }
          if (place.geometry.viewport) {
            // Only geocodes have viewport.
            bounds.union(place.geometry.viewport);
          } else {
            bounds.extend(place.geometry.location);
            console.log(place.geometry.location);
          }
        });
        this.centerLanLng = bounds;
        map.fitBounds(bounds);
        input.value = "";
      });
    },
    tipEvent() {
      if (document.getElementById("tip_content")) {
        document.getElementById("tip_content").remove();
      }

      let div = document.createElement("div");
      let mapDiv = document.getElementById("map");
      div.setAttribute("id", "tip_content");
      let newContent = document.createTextNode(this.tipContent);
      div.appendChild(newContent);
      div.style.backgroundColor = "white";
      div.style.borderRadius = "4px";
      div.style.padding = "4px 8px";
      div.style.boxShadow = "2px 2px 4px #808080";
      div.style.position = "fixed";
      div.style.zIndex = "9999";
      mapDiv.appendChild(div);
      div.style.display = "none";

      //click event
      this.map.addListener("mousemove", function (e) {
        div.style.display = "block";
        div.style.top = `${e.domEvent.pageY - 30}px`;
        div.style.left = `${e.domEvent.pageX + 5}px`;
      });
      this.map.addListener("mouseout", function () {
        console.log("mouseout");
        div.style.display = "none";
      });
    },
    siteBox() {
      let siteBox = document.createElement("div");
      siteBox.style.cssText = `
      margin : 10px;
      background-color : #fff;
      display : flex;
      flex-direction : row;
      justify-content : center;
      align-items : center;
      box-shadow : rgb(27 142 236 / 50%) 0px 2px 6px 0px;
      padding : 4px 12px;
      border-radius : 4px;
      `;
      /*
      siteBox.style.margin = "10px";
      siteBox.style.backgroundColor = "#fff";
      siteBox.style.display = "flex";
      siteBox.style.flexDirection = "row";
      siteBox.style.justifyContent = "center";
      siteBox.style.alignItems = "center";
      siteBox.style.boxShadow = "rgb(27 142 236 / 50%) 0px 2px 6px 0px";
      siteBox.style.padding = "4px 12px";
      siteBox.style.borderRadius = "4px";
      */
      this.siteTypeList.forEach((event) => {
        let item = document.createElement("div");
        item.onclick = function () {
          console.log(event.name + " on click!");
        };
        item.oncontextmenu = function (e) {
          e.preventDefault();
          console.log(event.name + "on right click");
        };

        item.style.cursor = "pointer";
        item.style.display = "flex";
        item.style.flexDirection = "row";
        item.style.justifyContent = "center";
        item.style.alignItems = "center";
        item.style.padding = "4px 8px";
        let icon = document.createElement("img");
        icon.src = event.icon;
        icon.style.height = "16px";
        icon.style.width = "16px";
        let text = document.createElement("span");
        text.appendChild(document.createTextNode(event.name));
        text.style.marginLeft = "4px";
        item.appendChild(icon);
        item.appendChild(text);
        siteBox.appendChild(item);
      });
      let newSite = document.createElement("div");
      newSite.appendChild(document.createTextNode("新增"));
      newSite.onclick = function () {
        console.log("newSite on click!");
      };
      newSite.style.cursor = "pointer";
      newSite.style.color = "#1890FF";
      newSite.style.marginLeft = "12px";
      newSite.style.padding = "4px 8px";
      siteBox.appendChild(newSite);
      this.map.controls[window.google.maps.ControlPosition.LEFT_TOP].push(
        siteBox
      );
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
  max-width: 1440px;
  width: 80%;
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
