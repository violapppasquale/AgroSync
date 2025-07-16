
<body>
  <h1>AgroSync - Calcolo Piante e Capienza</h1>
  <label>Coordinate (minimo 3, in formato decimale: lat lon)</label>
  <textarea id="coordInput" rows="5" placeholder="38.937222 17.015556\n38.937222 17.016389\n..."></textarea>

  <label>Distanza tra le piante (in metri)</label>
  <input type="number" id="spacing" placeholder="es. 6">

  <label>Distanza dai bordi (in metri)</label>
  <input type="number" id="border" placeholder="es. 2">

  <label>Numero desiderato di piante (facoltativo)</label>
  <input type="number" id="desired" placeholder="es. 100">

  <button onclick="elabora()">Calcola Area di Lavoro e Punti</button>
  <button onclick="trackPosition()">📍 Mostra la mia posizione</button>

  <div id="output"></div>
  <div id="map"></div>

  <script>
    let map = L.map('map').setView([38.936, 17.016], 17);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      maxZoom: 20,
    }).addTo(map);

    let polygonLayer, pointsLayer;
    let userMarker;

    function trackPosition() {
      if (!navigator.geolocation) {
        alert("Geolocalizzazione non supportata dal browser.");
        return;
      }

      navigator.geolocation.watchPosition(
        (pos) => {
          const lat = pos.coords.latitude;
          const lon = pos.coords.longitude;
          if (!userMarker) {
            userMarker = L.circleMarker([lat, lon], {
              radius: 6,
              color: 'red',
              fillOpacity: 0.9
            }).addTo(map);
          } else {
            userMarker.setLatLng([lat, lon]);
          }
        },
        (err) => {
          alert("Errore nella localizzazione: " + err.message);
        },
        {
          enableHighAccuracy: true,
          maximumAge: 0,
          timeout: 10000
        }
      );
    }

    function elabora() {
      const rawCoords = document.getElementById("coordInput").value.trim().split("\n");
      const spacing = parseFloat(document.getElementById("spacing").value);
      const border = parseFloat(document.getElementById("border").value);
      const desiredPlants = parseInt(document.getElementById("desired").value);
      const output = document.getElementById("output");

      if (rawCoords.length < 3 || isNaN(spacing) || isNaN(border)) {
        output.innerHTML = "<p class='red'>Inserisci almeno 3 coordinate valide, distanza tra piante e margine.</p>";
        return;
      }

      let coords = [];
      for (let line of rawCoords) {
        const parts = line.trim().split(/\s+/);
        if (parts.length !== 2) {
          output.innerHTML = "<p class='red'>Formato non valido: usa latitudine e longitudine separate da uno spazio.</p>";
          return;
        }
        const lat = parseFloat(parts[0]);
        const lon = parseFloat(parts[1]);
        if (isNaN(lat) || isNaN(lon)) {
          output.innerHTML = "<p class='red'>Errore nella conversione di una coordinata.</p>";
          return;
        }
        coords.push([lon, lat]); // GeoJSON format
      }

      if (coords[0][0] !== coords[coords.length - 1][0] || coords[0][1] !== coords[coords.length - 1][1]) {
        coords.push(coords[0]);
      }

      const polygon = turf.polygon([coords]);
      const area = turf.area(polygon);
      const areaHa = (area / 10000).toFixed(2);
      const areaM2 = area.toFixed(2);

      if (polygonLayer) map.removeLayer(polygonLayer);
      if (pointsLayer) map.removeLayer(pointsLayer);

      const leafletCoords = coords.map(([lon, lat]) => [lat, lon]);
      polygonLayer = L.polygon(leafletCoords, {color: 'green'}).addTo(map);
      map.fitBounds(polygonLayer.getBounds());

      const centroid = turf.centroid(polygon).geometry.coordinates;
      const lat0 = centroid[1];
      const lon0 = centroid[0];

      function latlonToXY(lat, lon) {
        const R = 6371000;
        const x = R * (lon - lon0) * Math.cos(lat0 * Math.PI / 180) * Math.PI / 180;
        const y = R * (lat - lat0) * Math.PI / 180;
        return [x, y];
      }

      function xyToLatlon(x, y) {
        const R = 6371000;
        const lat = lat0 + (y / R) * 180 / Math.PI;
        const lon = lon0 + (x / (R * Math.cos(lat0 * Math.PI / 180))) * 180 / Math.PI;
        return [lat, lon];
      }

      const localCoords = coords.map(([lon, lat]) => latlonToXY(lat, lon));
      const localPolygon = turf.polygon([localCoords]);

      let [minx, miny, maxx, maxy] = turf.bbox(localPolygon);
      minx += border;
      maxx -= border;
      miny += border;
      maxy -= border;

      const nx = Math.floor((maxx - minx) / spacing);
      const ny = Math.floor((maxy - miny) / spacing);

      const gridPoints = [];
      for (let i = 0; i <= nx; i++) {
        for (let j = 0; j <= ny; j++) {
          const x = minx + i * spacing;
          const y = miny + j * spacing;
          const pt = turf.point([x, y]);
          if (turf.booleanPointInPolygon(pt, localPolygon)) {
            const [lat, lon] = xyToLatlon(x, y);
            gridPoints.push({ lat, lon });
          }
        }
      }

      let maxCapacity = gridPoints.length;
      let usedPoints = gridPoints;
      if (!isNaN(desiredPlants) && desiredPlants < maxCapacity) {
        usedPoints = gridPoints.slice(0, desiredPlants);
      }

      output.innerHTML = `
        <p class='green'>Coordinate accettate ✅</p>
        <p>Totale punti poligono: ${coords.length - 1}</p>
        <p>Distanza tra piante: ${spacing} m<br>
        Distanza dal bordo: ${border} m</p>
        <p><strong>Superficie stimata (geodetica):</strong> ${areaM2} m² → ${areaHa} ettari</p>
        <p><strong>Capienza massima:</strong> ${maxCapacity} piante</p>
        ${!isNaN(desiredPlants) ? `<p><strong>Piante inserite:</strong> ${usedPoints.length}</p>` : ""}
      `;

      pointsLayer = L.layerGroup();
      usedPoints.forEach(({ lat, lon }) => {
        L.circleMarker([lat, lon], { radius: 2, color: 'blue' }).addTo(pointsLayer);
      });
      pointsLayer.addTo(map);
    }
  </script>
</body>

