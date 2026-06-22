<template>
  <div class="map-wrapper">
    <div v-if="mostrarPopup" class="modal-overlay">
      <div class="modal-content">
        <div class="modal-icon">📍</div>
        <h2>Ativar Localização em Tempo Real?</h2>
        <p>
          Para ajudar você a navegar pelo campus, gostaríamos de mostrar onde você está no mapa em tempo real.
        </p>
        <div class="modal-actions">
          <button class="btn-negativos" @click="recusarLocalizacao">Agora não</button>
          <button class="btn-positivo" @click="aceitarLocalizacao">Sim, ativar</button>
        </div>
      </div>
    </div>

    <button class="menu-toggle" @click="menuAberto = !menuAberto">
      <span v-if="menuAberto">✕</span>
      <span v-else>☰</span>
    </button>

    <aside class="sidebar" :class="{ open: menuAberto }">
      <h3>Filtros de Marcadores</h3>
      <div class="filter-group">
        <label>
          <input type="checkbox" v-model="filtros.blocos" />
          Blocos (A-L)
        </label>
        <label>
          <input type="checkbox" v-model="filtros.estruturas" />
          Refeitório, Quadra, Academia
        </label>
        <label>
          <input type="checkbox" v-model="filtros.acessibilidade" />
          Escadas, Rampas, Elevadores
        </label>
        <label>
          <input type="checkbox" v-model="filtros.banheiros" />
          Banheiros
        </label>
        <label>
          <input type="checkbox" v-model="filtros.entradas" />
          Portão, Univille, Garten
        </label>
      </div>

      <hr />

      <h3>Localização</h3>
      <button class="location-btn" @click="ativarLocalizacao">
        Minha Localização
      </button>
      <p v-if="localizacaoStatus" class="location-status">
        {{ localizacaoStatus }}
      </p>
    </aside>

    <div id="map"></div>
  </div>
</template>

<script setup>
import { onMounted, ref, watch, reactive, onUnmounted } from "vue";
import L from "leaflet";
import "leaflet/dist/leaflet.css";

const andarAtual = ref(1);
const mapa = ref(null);
const menuAberto = ref(true);
const localizacaoStatus = ref("");
const mostrarPopup = ref(false); // Controle do nosso modal customizado

let camadaImagemOverlay = null;
let marcadoresAtuais = [];
let marcadorUsuario = null;
let watchId = null;

const filtros = reactive({
  blocos: true,
  estruturas: true,
  acessibilidade: true,
  banheiros: true,
  entradas: true,
});

/*====================================================
LIMITES
====================================================*/
const cantoInferiorEsquerdo = [-26.255592, -48.857007];
const cantoSuperiorDireito = [-26.25116, -48.852338];

const limitesUniversidade = L.latLngBounds(
  cantoInferiorEsquerdo,
  cantoSuperiorDireito
);

/*====================================================
PONTOS
====================================================*/
const pontosDeInteresse = [
  { nome: "univille", coordenadas: [-26.251866, -48.856025], categoria: "entradas", tamanho: 70 },
  { nome: "garten", coordenadas: [-26.25383115301317, -48.853101890364904], categoria: "entradas", tamanho: 70 },
  { nome: "portao", coordenadas: [-26.252061, -48.854161], categoria: "entradas", tamanho: 55 },
  { nome: "a", coordenadas: [-26.252243590865646, -48.85449532462595], categoria: "blocos", tamanho: 55 },
  { nome: "b", coordenadas: [-26.252476937034576, -48.85471255925737], categoria: "blocos", tamanho: 55 },
  { nome: "c", coordenadas: [-26.252600188291428, -48.85502835627211], categoria: "blocos", tamanho: 55 },
  { nome: "d", coordenadas: [-26.252804915733297, -48.854714066797264], categoria: "blocos", tamanho: 55 },
  { nome: "e", coordenadas: [-26.253161, -48.854839], categoria: "blocos", tamanho: 55 },
  { nome: "k", coordenadas: [-26.25340301156343, -48.854834815446246], categoria: "blocos", tamanho: 55 },
  { nome: "f", coordenadas: [-26.25362258956487, -48.85489194329959], categoria: "blocos", tamanho: 55 },
  { nome: "g", coordenadas: [-26.25410200011257, -48.85460630403199], categoria: "blocos", tamanho: 55 },
  { nome: "h", coordenadas: [-26.254036127022285, -48.855299999429725], categoria: "blocos", tamanho: 55 },
  { nome: "i", coordenadas: [-26.253036346320055, -48.85532941250589], categoria: "blocos", tamanho: 55 },
  { nome: "j", coordenadas: [-26.253124878831727, -48.85598553368974], categoria: "blocos", tamanho: 55 },
  { nome: "l", coordenadas: [-26.254402088132885, -48.85582639179274], categoria: "blocos", tamanho: 55 },
  { nome: "refeitorio", coordenadas: [-26.253403011562707, -48.85596105032403], categoria: "estruturas", tamanho: 55 },
  { nome: "quadra", coordenadas: [-26.25290164023696, -48.85585495573923], categoria: "estruturas", tamanho: 55 },
  { nome: "academia", coordenadas: [-26.252535674399454, -48.8557611028373], categoria: "estruturas", tamanho: 55 },
  { nome: "rampa", coordenadas: [-26.253508, -48.854782], categoria: "acessibilidade", tamanho: 55 },
  { nome: "elevador", coordenadas: [-26.253564, -48.854632], categoria: "acessibilidade", tamanho: 55 },
  { nome: "banheiro", coordenadas: [-26.253597, -48.855198], category: "banheiros", tamanho: 55 },
  { nome: "escada", coordenadas: [-26.25355066995676, -48.85527063674971], categoria: "acessibilidade", tamanho: 55 },
];

/*====================================================
MAPA
====================================================*/
onMounted(() => {
  mapa.value = L.map("map", {
    maxBounds: limitesUniversidade,
    maxBoundsViscosity: 1,
    maxZoom: 22,
    zoomControl: true,
    zoomSnap: 0.5,
    zoomDelta: 0.5,
  });

  configurarLimitesEZoom();
  atualizarImagemEAndar();
  atualizarMarcadores();

  // 🔥 Abre o nosso popup customizado assim que entra no site
  mostrarPopup.value = true;

  window.addEventListener("resize", configurarLimitesEZoom);
});

onUnmounted(() => {
  if (watchId !== null) {
    navigator.geolocation.clearWatch(watchId);
  }
  window.removeEventListener("resize", configurarLimitesEZoom);
});

/*====================================================
ZOOM
====================================================*/
const configurarLimitesEZoom = () => {
  mapa.value.fitBounds(limitesUniversidade);
  const zoom = mapa.value.getBoundsZoom(limitesUniversidade, true);
  mapa.value.setMinZoom(zoom);
};

/*====================================================
MARCADORES
====================================================*/
const limparMarcadores = () => {
  marcadoresAtuais.forEach((m) => mapa.value.removeLayer(m));
  marcadoresAtuais = [];
};

const atualizarMarcadores = () => {
  limparMarcadores();

  pontosDeInteresse.forEach((ponto) => {
    if (!filtros[ponto.categoria]) return;

    const tamanho = ponto.tamanho || 55;

    const icone = L.icon({
      iconUrl: `/${ponto.nome}.png`,
      iconSize: [tamanho, tamanho],
      iconAnchor: [tamanho / 2, tamanho],
      popupAnchor: [0, -tamanho + 5],
      className: "marker-icon",
    });

    const marcador = L.marker(ponto.coordenadas, { icon: icone })
      .addTo(mapa.value)
      .bindPopup(`
        <div style="text-align:center;font-family:sans-serif;">
          <strong>${ponto.nome.toUpperCase()}</strong>
        </div>
      `);

    marcadoresAtuais.push(marcador);
  });
};

watch(filtros, atualizarMarcadores, { deep: true });

/*====================================================
TROCAR IMAGEM DO MAPA
====================================================*/
const atualizarImagemEAndar = () => {
  if (camadaImagemOverlay) {
    mapa.value.removeLayer(camadaImagemOverlay);
  }

  let url = "/mapaOficial.png";

  camadaImagemOverlay = L.imageOverlay(url, limitesUniversidade, {
    opacity: 1,
    interactive: false
  }).addTo(mapa.value);

  camadaImagemOverlay.bringToFront();
};

/*====================================================
AÇÕES DO POPUP CUSTOMIZADO
====================================================*/
const aceitarLocalizacao = () => {
  mostrarPopup.value = false;
  ativarLocalizacao(); // Dispara o rastreamento real
};

const recusarLocalizacao = () => {
  mostrarPopup.value = false;
  localizacaoStatus.value = "Localização não ativada.";
};

/*====================================================
LOCALIZAÇÃO EM TEMPO REAL
====================================================*/
const ativarLocalizacao = () => {
  if (!navigator.geolocation) {
    localizacaoStatus.value = "Geolocalização não suportada.";
    return;
  }

  localizacaoStatus.value = "Obtendo localização...";

  if (marcadorUsuario) {
    mapa.value.removeLayer(marcadorUsuario);
  }

  if (watchId !== null) {
    navigator.geolocation.clearWatch(watchId);
  }

  const iconeUsuario = L.divIcon({
    className: "user-location-marker",
    html: '<div class="pulse"></div><div class="dot"></div>',
    iconSize: [40, 40],
    iconAnchor: [20, 20],
  });

  watchId = navigator.geolocation.watchPosition(
    (position) => {
      const lat = position.coords.latitude;
      const lng = position.coords.longitude;

      localizacaoStatus.value = `Lat: ${lat.toFixed(6)}, Lng: ${lng.toFixed(6)}`;

      if (marcadorUsuario) {
        marcadorUsuario.setLatLng([lat, lng]);
      } else {
        marcadorUsuario = L.marker([lat, lng], { icon: iconeUsuario })
          .addTo(mapa.value)
          .bindPopup("Você está aqui!");
      }

      if (!marcadorUsuario._centrado) {
        mapa.value.setView([lat, lng], 19);
        marcadorUsuario._centrado = true;
      }
    },
    (error) => {
      switch (error.code) {
        case error.PERMISSION_DENIED:
          localizacaoStatus.value = "Permissão negada pelo navegador.";
          break;
        default:
          localizacaoStatus.value = "Erro ao obter localização.";
      }
    },
    {
      enableHighAccuracy: true,
      timeout: 10000,
      maximumAge: 0,
    }
  );
};
</script>

<style scoped>
* {
  box-sizing: border-box;
}

.map-wrapper {
  width: 100vw;
  height: 100vh;
  position: fixed;
  top: 0;
  left: 0;
  overflow: hidden;
}

#map {
  width: 100%;
  height: 100%;
  position: absolute;
  top: 0;
  left: 0;
}

/*====================================================
ESTILOS DO POPUP MODAL (NOVO)
====================================================*/
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.5);
  backdrop-filter: blur(5px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000; /* Fica acima de tudo, inclusive do menu */
}

.modal-content {
  background: white;
  padding: 30px;
  border-radius: 20px;
  width: 90%;
  max-width: 400px;
  text-align: center;
  box-shadow: 0 15px 40px rgba(0,0,0,0.3);
  font-family: sans-serif;
}

.modal-icon {
  font-size: 40px;
  margin-bottom: 15px;
}

.modal-content h2 {
  margin: 0 0 12px 0;
  color: #2c3e50;
  font-size: 20px;
}

.modal-content p {
  color: #666;
  font-size: 14px;
  line-height: 1.5;
  margin: 0 0 24px 0;
}

.modal-actions {
  display: flex;
  gap: 12px;
}

.modal-actions button {
  flex: 1;
  padding: 12px;
  border: none;
  border-radius: 10px;
  font-weight: bold;
  font-size: 14px;
  cursor: pointer;
  transition: background 0.2s;
}

.btn-positivo {
  background: #3498db;
  color: white;
}

.btn-positivo:hover {
  background: #2980b9;
}

.btn-negativos {
  background: #eee;
  color: #555;
}

.btn-negativos:hover {
  background: #ddd;
}

/*====================================================
RESTA DOS COMPONENTES
====================================================*/
.menu-toggle {
  position: absolute;
  top: 15px;
  right: 20px;
  left: auto;
  z-index: 1100;
  width: 48px;
  height: 48px;
  border: none;
  border-radius: 14px;
  background: white;
  color: #555;
  font-size: 20px;
  cursor: pointer;
  box-shadow: 0 8px 20px rgba(0,0,0,.15);
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.2s;
}

.menu-toggle:hover {
  background: #f5f5f5;
}

.sidebar {
  position: absolute;
  top: 20px;
  right: -340px;
  width: 290px;
  max-height: calc(100% - 40px);
  border-radius: 22px;
  background: rgba(255, 255, 255, .96);
  backdrop-filter: blur(18px);
  box-shadow: 0 12px 35px rgba(0,0,0,.18);
  padding: 22px;
  transition: .35s;
  z-index: 1050; 
}

.sidebar.open {
  right: 20px;
}

.sidebar h3 {
  margin: 0 0 10px 0;
  font-size: 14px;
  color: #2c3e50;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.sidebar hr {
  border: none;
  border-top: 1px solid #eee;
  margin: 20px 0;
}

.filter-group {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.filter-group label {
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 14px;
  color: #333;
  cursor: pointer;
}

.filter-group input[type="checkbox"] {
  width: 18px;
  height: 18px;
  cursor: pointer;
  accent-color: #3498db;
}

.location-btn {
  width: 100%;
  padding: 12px;
  border: none;
  border-radius: 6px;
  background: #27ae60;
  color: white;
  font-weight: bold;
  font-size: 14px;
  cursor: pointer;
  transition: background 0.2s;
}

.location-btn:hover {
  background: #219a52;
}

.location-status {
  margin-top: 10px;
  font-size: 12px;
  color: #666;
  word-break: break-word;
}

:deep(.leaflet-container){
  background: #2c3e50;
}

:deep(.leaflet-popup-content-wrapper) {
  border-radius: 8px;
}

:deep(.marker-icon) {
  width: auto !important;
}

/*====================================================
MARCADOR DO USUÁRIO
====================================================*/
:deep(.user-location-marker) {
  position: relative;
}

:deep(.user-location-marker .dot) {
  width: 24px;
  height: 24px;
  background: #3498db;
  border: 4px solid white;
  border-radius: 50%;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  box-shadow: 0 3px 8px rgba(0, 0, 0, 0.4);
}

:deep(.user-location-marker .pulse) {
  width: 75px;
  height: 75px;
  background: rgba(52, 152, 219, 0.4);
  border-radius: 50%;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  animation: pulse 2s infinite ease-out;
}

@keyframes pulse {
  0% {
    transform: translate(-50%, -50%) scale(0.4);
    opacity: 1;
  }
  100% {
    transform: translate(-50%, -50%) scale(1.6);
    opacity: 0;
  }
}
</style>