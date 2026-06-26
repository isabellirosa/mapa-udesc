<template>
  <div class="map-wrapper">
    <div v-if="mostrarPopup" class="modal-overlay">
      <div class="modal-content">
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
          Blocos (de A até N)
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
          <input type="checkbox" v-model="filtros.mapatatil" />
          Mapas Táteis
        </label>
        <label>
          <input type="checkbox" v-model="filtros.banheiros" />
          Banheiros Acessíveis
        </label>
        <label>
          <input type="checkbox" v-model="filtros.entradas" />
          Outros
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

delete L.Icon.Default.prototype._getIconUrl;
L.Icon.Default.mergeOptions({
  iconRetinaUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-icon-2x.png',
  iconUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-icon.png',
  shadowUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-shadow.png',
});

const mapa = ref(null);
const menuAberto = ref(true);
const localizacaoStatus = ref("");
const mostrarPopup = ref(false);

let camadaImagemOverlay = null;
let marcadoresAtuais = [];
let marcadorUsuario = null;
let watchId = null;


const filtros = reactive({
  blocos: true,
  estruturas: true,
  acessibilidade: false,
  mapatatil: false,
  banheiros: false,
  entradas: true,
});


const cantoInferiorEsquerdo = [-26.255592, -48.857007];
const cantoSuperiorDireito = [-26.25116, -48.852338];
const limitesUniversidade = L.latLngBounds(cantoInferiorEsquerdo, cantoSuperiorDireito);


const limitesExpandidosBrancos = limitesUniversidade.pad(0.15);


const pontosDeInteresse = [

  { nome: "univille", coordenadas: [-26.251942488469627, -48.85569276267728], categoria: "entradas", tamanho: 80, titulo: "Universidade Univille", desc: "Universidade da Região de Joinville (Univille) é uma instituição de ensino superior privada." },
  { nome: "garten", coordenadas: [-26.25383115301317, -48.853101890364904], categoria: "entradas", tamanho: 80, titulo: " Garten Shopping", desc: "Centro de compras, lazer e gastronomia da região Norte de Santa Catarina." },
  { nome: "portao", coordenadas: [-26.252061, -48.854161], categoria: "entradas", tamanho: 55, titulo: "Guarita", desc: "Acesso para pedestres e veículos." },
  { nome: "a", coordenadas: [-26.252243590865646, -48.85449532462595], categoria: "blocos", tamanho: 55, titulo: "Bloco A", desc: "Recepção Geral e Secretaria de Graduação." },
  { nome: "b", coordenadas: [-26.252476937034576, -48.85471255925737], categoria: "blocos", tamanho: 55, titulo: "Bloco B", desc: "Laboratórios e Salas de Aula." },
  { nome: "c", coordenadas: [-26.252600188291428, -48.85502835627211], categoria: "blocos", tamanho: 55, titulo: "Bloco C", desc: "Licenciatura em Física, Licenciatura em Matemática e Licenciatura em Química." },
  { nome: "d", coordenadas: [-26.252804915733297, -48.854714066797264], categoria: "blocos", tamanho: 55, titulo: "Bloco D", desc: "Laboratórios Multidisciplinares de Pesquisa e Práticas." },
  { nome: "e", coordenadas: [-26.253161, -48.854839], categoria: "blocos", tamanho: 55, titulo: "Bloco E", desc: "Departamento de Engenharia Elétrica e Auditório do Bloco." },
  { nome: "k", coordenadas: [-26.25340301156343, -48.854834815446246], categoria: "blocos", tamanho: 55, titulo: "Bloco K", desc: "Bloco de Salas de Aula Gerais." },
  { nome: "f", coordenadas: [-26.25362258956487, -48.85489194329959], categoria: "blocos", tamanho: 55, titulo: "Bloco F", desc: "Ciência da Computação, TADS, Auditório e Reprografia Central." },
  { nome: "g", coordenadas: [-26.25410200011257, -48.85460630403199], categoria: "blocos", tamanho: 55, titulo: "Bloco G", desc: "Departamento de Engenharia Mecânica e Oficinas Técnicas." },
  { nome: "h", coordenadas: [-26.254036127022285, -48.855299999429725], categoria: "blocos", tamanho: 55, titulo: "Bloco H", desc: "Laboratórios práticos e salas de aula da Engenharia Civil." },
  { nome: "i", coordenadas: [-26.253036346320055, -48.85532941250589], categoria: "blocos", tamanho: 55, titulo: "Bloco I", desc: "Biblioteca Universitária, Auditório Principal e Salas de Aula." },
  { nome: "j", coordenadas: [-26.253124878831727, -48.85598553368974], categoria: "blocos", tamanho: 55, titulo: "Bloco J", desc: "Centro de Projetos Multidisciplinares e de Visitação da UDESC Joinville." },
  { nome: "l", coordenadas: [-26.254402088132885, -48.85582639179274], categoria: "blocos", tamanho: 55, titulo: "Bloco L", desc: "Engenharia de Produção e Secretaria Integrada de Engenharia Civil." },
  { nome: "n", coordenadas: [-26.252570935150995, -48.855270774266835], categoria: "blocos", tamanho: 55, titulo: "Bloco N", desc: "Atualmente em obras" },
  { nome: "m", coordenadas: [-26.2545386681672, -48.855117896870645], categoria: "blocos", tamanho: 55, titulo: "Bloco M", desc: "Setor de Serviços e a Prefeitura do Campus.." },
  { nome: "refeitorio", coordenadas: [-26.253403011562707, -48.85596105032403], categoria: "estruturas", tamanho: 55, titulo: "Restaurante Universitário", desc: "Refeitório do Campus, Centros Acadêmicos (CAs) e Serviço de Orientação ao Estudante (SOE)." },
  { nome: "quadra", coordenadas: [-26.25290164023696, -48.85585495573923], categoria: "estruturas", tamanho: 55, titulo: "Ginásio Universitário", desc: "Quadras Poliesportivas para eventos e práticas acadêmicas." },
  { nome: "academia", coordenadas: [-26.252535674399454, -48.8557611028373], categoria: "estruturas", tamanho: 55, titulo: "Academia Univille", desc: "Academia do Campus e Espaço de Cuidado com a Saúde." },
  { nome: "elevador", coordenadas: [-26.252981, -48.855392], categoria: "acessibilidade", tamanho: 55, titulo: "Elevador Acessível", desc: "Equipamento de acessibilidade motora para deslocamento entre andares." },
  { nome: "elevador", coordenadas: [-26.253127947070404, -48.85540673086851], categoria: "acessibilidade", tamanho: 55, titulo: "Elevador Acessível", desc: "Equipamento de acessibilidade motora para deslocamento entre andares." },
  { nome: "elevador", coordenadas: [-26.253573181365446, -48.85462903208101], categoria: "acessibilidade", tamanho: 55, titulo: "Elevador Acessível", desc: "Equipamento de acessibilidade motora para deslocamento entre andares." },
  { nome: "elevador", coordenadas: [-26.25346140385683, -48.85462545367419], categoria: "acessibilidade", tamanho: 55, titulo: "Elevador Acessível", desc: "Equipamento de acessibilidade motora para deslocamento entre andares." },
  { nome: "elevador", coordenadas: [-26.25220831673813, -48.854591525001425], categoria: "acessibilidade", tamanho: 55, titulo: "Elevador Acessível", desc: "Equipamento de acessibilidade motora para deslocamento entre andares." },
  { nome: "elevador", coordenadas: [-26.252546236551225, -48.855005558211815], categoria: "acessibilidade", tamanho: 55, titulo: "Elevador Acessível", desc: "Equipamento de acessibilidade motora para deslocamento entre andares." },
  { nome: "mapatatil", coordenadas: [-26.254424020590726, -48.855526879967876], categoria: "mapatatil", tamanho: 55, titulo: "Mapa Tátil", desc: "Dispositivo de orientação em braile e alto-relevo para deficientes visuais." },
  { nome: "mapatatil", coordenadas: [-26.254008349732562, -48.85518012544649], categoria: "mapatatil", tamanho: 55, titulo: "Mapa Tátil", desc: "Dispositivo de orientação em braile e alto-relevo para deficientes visuais." },
  { nome: "mapatatil", coordenadas: [-26.253974480183118, -48.854459150695966], categoria: "mapatatil", tamanho: 55, titulo: "Mapa Tátil", desc: "Dispositivo de orientação em braile e alto-relevo para deficientes visuais." },
  { nome: "mapatatil", coordenadas: [-26.253221760649925, -48.85443738926518], categoria: "mapatatil", tamanho: 55, titulo: "Mapa Tátil", desc: "Dispositivo de orientação em braile e alto-relevo para deficientes visuais." },
  { nome: "mapatatil", coordenadas: [-26.25336367731001, -48.855769747258755], categoria: "mapatatil", tamanho: 55, titulo: "Mapa Tátil", desc: "Dispositivo de orientação em braile e alto-relevo para deficientes visuais." },
  { nome: "mapatatil", coordenadas: [-26.252598587774777, -48.85466228713793], categoria: "mapatatil", tamanho: 55, titulo: "Mapa Tátil", desc: "Dispositivo de orientação em braile e alto-relevo para deficientes visuais." },
  { nome: "mapatatil", coordenadas: [-26.252317589987218, -48.85463126585165], categoria: "mapatatil", tamanho: 55, titulo: "Mapa Tátil", desc: "Dispositivo de orientação em braile e alto-relevo para deficientes visuais." },
  { nome: "mapatatil", coordenadas: [-26.25225827029245, -48.854302261006026], categoria: "mapatatil", tamanho: 55, titulo: "Mapa Tátil", desc: "Dispositivo de orientação em braile e alto-relevo para deficientes visuais." },
  { nome: "escada", coordenadas: [-26.253722572101097, -48.85542230877457], categoria: "acessibilidade", tamanho: 55, titulo: "Escada", desc: "Acesso limitado para pessoas com mobilidade reduzida." },
  { nome: "escada", coordenadas: [-26.252876802727187, -48.85555570032768], categoria: "acessibilidade", tamanho: 55, titulo: "Escada", desc: "Acesso limitado para pessoas com mobilidade reduzida." },
  { nome: "escada", coordenadas: [-26.25371144358737, -48.85442652530534], categoria: "acessibilidade", tamanho: 55, titulo: "Escada", desc: "Acesso limitado para pessoas com mobilidade reduzida." },
  { nome: "rampa", coordenadas: [-26.253755957604582, -48.85516793417577], categoria: "acessibilidade", tamanho: 55, titulo: "Rampa de Acessibilidade", desc: "Acesso inclinado regulamentado para cadeirantes e pessoas com mobilidade reduzida." },
  { nome: "rampa", coordenadas: [-26.25350834820514, -48.85480498507884], categoria: "acessibilidade", tamanho: 55, titulo: "Rampa de Acessibilidade", desc: "Acesso inclinado regulamentado para cadeirantes e pessoas com mobilidade reduzida." },
  { nome: "rampa", coordenadas: [-26.25265352969627, -48.85436244961437], categoria: "acessibilidade", tamanho: 55, titulo: "Rampa de Acessibilidade", desc: "Acesso inclinado regulamentado para cadeirantes e pessoas com mobilidade reduzida." },
  { nome: "rampa", coordenadas: [-26.253107449502224, -48.85433996115613], categoria: "acessibilidade", tamanho: 55, titulo: "Rampa de Acessibilidade", desc: "Acesso inclinado regulamentado para cadeirantes e pessoas com mobilidade reduzida." },
  { nome: "rampa", coordenadas: [-26.25411763330989, -48.85445134233276], categoria: "acessibilidade", tamanho: 55, titulo: "Rampa de Acessibilidade", desc: "Acesso inclinado regulamentado para cadeirantes e pessoas com mobilidade reduzida." },
  { nome: "banheiro", coordenadas: [-26.254402362511847, -48.855679189426915], categoria: "banheiros", tamanho: 55, titulo: "Sanitário Acessível", desc: "Instalação sanitária totalmente adaptada para PNE (Pessoas com Necessidades Especiais)." },
  { nome: "banheiro", coordenadas: [-26.25455060679538, -48.85504024403134], categoria: "banheiros", tamanho: 55, titulo: "Sanitário Acessível", desc: "Instalação sanitária totalmente adaptada para PNE (Pessoas com Necessidades Especiais)." },
  { nome: "banheiro", coordenadas: [-26.253880052077417, -48.85485165855539], categoria: "banheiros", tamanho: 55, titulo: "Sanitário Acessível", desc: "Instalação sanitária totalmente adaptada para PNE (Pessoas com Necessidades Especiais)." },
  { nome: "banheiro", coordenadas: [-26.25425214077942, -48.854871626425684], categoria: "banheiros", tamanho: 55, titulo: "Sanitário Acessível", desc: "Instalação sanitária totalmente adaptada para PNE (Pessoas com Necessidades Especiais)." },
  { nome: "banheiro", coordenadas: [-26.25409693814938, -48.8553996656961], categoria: "banheiros", tamanho: 55, titulo: "Sanitário Acessível", desc: "Instalação sanitária totalmente adaptada para PNE (Pessoas com Necessidades Especiais)." },
  { nome: "banheiro", coordenadas: [-26.253581290571642, -48.85468588684532], categoria: "banheiros", tamanho: 55, titulo: "Sanitário Acessível", desc: "Instalação sanitária totalmente adaptada para PNE (Pessoas com Necessidades Especiais)." },
  { nome: "banheiro", coordenadas: [-26.253352594121818, -48.85480692262471], categoria: "banheiros", tamanho: 55, titulo: "Sanitário Acessível", desc: "Instalação sanitária totalmente adaptada para PNE (Pessoas com Necessidades Especiais)." },
  { nome: "banheiro", coordenadas: [-26.253306830438316, -48.85586989210336], categoria: "banheiros", tamanho: 55, titulo: "Sanitário Acessível", desc: "Instalação sanitária totalmente adaptada para PNE (Pessoas com Necessidades Especiais)." },
  { nome: "banheiro", coordenadas: [-26.25304939291578, -48.856019939355896], categoria: "banheiros", tamanho: 55, titulo: "Sanitário Acessível", desc: "Instalação sanitária totalmente adaptada para PNE (Pessoas com Necessidades Especiais)." },
  { nome: "banheiro", coordenadas: [-26.252583273510993, -48.855632860923876], categoria: "banheiros", tamanho: 55, titulo: "Sanitário Acessível", desc: "Instalação sanitária totalmente adaptada para PNE (Pessoas com Necessidades Especiais)." },
  { nome: "banheiro", coordenadas: [-26.252942236545156, -48.8554271130699], categoria: "banheiros", tamanho: 55, titulo: "Sanitário Acessível", desc: "Instalação sanitária totalmente adaptada para PNE (Pessoas com Necessidades Especiais)." },
  { nome: "banheiro", coordenadas: [-26.253144576363262, -48.85493515278803], categoria: "banheiros", tamanho: 55, titulo: "Sanitário Acessível", desc: "Instalação sanitária totalmente adaptada para PNE (Pessoas com Necessidades Especiais)." },
  { nome: "banheiro", coordenadas: [-26.252668705890002, -48.85500691451774], categoria: "banheiros", tamanho: 55, titulo: "Sanitário Acessível", desc: "Instalação sanitária totalmente adaptada para PNE (Pessoas com Necessidades Especiais)." },
  { nome: "banheiro", coordenadas: [-26.25271356261427, -48.854748137360694], categoria: "banheiros", tamanho: 55, titulo: "Sanitário Acessível", desc: "Instalação sanitária totalmente adaptada para PNE (Pessoas com Necessidades Especiais)." },
  { nome: "banheiro", coordenadas: [-26.252594594737655, -48.854600264706015], categoria: "banheiros", tamanho: 55, titulo: "Sanitário Acessível", desc: "Instalação sanitária totalmente adaptada para PNE (Pessoas com Necessidades Especiais)." },
  { nome: "banheiro", coordenadas: [-26.252300099961662, -48.85440890009408], categoria: "banheiros", tamanho: 55, titulo: "Sanitário Acessível", desc: "Instalação sanitária totalmente adaptada para PNE (Pessoas com Necessidades Especiais)." },
];

onMounted(() => {
  mapa.value = L.map("map", {
    maxBounds: limitesExpandidosBrancos,
    maxBoundsViscosity: 0.4,
    maxZoom: 22,
    zoomControl: true,
    zoomSnap: 1,
    zoomDelta: 1,

    zoomAnimation: false,
    fadeAnimation: false,
    markerZoomAnimation: false,
  });

  configurarLimitesEZoom();
  atualizarImagemEAndar();
  atualizarMarcadores();

  mostrarPopup.value = true;

  window.addEventListener("resize", configurarLimitesEZoom);
});

onUnmounted(() => {
  if (watchId !== null) {
    navigator.geolocation.clearWatch(watchId);
  }
  window.removeEventListener("resize", configurarLimitesEZoom);
});


const configurarLimitesEZoom = () => {
  if (!mapa.value) return;

  const centro = limitesUniversidade.getCenter();

  const zoomInicial = 18;
  const zoomMinimo = 17;

  mapa.value.setMinZoom(zoomMinimo);

  mapa.value.setView(centro, zoomInicial);
};

const limparMarcadores = () => {
  marcadoresAtuais.forEach((m) => mapa.value.removeLayer(m));
  marcadoresAtuais = [];
};

const atualizarMarcadores = () => {
  console.log("Atualizando");
  limparMarcadores();

  pontosDeInteresse.forEach((ponto) => {
    console.log(ponto.nome);
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
        <div style="font-family: sans-serif; min-width: 200px; padding: 5px;">
          <h4 style="margin: 0 0 6px 0; color: #2c3e50; font-size: 15px; border-bottom: 1px solid #eef2f5; padding-bottom: 4px;">
            ${ponto.titulo}
          </h4>
          <p style="margin: 0; font-size: 12.5px; color: #5a6c7d; line-height: 1.4; text-align: justify;">
            ${ponto.desc}
          </p>
        </div>
      `);

    marcadoresAtuais.push(marcador);
  });
};

watch(filtros, atualizarMarcadores, { deep: true });

const atualizarImagemEAndar = () => {
  if (camadaImagemOverlay) {
    mapa.value.removeLayer(camadaImagemOverlay);
  }

  const url = "/mapaOficial.png";

  camadaImagemOverlay = L.imageOverlay(url, limitesUniversidade, {
    opacity: 1,
    interactive: false
  }).addTo(mapa.value);

  camadaImagemOverlay.bringToFront();
};


const aceitarLocalizacao = () => {
  mostrarPopup.value = false;
  ativarLocalizacao();
};

const recusarLocalizacao = () => {
  mostrarPopup.value = false;
  localizacaoStatus.value = "Localização não ativada.";
};

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
  z-index: 2000;
}

.modal-content {
  background: white;
  padding: 30px;
  border-radius: 20px;
  width: 90%;
  max-width: 400px;
  text-align: center;
  box-shadow: 0 15px 40px rgba(0, 0, 0, 0.3);
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
  box-shadow: 0 8px 20px rgba(0, 0, 0, .15);
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
  box-shadow: 0 12px 35px rgba(0, 0, 0, .18);
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

:deep(.leaflet-container) {
  background: #ffffff;
}

:deep(.leaflet-popup-content-wrapper) {
  border-radius: 8px;
}

:deep(.marker-icon) {
  width: auto !important;
}

:deep(.user-location-marker) {
  position: relative;
}

:deep(.user-location-marker .dot) {
  width: 24px;
  height: 24px;
  background: #ffffff;
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
  background: rgb(255, 255, 255);
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