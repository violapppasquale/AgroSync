<!DOCTYPE html>
<html>
<head>
  <title>AgroSync - Piante e Capienza</title>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <script src="https://cdn.jsdelivr.net/npm/@turf/turf@6/turf.min.js"></script>
  <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
  <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
  <style>
    body { font-family: sans-serif; background: #f0f0f0; padding: 30px; }
    h1 { color: #2c3e50; }
    label { display: block; margin: 10px 0 5px; }
    input, button, textarea {
      padding: 10px;
      width: 100%;
      max-width: 500px;
      font-size: 16px;
      margin-bottom: 10px;
    }
    #output {
      background: #fff;
      padding: 20px;
      margin-top: 20px;
      border-radius: 5px;
      max-width: 600px;
      box-shadow: 0 0 5px rgba(0,0,0,0.1);
    }
    .green { color: green; font-weight: bold; }
    .red { color: red; font-weight: bold; }
    #map { height: 600px; margin-top: 30px; border: 1px solid #ccc; border-radius: 10px; }
  </style>
</head>
<body>
  <h1>AgroSync - Calcolo Piante e Capienza</h1>
  <label>Coordinate (minimo 3, in formato decimale: lat lon)</label>
  <textarea id="coordInput" rows="5" placeholder="38.937222 17.015556
38.937222 17.016389
..."></textarea>

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
    let userMarker = null;

    function trackPosition() {
      if (!navigator.geolocation) {
        alert("Geolocalizzazione non supportata dal browser.");
        return;
      }

      navigator.geolocation.watchPosition(
        (position) => {
          const lat = position.coords.latitude;
          const lon = position.coords.longitude;

          if (!userMarker) {
            userMarker = L.circleMarker([lat, lon], {
              radius: 6,
              color: 'red',
              fillOpacity: 1
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
          timeout: 10000,
          maximumAge: 0
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

      const bbox = turf.bbox(polygon);
      const grid = turf.pointGrid(bbox, spacing, {units: 'meters'});
      const validPoints = turf.pointsWithinPolygon(grid, polygon);

      let maxCapacity = validPoints.features.length;
      let placedPoints = validPoints.features;
      if (!isNaN(desiredPlants) && desiredPlants < maxCapacity) {
        placedPoints = validPoints.features.slice(0, desiredPlants);
      }

      output.innerHTML = `
        <p class='green'>Coordinate accettate ✅</p>
        <p>Totale punti poligono: ${coords.length - 1}</p>
        <p>Distanza tra piante: ${spacing} m<br>
        Distanza dal bordo: ${border} m</p>
        <p><strong>Superficie stimata (geodetica):</strong> ${areaM2} m² → ${areaHa} ettari</p>
        <p><strong>Capienza massima:</strong> ${maxCapacity} piante</p>
        ${!isNaN(desiredPlants) ? `<p><strong>Piante inserite:</strong> ${placedPoints.length}</p>` : ""}
      `;

      pointsLayer = L.layerGroup();
      placedPoints.forEach(pt => {
        const [lon, lat] = pt.geometry.coordinates;
        L.circleMarker([lat, lon], {radius: 2, color: 'blue'}).addTo(pointsLayer);
      });
      pointsLayer.addTo(map);
    }
  </script>
</body>
</html>

