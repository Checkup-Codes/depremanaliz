<template>
  <div class="app">
    <header>
      <h1>Türkiye Deprem Analizi</h1>
      <div class="filters">
        <div class="date-picker">
          <label>Başlangıç Tarihi:</label>
          <input type="date" v-model="startDate">
        </div>
        <div class="date-picker">
          <label>Bitiş Tarihi:</label>
          <input type="date" v-model="endDate">
        </div>
        <button class="search-button" @click="fetchEarthquakes" :disabled="!selectedArea || isLoading">
          <span v-if="!isLoading">Depremleri Göster</span>
          <span v-else class="loading-text">
            <span class="loading-spinner"></span>
            Veriler Yükleniyor...
          </span>
        </button>
      </div>
      <div class="instructions" v-if="!selectedArea">
        Lütfen harita üzerinde dikdörtgen çizerek bir bölge seçin
      </div>
    </header>
    <div class="content">
      <div class="map-container" :class="{ 'loading-overlay': isLoading }">
        <div id="map" ref="mapContainer"></div>
        <div v-if="isLoading" class="loading-container">
          <div class="loading-spinner large"></div>
          <div class="loading-message">Deprem verileri yükleniyor...</div>
        </div>
      </div>
      <div class="stats-panel">
        <h2>Bölge İstatistikleri</h2>
        <div v-if="selectedArea" class="stats-content">
          <div class="time-period-info">
            {{ getTimePeriodInfo }}
          </div>
          <div class="stat-card">
            <div class="stat-icon">📏</div>
            <div class="stat-details">
              <div class="stat-label">Seçili Alan</div>
              <div class="stat-value">{{ formatArea(calculateAreaInKm2) }} km²</div>
            </div>
          </div>
          <div class="stat-card">
            <div class="stat-icon">📊</div>
            <div class="stat-details">
              <div class="stat-label">Deprem Sayısı</div>
              <div class="stat-value">{{ markers.length }}</div>
              <div class="stat-subtext">{{ getFrequencyAnalysis }}</div>
            </div>
          </div>
          <div class="stat-card">
            <div class="stat-icon">📈</div>
            <div class="stat-details">
              <div class="stat-label">Ortalama Büyüklük</div>
              <div class="stat-value">{{ calculateAverageMagnitude }}</div>
            </div>
          </div>
          <div class="magnitude-distribution-card">
            <h3>
              <span class="card-icon">🎯</span>
              Büyüklük Dağılımı
            </h3>
            <div class="magnitude-bars">
              <div 
                v-for="(count, range) in magnitudeDistribution" 
                :key="range" 
                class="magnitude-bar-item"
              >
                <div class="magnitude-bar-label">{{ range }}</div>
                <div class="magnitude-bar-container">
                  <div 
                    class="magnitude-bar" 
                    :style="{ width: calculateBarWidth(count) }"
                    :class="getMagnitudeClass(range)"
                  ></div>
                </div>
                <div class="magnitude-bar-count">{{ count }}</div>
              </div>
            </div>
          </div>
          <div class="analysis-card" v-if="getSignificantEvents.length">
            <h3>
              <span class="card-icon">⚠️</span>
              Önemli Depremler
            </h3>
            <div class="significant-events">
              <div v-for="event in getSignificantEvents" :key="event.time" class="significant-event">
                <div class="event-magnitude">{{ event.magnitude }}</div>
                <div class="event-details">
                  <div class="event-location">{{ event.place }}</div>
                  <div class="event-time">{{ formatDate(event.time) }}</div>
                </div>
              </div>
            </div>
          </div>
        </div>
        <div v-else class="no-selection">
          <span class="emoji">🎯</span>
          <p>Lütfen haritadan bir bölge seçin</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, onBeforeUnmount, computed } from 'vue'
import axios from 'axios'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'
import 'leaflet-draw'
import 'leaflet-draw/dist/leaflet.draw.css'

// Leaflet ikon sorunu için gerekli düzeltme
delete L.Icon.Default.prototype._getIconUrl
L.Icon.Default.mergeOptions({
  iconRetinaUrl: require('leaflet/dist/images/marker-icon-2x.png'),
  iconUrl: require('leaflet/dist/images/marker-icon.png'),
  shadowUrl: require('leaflet/dist/images/marker-shadow.png'),
})

export default {
  name: 'App',
  setup() {
    const mapContainer = ref(null)
    const map = ref(null)
    const markers = ref([])
    const selectedArea = ref(null)
    const drawnItems = ref(null)
    const drawControl = ref(null)
    const isLoading = ref(false)
    const startDate = ref(new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString().split('T')[0])
    const endDate = ref(new Date().toISOString().split('T')[0])

    const initMap = () => {
      map.value = L.map(mapContainer.value, {
        zoomControl: true,
        drawControl: false,
        zoomAnimation: true,
        preferCanvas: true
      }).setView([39.0, 35.0], 6)

      L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '© OpenStreetMap contributors',
        zIndex: 1
      }).addTo(map.value)

      // Initialize the FeatureGroup for drawn items
      drawnItems.value = new L.FeatureGroup()
      map.value.addLayer(drawnItems.value)

      // Sadece dikdörtgen çizim aracı
      drawControl.value = new L.Control.Draw({
        position: 'topleft',
        draw: {
          polygon: false,
          circle: false,
          circlemarker: false,
          marker: false,
          polyline: false,
          rectangle: {
            shapeOptions: {
              color: '#3388ff',
              weight: 2,
              zIndex: 100
            }
          }
        },
        edit: false
      })
      map.value.addControl(drawControl.value)

      // Handle draw events
      const handleDrawCreated = (event) => {
        drawnItems.value.clearLayers()
        const layer = event.layer
        drawnItems.value.addLayer(layer)
        selectedArea.value = layer.getBounds()
      }

      map.value.on(L.Draw.Event.CREATED, handleDrawCreated)

      // Store event cleanup functions
      map.value.drawCleanup = () => {
        map.value.off(L.Draw.Event.CREATED, handleDrawCreated)
      }

      // Add zoom change handler
      map.value.on('zoomend', updateMarkerSizes)
    }

    const updateMarkerSizes = () => {
      if (!map.value) return
      
      const zoom = map.value.getZoom()
      markers.value = markers.value.filter(marker => {
        // Check if marker is still on the map
        if (marker && marker.getLatLng) {
          try {
            marker.setRadius(getRadiusByMagnitude(marker.magnitude, zoom))
            return true
          } catch (error) {
            // If marker is invalid or removed, filter it out
            return false
          }
        }
        return false
      })
    }

    const getRadiusByMagnitude = (magnitude, zoom) => {
      if (!zoom) return 0
      // Base radius is smaller and depends on zoom level
      const baseRadius = Math.max(500, 2000 / Math.pow(2, zoom - 6))
      return baseRadius * Math.pow(1.8, magnitude - 2.5)
    }

    const fetchEarthquakes = async () => {
      if (!selectedArea.value) return

      try {
        isLoading.value = true
        // Clear existing markers and references
        markers.value.forEach(marker => {
          if (marker && marker.remove) {
            marker.remove()
          }
        })
        markers.value = []

        const bounds = selectedArea.value
        const response = await axios.get(`https://earthquake.usgs.gov/fdsnws/event/1/query`, {
          params: {
            format: 'geojson',
            starttime: startDate.value,
            endtime: endDate.value,
            minlatitude: bounds.getSouth(),
            maxlatitude: bounds.getNorth(),
            minlongitude: bounds.getWest(),
            maxlongitude: bounds.getEast(),
            minmagnitude: 2.5
          }
        })

        if (!map.value) return // Check if map still exists

        const zoom = map.value.getZoom()
        response.data.features.forEach(quake => {
          const { coordinates } = quake.geometry
          const { mag, place, time } = quake.properties
          
          const marker = L.circle([coordinates[1], coordinates[0]], {
            color: getColorByMagnitude(mag),
            fillColor: getColorByMagnitude(mag),
            fillOpacity: 0.5,
            radius: getRadiusByMagnitude(mag, zoom)
          })
          
          marker.magnitude = mag
          marker.time = time // Zamanı marker'a kaydet
          
          const formattedDate = new Date(time).toLocaleString('tr-TR', {
            year: 'numeric',
            month: 'long',
            day: 'numeric',
            hour: '2-digit',
            minute: '2-digit',
            hour12: false
          })
          
          const popupContent = `
            <div class="popup-content">
              <div class="popup-row">
                <span class="popup-label">Büyüklük:</span>
                <span class="popup-value">${mag.toFixed(1)}</span>
              </div>
              <div class="popup-row">
                <span class="popup-label">Konum:</span>
                <span class="popup-value">${place || 'Bilinmiyor'}</span>
              </div>
              <div class="popup-row">
                <span class="popup-label">Tarih:</span>
                <span class="popup-value">${formattedDate}</span>
              </div>
            </div>
          `
          
          const popupOptions = {
            className: 'custom-popup'
          }
          
          marker.bindPopup(popupContent, popupOptions)
          
          if (map.value) {
            marker.addTo(map.value)
            markers.value.push(marker)
          }
        })
      } catch (error) {
        console.error('Deprem verileri alınamadı:', error)
      } finally {
        isLoading.value = false
      }
    }

    const getColorByMagnitude = (magnitude) => {
      if (magnitude >= 6) return '#FF0000'
      if (magnitude >= 5) return '#FF6B00'
      if (magnitude >= 4) return '#FFA500'
      if (magnitude >= 3) return '#FFD700'
      return '#90EE90'
    }

    const calculateAreaInKm2 = computed(() => {
      if (!selectedArea.value) return 0
      
      const bounds = selectedArea.value
      const width = calculateDistance(
        bounds.getNorth(),
        bounds.getWest(),
        bounds.getNorth(),
        bounds.getEast()
      )
      const height = calculateDistance(
        bounds.getNorth(),
        bounds.getWest(),
        bounds.getSouth(),
        bounds.getWest()
      )
      return width * height
    })

    const calculateDistance = (lat1, lon1, lat2, lon2) => {
      const R = 6371 // Earth's radius in km
      const dLat = toRad(lat2 - lat1)
      const dLon = toRad(lon2 - lon1)
      const a = Math.sin(dLat/2) * Math.sin(dLat/2) +
                Math.cos(toRad(lat1)) * Math.cos(toRad(lat2)) *
                Math.sin(dLon/2) * Math.sin(dLon/2)
      const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a))
      return R * c
    }

    const toRad = (value) => {
      return value * Math.PI / 180
    }

    const formatArea = (area) => {
      return Math.round(area).toLocaleString('tr-TR')
    }

    const calculateAverageMagnitude = computed(() => {
      if (!markers.value.length) return '0.0'
      const sum = markers.value.reduce((acc, marker) => acc + marker.magnitude, 0)
      return (sum / markers.value.length).toFixed(1)
    })

    const magnitudeDistribution = computed(() => {
      const distribution = {
        '2.5-2.9': 0,
        '3.0-3.9': 0,
        '4.0-4.9': 0,
        '5.0-5.9': 0,
        '6.0+': 0
      }

      markers.value.forEach(marker => {
        const mag = marker.magnitude
        if (mag >= 6.0) distribution['6.0+']++
        else if (mag >= 5.0) distribution['5.0-5.9']++
        else if (mag >= 4.0) distribution['4.0-4.9']++
        else if (mag >= 3.0) distribution['3.0-3.9']++
        else if (mag >= 2.5) distribution['2.5-2.9']++
      })

      return distribution
    })

    const getTimePeriodInfo = computed(() => {
      if (!startDate.value || !endDate.value) return ''
      
      const start = new Date(startDate.value)
      const end = new Date(endDate.value)
      const diffTime = Math.abs(end - start)
      const diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24))
      
      if (diffDays > 365) {
        const years = (diffDays / 365).toFixed(1)
        return `${years} yıllık deprem analizi`
      } else if (diffDays > 30) {
        const months = Math.round(diffDays / 30)
        return `${months} aylık deprem analizi`
      } else {
        return `${diffDays} günlük deprem analizi`
      }
    })

    const getFrequencyAnalysis = computed(() => {
      if (!markers.value.length || !startDate.value || !endDate.value) return ''
      
      const start = new Date(startDate.value)
      const end = new Date(endDate.value)
      const diffTime = Math.abs(end - start)
      const diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24))
      
      const frequency = (markers.value.length / diffDays).toFixed(1)
      return `Günde ortalama ${frequency} deprem`
    })

    const getSignificantEvents = computed(() => {
      if (!markers.value.length) return []
      
      return markers.value
        .filter(marker => marker.magnitude >= 4.0)
        .map(marker => {
          const popup = marker.getPopup()
          const content = popup ? popup.getContent() : ''
          let place = 'Bilinmiyor'
          
          // HTML içeriğini geçici bir div oluşturarak parse edelim
          if (content) {
            const tempDiv = document.createElement('div')
            tempDiv.innerHTML = content
            const placeElement = tempDiv.querySelector('.popup-value:nth-child(2)')
            if (placeElement) {
              place = placeElement.textContent.trim()
            }
          }
          
          return {
            magnitude: marker.magnitude.toFixed(1),
            place: place,
            time: new Date(marker.time)
          }
        })
        .sort((a, b) => b.magnitude - a.magnitude)
        .slice(0, 5)
    })

    const calculateBarWidth = (count) => {
      const maxCount = Math.max(...Object.values(magnitudeDistribution.value))
      return `${(count / maxCount) * 100}%`
    }

    const getMagnitudeClass = (range) => {
      if (range === '6.0+') return 'magnitude-severe'
      if (range === '5.0-5.9') return 'magnitude-high'
      if (range === '4.0-4.9') return 'magnitude-medium'
      if (range === '3.0-3.9') return 'magnitude-low'
      return 'magnitude-minor'
    }

    const formatDate = (date) => {
      return new Date(date).toLocaleDateString('tr-TR', {
        day: 'numeric',
        month: 'long',
        year: 'numeric',
        hour: '2-digit',
        minute: '2-digit'
      })
    }

    onMounted(() => {
      initMap()
    })

    onBeforeUnmount(() => {
      if (map.value && map.value.drawCleanup) {
        map.value.drawCleanup()
      }
      if (map.value) {
        map.value.remove()
      }
    })

    return {
      mapContainer,
      startDate,
      endDate,
      fetchEarthquakes,
      selectedArea,
      isLoading,
      markers,
      calculateAreaInKm2,
      formatArea,
      calculateAverageMagnitude,
      magnitudeDistribution,
      getTimePeriodInfo,
      getFrequencyAnalysis,
      getSignificantEvents,
      calculateBarWidth,
      getMagnitudeClass,
      formatDate
    }
  }
}
</script>

<style>
.app {
  height: 100vh;
  display: flex;
  flex-direction: column;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
}

header {
  background: linear-gradient(135deg, #2193b0, #6dd5ed);
  color: white;
  padding: 1rem;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
}

h1 {
  margin: 0 0 1rem 0;
  text-align: center;
  font-weight: 600;
}

.filters {
  display: flex;
  justify-content: center;
  gap: 1.5rem;
  align-items: center;
  flex-wrap: wrap;
}

.date-picker {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  background: rgba(255,255,255,0.1);
  padding: 0.5rem 1rem;
  border-radius: 8px;
}

input[type="date"] {
  padding: 0.5rem;
  border: none;
  border-radius: 4px;
  background: white;
  color: #333;
}

.search-button {
  padding: 0.75rem 1.5rem;
  background: #4CAF50;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-weight: 600;
  transition: all 0.3s ease;
}

.search-button:hover:not(:disabled) {
  background: #45a049;
  transform: translateY(-1px);
}

.content {
  flex-grow: 1;
  display: flex;
  overflow: hidden;
  position: relative;
  height: calc(100vh - 120px);
}

.map-container {
  flex: 3;
  position: relative;
  box-shadow: inset -5px 0 10px rgba(0,0,0,0.1);
  height: 100%;
}

#map {
  width: 100%;
  height: 100%;
  z-index: 1;
}

.leaflet-container {
  width: 100%;
  height: 100%;
}

.leaflet-control-container {
  position: absolute;
  z-index: 2;
}

.loading-overlay {
  position: relative;
}

.loading-container {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(255, 255, 255, 0.8);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  z-index: 8000;
}

.loading-spinner {
  display: inline-block;
  width: 20px;
  height: 20px;
  border: 3px solid #f3f3f3;
  border-top: 3px solid #3498db;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin-right: 8px;
  vertical-align: middle;
}

.loading-spinner.large {
  width: 50px;
  height: 50px;
  border-width: 4px;
}

.loading-text {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.loading-overlay .leaflet-control-container {
  pointer-events: none;
  opacity: 0.5;
}

.stats-panel {
  flex: 1;
  background: #f8f9fa;
  padding: 1.5rem;
  overflow-y: auto;
  border-left: 1px solid #dee2e6;
}

.time-period-info {
  text-align: center;
  color: #2193b0;
  font-size: 1.2rem;
  font-weight: 600;
  margin-bottom: 1.5rem;
  padding: 0.5rem;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
}

.stat-card {
  display: flex;
  align-items: center;
  background: white;
  padding: 1.2rem;
  border-radius: 12px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.05);
  margin-bottom: 1rem;
  transition: transform 0.3s ease;
}

.stat-card:hover {
  transform: translateY(-2px);
}

.stat-icon {
  font-size: 2rem;
  margin-right: 1rem;
}

.stat-details {
  flex-grow: 1;
}

.stat-label {
  color: #666;
  font-size: 0.9rem;
  margin-bottom: 0.3rem;
}

.stat-value {
  font-size: 1.4rem;
  font-weight: 600;
  color: #2193b0;
}

.stat-subtext {
  font-size: 0.9rem;
  color: #666;
  margin-top: 0.3rem;
}

.magnitude-distribution-card {
  background: white;
  padding: 1.5rem;
  border-radius: 12px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.05);
  margin-top: 1rem;
}

.magnitude-bars {
  margin-top: 1rem;
}

.magnitude-bar-item {
  display: flex;
  align-items: center;
  margin-bottom: 0.8rem;
  gap: 1rem;
}

.magnitude-bar-label {
  width: 70px;
  font-size: 0.9rem;
  color: #666;
}

.magnitude-bar-container {
  flex-grow: 1;
  height: 12px;
  background: #f0f0f0;
  border-radius: 6px;
  overflow: hidden;
}

.magnitude-bar {
  height: 100%;
  border-radius: 6px;
  transition: width 0.3s ease;
}

.magnitude-bar-count {
  width: 40px;
  text-align: right;
  font-size: 0.9rem;
  color: #666;
}

.magnitude-severe { background: #ff4444; }
.magnitude-high { background: #ff8800; }
.magnitude-medium { background: #ffbb33; }
.magnitude-low { background: #00C851; }
.magnitude-minor { background: #33b5e5; }

.card-icon {
  margin-right: 0.5rem;
}

.analysis-card {
  background: white;
  padding: 1.5rem;
  border-radius: 12px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.05);
  margin-top: 1rem;
}

.significant-events {
  margin-top: 1rem;
}

.significant-event {
  display: flex;
  align-items: center;
  padding: 0.8rem;
  border-bottom: 1px solid #eee;
}

.event-magnitude {
  font-size: 1.2rem;
  font-weight: 600;
  color: #ff4444;
  margin-right: 1rem;
  min-width: 50px;
}

.event-details {
  flex-grow: 1;
}

.event-location {
  font-weight: 500;
  margin-bottom: 0.2rem;
}

.event-time {
  font-size: 0.9rem;
  color: #666;
}

.no-selection {
  text-align: center;
  padding: 2rem;
}

.no-selection .emoji {
  font-size: 3rem;
  margin-bottom: 1rem;
  display: block;
}

/* Leaflet kontrol düzeltmeleri */
.leaflet-top,
.leaflet-bottom {
  z-index: 9999 !important;
}

.leaflet-draw.leaflet-control {
  z-index: 9999 !important;
}

.leaflet-draw-toolbar {
  z-index: 9999 !important;
}

.leaflet-draw-actions {
  z-index: 10000 !important;
}

.leaflet-control-zoom {
  z-index: 9999 !important;
}

.leaflet-control-attribution {
  z-index: 9999 !important;
}

/* Map pane z-index düzeltmesi */
.leaflet-map-pane {
  z-index: 2;
}

.leaflet-tile-pane {
  z-index: 3;
}

.leaflet-overlay-pane {
  z-index: 4;
}

.leaflet-marker-pane {
  z-index: 5;
}

.leaflet-tooltip-pane {
  z-index: 6;
}

.leaflet-popup-pane {
  z-index: 7;
}

/* Loading container z-index düzeltmesi */
.loading-container {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(255, 255, 255, 0.8);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  z-index: 8000;
}

/* Popup stilleri */
.custom-popup .leaflet-popup-content-wrapper {
  background: white;
  border-radius: 8px;
  padding: 0;
  box-shadow: 0 3px 14px rgba(0,0,0,0.2);
}

.custom-popup .leaflet-popup-content {
  margin: 0;
  padding: 12px;
}

.popup-content {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

.popup-row {
  display: flex;
  align-items: flex-start;
  margin-bottom: 8px;
  line-height: 1.4;
}

.popup-row:last-child {
  margin-bottom: 0;
}

.popup-label {
  font-weight: 600;
  color: #666;
  min-width: 80px;
  padding-right: 8px;
}

.popup-value {
  color: #333;
  flex: 1;
}
</style> 