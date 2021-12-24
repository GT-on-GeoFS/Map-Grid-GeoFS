// ==UserScript==
// @name Map Grid
// @description Coordinates Grid for Map
// @author TKE587
// @namespace GeoFS-Plugins
// @match http://*/geofs.php*
// @match https://*/geofs.php*
// @run-at document-end
// @version 0.1.2
// @grant none
// ==/UserScript==
// Crédits: jieter/Leaflet.Grid - https://github.com/jieter/Leaflet.Grid
// Crédits: ardhi/Leaflet.MousePosition https://github.com/ardhi/Leaflet.MousePosition
// Crédits: okainov/js-coordinates https://github.com/okainov/js-coordinates/blob/master/coords.js

(function() {
    'use strict';
    function parseDD(line) {
  const re = /^\D*?(-?[0-9]+(?:[.,][0-9]{1,20})?)[,\s]+(-?[0-9]+(?:[.,][0-9]{1,20})?)\D*$/;
  const match = re.exec(line.trim());
  if (!match) {
    return false;
  }
  var lat = parseFloat(match[1].replace(',', '.'));
  var lon = parseFloat(match[2].replace(',', '.'));
  if (Math.abs(lat) > 90 || Math.abs(lon) > 180) {
    return false;
  }
  return {'lat': lat, 'lon': lon};
}

function parseWSG84(line) {
  const re = /^\D*([NS])\s*(\d{1,2}).\s*(\d{1,2}(?:[.,]\d{1,4})?)'?[,\s/\\]+([EW])\s*(\d{1,3}).\s*(\d{1,2}(?:[.,]\d{1,4})?)'?\D*$/;
  const match = re.exec(line.trim());
  if (!match) {
    return false;
  }
  const latDeg = parseInt(match[2], 10);
  const lonDeg = parseInt(match[5], 10);
  if (latDeg > 90 || lonDeg > 180) {
    return false;
  }

  const latMin = parseFloat(match[3].replace(',', '.'));
  const lonMin = parseFloat(match[6].replace(',', '.'));

  return {
    'lat': match[1], 'lat_deg': latDeg, 'lat_min': latMin,
    'lon': match[4], 'lon_deg': lonDeg, 'lon_min': lonMin
  };
}

function parseWSG84_SimBrief(line) {
  const re = /^\D*([NS])\s*(\d{1,2})\s*(\d{1,2}(?:[.,]\d{1,4})?)'?[,\s/\\]+([EW])\s*(\d{1,3})\s*(\d{1,2}(?:[.,]\d{1,4})?)'?\D*$/;
  const match = re.exec(line.trim());
  if (!match) {
    return false;
  }
  console.log(match);
  const latDeg = parseInt(match[2], 10);
  const lonDeg = parseInt(match[5], 10);
  if (latDeg > 90 || lonDeg > 180) {
    return false;
  }

  const latMin = parseFloat(match[3].replace(',', '.'));
  const lonMin = parseFloat(match[6].replace(',', '.'));

  return {
    'lat': match[1], 'lat_deg': latDeg, 'lat_min': latMin,
    'lon': match[4], 'lon_deg': lonDeg, 'lon_min': lonMin
  };
}

function checkGlue(glue) {
  var newGlue = ', ';
  if (typeof glue === 'undefined') {
    newGlue = ', ';
  } else {
    newGlue = glue;
  }
  return newGlue;
}

/**
 * @returns {string}
 * @param {string} lat
 * @param {number} latDeg
 * @param {number} latMin
 * @param {string} lon
 * @param {number} lonDeg
 * @param {number} lonMin
 * @param {string} _glue
 */
function WGS84toDD(lat, latDeg, latMin, lon, lonDeg, lonMin, _glue) {
  var glue = checkGlue(_glue);

  var la = latDeg + (latMin / 60);
  var lo = lonDeg + (lonMin / 60);
  if (lat === 'S') la = -la;
  if (lon === 'W') lo = -lo;

  return la.toFixed(5) + glue + lo.toFixed(5);
}

/**
 * @returns {string}
 * @param {number} _lat
 * @param {number} _lon
 * @param {string} _glue symbols used to glue lat and lon together in result string
 */
function DDtoWGS84(_lat, _lon, _glue) {
  var glue = checkGlue(_glue);

  var latLetter = _lat >= 0 ? 'N' : 'S';
  var lotLetter = _lon >= 0 ? 'E' : 'W';

  var lat = Math.abs(_lat);
  var lon = Math.abs(_lon);

  var latDeg = Math.floor(lat);
  var lonDeg = Math.floor(lon);

  var latMin = (lat - latDeg) * 60;
  var lonMin = (lon - lonDeg) * 60;

  // \xB0 is °
  return latLetter + ' ' + latDeg + '\xB0 ' + latMin.toFixed(3) + "'" + glue +
    lotLetter + ' ' + lonDeg + '\xB0 ' + lonMin.toFixed(3) + "'";
}

function transformCoordinatesString(line, glue) {
  var coordsFrom = line.trim();
  var coordsTo = coordsFrom;

  var res = parseWSG84(coordsFrom);
  if (res) {
    coordsTo = WGS84toDD(res.lat, res.lat_deg, res.lat_min,
      res.lon, res.lon_deg, res.lon_min, glue);
  } else if (parseDD(coordsFrom)) {
    res = parseDD(coordsFrom);
    coordsTo = DDtoWGS84(res.lat, res.lon, glue);
  }
  return coordsTo;
}




    var polyline, routeLayer;
    var routeOk = false;
    var routeBtn = document.createElement("button");
    routeBtn.setAttribute("class", "mdl-button mdl-js-button geofs-f-standard-ui");
    routeBtn.setAttribute("id", "TKE587toggle-route");
    routeBtn.innerHTML = "Route";
    document.getElementsByClassName("geofs-ui-bottom")[0].appendChild(routeBtn);

    var rteBtn = document.getElementById("TKE587toggle-route");

    rteBtn.addEventListener("click", function(e){
       if(typeof(ui.mapInstance) == 'undefined'){
        alert("Open Nav Panel First to use the Route Functionality");
       }
       else{


           var rte = null;
           var fRte = [];
           var tempRte=[];
           var cLine = null;
           var parsedLine=null;
           var ddFormatedCoords = null;
           var formatedRoute = [];
           var tCirc = [];
           routeOk=true;
           rte = prompt("Enter your route:");
           if(rte != null){
               rte = rte.trim();
               if(rte =="" || rte.toLowerCase() == "remove"){
                   if(rte.toLowerCase() == "remove" && typeof(routeLayer)!=="undefined"){
                     routeLayer.remove();
                   }
                   else{
                     rte= undefined;
                     routeOk=false;
                     alert('Empty route!');
                   }
               }
               else{
                   rte = rte.replace(/\[/gi, '');
                   rte = rte.replace(/\]/gi, '');
                   rte = rte.replace(/ /gi, '');
                   var tRte = rte.split(",");
                   //console.log(tRte);
                   if(0 !== tRte.length % 2){
                       routeOk=false;
                       alert("Invalid Route!! Check Waypoints Coordinates");
                   }
                   else{
                       for(var i=0;i<tRte.length;i++){
                           tempRte.push(tRte[i]);
                           if(i!==0 && 0 !== i % 2){
                               fRte.push(tempRte);
                               tempRte=[];
                           }
                       }
                       //console.log(fRte);
                       for(i=0;i<fRte.length;i++){
                           cLine = fRte[i][0] + ' ' + fRte[i][1];
                           parsedLine = parseWSG84_SimBrief(cLine);
                           if(!parsedLine){
                               let j=i+1;
                               alert("Invalid Waypoint (Check wpt number #" + j + ")");
                               formatedRoute = [];
                               routeOk=false;
                               break;
                           }
                           //console.log(parsedLine);
                           ddFormatedCoords = WGS84toDD(parsedLine.lat, parsedLine.lat_deg, parsedLine.lat_min, parsedLine.lon, parsedLine.lon_deg, parsedLine.lon_min, "|");
                           formatedRoute.push(ddFormatedCoords.split("|"));
                           //console.log(ddFormatedCoords);
                       }
                       //console.log(formatedRoute);
                   }
               }
               //console.log(routeOk);
               if(routeOk){
                 if(typeof(polyline)!=="undefined"){
                    routeLayer.remove();
                 }
                 for(i=0;i<formatedRoute.length;i++){
                   tCirc.push(L.circle(formatedRoute[i], {radius: 1000, fill: true, fillColor: 'blue', fillOpacity: 1}));
                 }
                 //console.log(tCirc);
                 polyline = L.polyline(formatedRoute, {color: 'purple', weight:6});
                 routeLayer = L.layerGroup(tCirc).addLayer(polyline).addTo(ui.mapInstance.apiMap.map);
               }
           }
       }

    });

    var els = document.getElementsByClassName("mdl-button");
    [].forEach.call(els, function (els) {
      if(els.getAttribute("data-toggle-panel") && els.getAttribute("data-toggle-panel")==".geofs-map-list"){
          els.addEventListener("click",function(e){
            L.Grid = L.LayerGroup.extend({
	options: {
		xticks: 8,
		yticks: 5,

		// 'decimal' or one of the templates below
		coordStyle: 'MinDec',
		coordTemplates: {
			'MinDec': '{degAbs}&deg;&nbsp;{minDec}\'{dir}',
			'DMS': '{degAbs}{dir}{min}\'{sec}"'
		},

		// Path style for the grid lines
		lineStyle: {
			stroke: true,
			color: '#111',
			opacity: 0.6,
			weight: 1
		},

		// Redraw on move or moveend
		redraw: 'move'
	},

	initialize: function (options) {
		L.LayerGroup.prototype.initialize.call(this);
		L.Util.setOptions(this, options);

	},

	onAdd: function (map) {
		this._map = map;

		var grid = this.redraw();
		this._map.on('viewreset '+ this.options.redraw, function () {
			grid.redraw();
		});

		this.eachLayer(map.addLayer, map);
	},

	onRemove: function (map) {
		// remove layer listeners and elements
		map.off('viewreset '+ this.options.redraw, this.map);
		this.eachLayer(this.removeLayer, this);
	},

	redraw: function () {
		// pad the bounds to make sure we draw the lines a little longer
		this._bounds = this._map.getBounds().pad(0.5);

		var grid = [];
		var i;

		var latLines = this._latLines();
		for (i in latLines) {
			if (Math.abs(latLines[i]) > 90) {
				continue;
			}
			grid.push(this._horizontalLine(latLines[i]));
			grid.push(this._label('lat', latLines[i]));
		}

		var lngLines = this._lngLines();
		for (i in lngLines) {
			grid.push(this._verticalLine(lngLines[i]));
			grid.push(this._label('lng', lngLines[i]));
		}

		this.eachLayer(this.removeLayer, this);

		for (i in grid) {
			this.addLayer(grid[i]);
		}
		return this;
	},

	_latLines: function () {
		return this._lines(
			this._bounds.getSouth(),
			this._bounds.getNorth(),
			this.options.yticks * 2,
			this._containsEquator()
		);
	},
	_lngLines: function () {
		return this._lines(
			this._bounds.getWest(),
			this._bounds.getEast(),
			this.options.xticks * 2,
			this._containsIRM()
		);
	},

	_lines: function (low, high, ticks, containsZero) {
		var delta = low - high,
			tick = this._round(delta / ticks, delta);

		if (containsZero) {
			low = Math.floor(low / tick) * tick;
		} else {
			low = this._snap(low, tick);
		}

		var lines = [];
		for (var i = -1; i <= ticks; i++) {
			lines.push(low - (i * tick));
		}
		return lines;
	},

	_containsEquator: function () {
		var bounds = this._map.getBounds();
		return bounds.getSouth() < 0 && bounds.getNorth() > 0;
	},

	_containsIRM: function () {
		var bounds = this._map.getBounds();
		return bounds.getWest() < 0 && bounds.getEast() > 0;
	},

	_verticalLine: function (lng) {
		return new L.Polyline([
			[this._bounds.getNorth(), lng],
			[this._bounds.getSouth(), lng]
		], this.options.lineStyle);
	},
	_horizontalLine: function (lat) {
		return new L.Polyline([
			[lat, this._bounds.getWest()],
			[lat, this._bounds.getEast()]
		], this.options.lineStyle);
	},

	_snap: function (num, gridSize) {
		return Math.floor(num / gridSize) * gridSize;
	},

	_round: function (num, delta) {
		var ret;

		delta = Math.abs(delta);
		if (delta >= 1) {
			if (Math.abs(num) > 1) {
				ret = Math.round(num);
			} else {
				ret = (num < 0) ? Math.floor(num) : Math.ceil(num);
			}
		} else {
			var dms = this._dec2dms(delta);
			if (dms.min >= 1) {
				ret = Math.ceil(dms.min) * 60;
			} else {
				ret = Math.ceil(dms.minDec * 60);
			}
		}

		return ret;
	},

	_label: function (axis, num) {
		var latlng;
		var bounds = this._map.getBounds().pad(-0.005);

		if (axis == 'lng') {
			latlng = L.latLng(bounds.getNorth(), num);
		} else {
			latlng = L.latLng(num, bounds.getWest());
		}

		return L.marker(latlng, {
			icon: L.divIcon({
				iconSize: [0, 0],
				className: 'leaflet-grid-label',
				html: '<div class="' + axis + '">' + this.formatCoord(num, axis) + '</div>'
			})
		});
	},

	_dec2dms: function (num) {
		var deg = Math.floor(num);
		var min = ((num - deg) * 60);
		var sec = Math.floor((min - Math.floor(min)) * 60);
		return {
			deg: deg,
			degAbs: Math.abs(deg),
			min: Math.floor(min),
			minDec: min,
			sec: sec
		};
	},

	formatCoord: function (num, axis, style) {
		if (!style) {
			style = this.options.coordStyle;
		}
		if (style == 'decimal') {
			var digits;
			if (num >= 10) {
				digits = 2;
			} else if (num >= 1) {
				digits = 3;
			} else {
				digits = 4;
			}
			return num.toFixed(digits);
		} else {
			// Calculate some values to allow flexible templating
			var dms = this._dec2dms(num);

			var dir;
			if (dms.deg === 0) {
				dir = '&nbsp;';
			} else {
				if (axis == 'lat') {
					dir = (dms.deg > 0 ? 'N' : 'S');
				} else {
					dir = (dms.deg > 0 ? 'E' : 'W');
				}
			}

			return L.Util.template(
				this.options.coordTemplates[style],
				L.Util.extend(dms, {
					dir: dir,
					minDec: Math.round(dms.minDec, 2)
				})
			);
		}
	}

});

L.grid = function (options) {
	return new L.Grid(options);
};

var installTrial = setInterval(function(){
    if(typeof(ui.mapInstance) == 'undefined'){
    }
    else{
    clearInterval(installTrial);
    L.grid().addTo(ui.mapInstance.apiMap.map);
    L.grid({
	redraw: 'moveend'
    }).addTo(ui.mapInstance.apiMap.map);

    L.Control.MousePosition = L.Control.extend({
        options: {
            position: 'bottomleft',
            separator: ' | ',
            emptyString: 'Unavailable',
            lngFirst: false,
            numDigits: 5,
            lngFormatter: undefined,
            latFormatter: undefined,
            ddtowsg : true,
            prefix: ""
        },

        onAdd: function (map) {

            if(!document.getElementsByClassName('leaflet-control-mouseposition').length){
                this._container = L.DomUtil.create('div', 'leaflet-control-mouseposition');
                L.DomEvent.disableClickPropagation(this._container);
                map.on('mousemove', this._onMouseMove, this);
                this._container.innerHTML=this.options.emptyString;

            }
            else{
                this._container=document.getElementsByClassName('leaflet-control-mouseposition')[0];
            }
            return this._container;
        },

        onRemove: function (map) {
            map.off('mousemove', this._onMouseMove)
        },

        _onMouseMove: function (e) {
            var value=null;
            if(!this.options.ddtowsg){
                var lng = this.options.lngFormatter ? this.options.lngFormatter(e.latlng.lng) : L.Util.formatNum(e.latlng.lng, this.options.numDigits);
                var lat = this.options.latFormatter ? this.options.latFormatter(e.latlng.lat) : L.Util.formatNum(e.latlng.lat, this.options.numDigits);
                value = this.options.lngFirst ? lng + this.options.separator + lat : lat + this.options.separator + lng;
            }
            else{
                value= DDtoWGS84(e.latlng.lat,e.latlng.lng,' | ');
            }
            var prefixAndValue = this.options.prefix + ' ' + value;
            this._container.innerHTML = prefixAndValue;
        }

    });

    L.Map.mergeOptions({
        positionControl: false
    });

    L.Map.addInitHook(function () {
        if (this.options.positionControl) {
            this.positionControl = new L.Control.MousePosition();
            this.addControl(this.positionControl);
        }
    });

    L.control.mousePosition = function (options) {
        return new L.Control.MousePosition(options);
    };

L.control.mousePosition().addTo(ui.mapInstance.apiMap.map);
}

},2000);

          });
      }
    });
})();
