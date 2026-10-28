<!DOCTYPE html>
<html lang="es" class="h-full bg-slate-950 text-slate-100">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
    <title>Costa Palmas — Dashboard Operativo</title>
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <script>
        tailwind.config = {
            darkMode: 'class',
        }
    </script>
    <!-- Leaflet CSS -->
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <!-- FontAwesome -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <style>
        html, body {
            background-color: #020617 !important;
            color: #f8fafc !important;
            height: 100%;
            height: 100dvh;
            width: 100vw;
            margin: 0;
            padding: 0;
            overflow: hidden;
            -webkit-text-size-adjust: 100%;
            -webkit-font-smoothing: antialiased;
        }

        #view-map-leaflet {
            position: absolute !important;
            inset: 0 !important;
            width: 100% !important;
            height: 100% !important;
            z-index: 20;
            background: #020617;
        }

        #leaflet-map {
            width: 100% !important;
            height: 100% !important;
            z-index: 10;
            touch-action: pan-x pan-y;
        }

        .map-container {
            position: relative;
            display: inline-block;
            margin: 0 auto;
            touch-action: pan-x pan-y pinch-zoom;
            user-select: none;
            -webkit-user-select: none;
        }

        .sector-polygon {
            fill-opacity: 0.35;
            stroke-width: 3;
            cursor: pointer;
            transition: fill-opacity 0.2s ease, stroke-width 0.2s ease;
        }
        .sector-polygon:hover, .sector-polygon.active {
            fill-opacity: 0.65;
            stroke-width: 5;
            filter: drop-shadow(0 0 8px rgba(255,255,255,0.6));
        }

        .sec-1 { fill: #22c55e; stroke: #15803d; }
        .sec-2 { fill: #3b82f6; stroke: #1d4ed8; }
        .sec-3 { fill: #eab308; stroke: #ca8a04; }
        .sec-4 { fill: #f97316; stroke: #c2410c; }
        .sec-aman { fill: #a855f7; stroke: #6b21a8; }

        .access-hitbox { cursor: pointer; }
        .access-hitbox:hover circle.pin-marker {
            fill: #ef4444 !important;
            r: 18 !important;
        }

        .bottom-sheet {
            transform: translateY(100%);
            transition: transform 0.3s cubic-bezier(0.16, 1, 0.3, 1);
        }
        .bottom-sheet.open {
            transform: translateY(0);
        }

        .no-scrollbar::-webkit-scrollbar { display: none; }
        .no-scrollbar { -ms-overflow-style: none; scrollbar-width: none; }
    </style>
</head>
<body class="bg-slate-950 flex flex-col h-[100dvh] w-screen overflow-hidden m-0 p-0 text-slate-100">

    <!-- PANTALLA DE LOGIN -->
    <div id="auth-screen" class="fixed inset-0 z-50 bg-slate-950 flex items-center justify-center p-4">
        <div class="bg-slate-900 w-full max-w-sm rounded-3xl p-6 shadow-2xl space-y-5 relative flex flex-col justify-between border border-slate-800" style="max-height: 520px;">
            <div class="space-y-4">
                <div class="text-center">
                    <h2 class="text-2xl font-bold text-white">Costa Palmas</h2>
                    <p class="text-xs text-slate-400 mt-1">Acceso Autorizado — Bellman & Transport</p>
                </div>

                <div class="space-y-3">
                    <div>
                        <label class="block text-[10px] font-semibold text-slate-300 mb-1">USUARIO</label>
                        <input type="text" id="auth-user" class="w-full px-3 py-2 border border-slate-700 rounded-xl focus:outline-none focus:ring-2 focus:ring-blue-500 text-sm text-white bg-slate-800" placeholder="Ej. xasharitl, francisco...">
                    </div>
                    <div>
                        <label class="block text-[10px] font-semibold text-slate-300 mb-1">CONTRASEÑA</label>
                        <input type="password" id="auth-pass" class="w-full px-3 py-2 border border-slate-700 rounded-xl focus:outline-none focus:ring-2 focus:ring-blue-500 text-sm text-white bg-slate-800" placeholder="••••••••">
                    </div>

                    <button onclick="handleAuth()" id="auth-btn" class="w-full bg-blue-600 text-white font-semibold py-2.5 rounded-xl hover:bg-blue-700 transition shadow-md text-xs">
                        Iniciar Sesión
                    </button>
                </div>
            </div>

            <div class="pt-3 border-t border-slate-800 flex justify-between items-center text-[10px] text-slate-400">
                <span>Verificación en la Nube (Excel)</span>
                <span class="text-slate-500 font-medium">Costa Palmas Security</span>
            </div>
        </div>
    </div>

    <!-- MODAL PERFIL DE OPERADOR -->
    <div id="admin-modal" class="fixed inset-0 z-50 bg-black/70 hidden items-center justify-center p-4">
        <div class="bg-slate-900 text-white w-full max-w-md rounded-3xl p-6 shadow-2xl space-y-4 max-h-[90vh] flex flex-col border border-slate-800">
            <div class="flex justify-between items-center border-b border-slate-800 pb-3">
                <h3 class="font-bold text-lg">Perfil de Operador</h3>
                <button onclick="toggleAdminPanel()" class="text-slate-400 hover:text-white font-bold text-xl">&times;</button>
            </div>

            <div class="flex-1 space-y-3 py-1">
                <p class="text-xs text-slate-400">Datos de la sesión activa:</p>
                <div class="bg-slate-800 p-3 rounded-xl border border-slate-700 space-y-2 text-xs">
                    <div>
                        <label class="font-semibold text-slate-300">Usuario Actual:</label>
                        <input type="text" id="profile-user" readonly class="w-full mt-1 p-2 bg-slate-700 border border-slate-600 rounded-lg text-slate-200 font-medium">
                    </div>
                    <div>
                        <label class="font-semibold text-slate-300">Nombre Completo / Puesto:</label>
                        <input type="text" id="profile-name" class="w-full mt-1 p-2 bg-slate-700 border border-slate-600 rounded-lg text-white">
                    </div>
                    <button onclick="saveProfileData()" class="w-full bg-blue-600 hover:bg-blue-700 text-white py-2 rounded-lg font-semibold transition">Guardar Cambios</button>
                </div>
            </div>

            <div class="pt-3 border-t border-slate-800 flex justify-between items-center">
                <button onclick="clearActiveSession()" class="text-xs bg-red-900/30 text-red-400 px-3 py-2 rounded-xl font-semibold hover:bg-red-900/50 transition flex items-center gap-1">
                    <i class="fa-solid fa-right-from-bracket"></i> Cerrar sesión
                </button>
                <button onclick="toggleAdminPanel()" class="bg-slate-700 text-white text-xs px-4 py-2 rounded-xl font-semibold hover:bg-slate-600 transition">
                    Cerrar Panel
                </button>
            </div>
        </div>
    </div>

    <!-- CONTENEDOR PRINCIPAL -->
    <div id="appContainer" class="flex flex-col h-full w-full bg-slate-950 text-slate-100 relative overflow-hidden">
        
        <!-- ENCABEZADO -->
        <header id="appHeader" class="bg-slate-900 text-white p-3.5 shadow-md shrink-0 z-30 border-b border-slate-800">
            <div class="flex justify-between items-center max-w-4xl mx-auto w-full">
                <div class="flex items-center gap-2.5">
                    <div class="bg-amber-500/20 text-amber-400 p-2 rounded-xl border border-amber-500/30">
                        <i class="fa-solid fa-map-location-dot text-base"></i>
                    </div>
                    <div>
                        <h1 class="font-bold text-sm text-white">Costa Palmas</h1>
                        <div class="flex items-center gap-2 mt-0.5">
                            <span id="syncIndicatorDot" class="w-2 h-2 rounded-full bg-green-400 animate-pulse"></span>
                            <p id="syncStatus" class="text-[10px] text-slate-400">Sincronizando...</p>
                        </div>
                    </div>
                </div>

                <div class="flex items-center gap-2">
                    <button onclick="toggleAdminPanel()" id="userBadgeBtn" class="w-9 h-9 rounded-full bg-blue-800 hover:bg-blue-700 flex items-center justify-center font-bold border border-blue-600 text-xs transition-colors text-white shadow-sm" title="Panel de Operador">
                        CP
                    </button>
                </div>
            </div>
        </header>

        <!-- CONTENIDO PRINCIPAL -->
        <main class="flex-1 overflow-y-auto relative flex flex-col no-scrollbar w-full max-w-4xl mx-auto">
            
            <!-- VISTA 1: DIRECTORIO DE RESIDENTES (2 COLUMNAS) -->
            <div id="view-directory" class="flex flex-col w-full pb-24">
                <div class="sticky top-0 z-10 bg-slate-950/95 backdrop-blur-md px-4 pt-3 pb-3 space-y-2.5 border-b border-slate-900 shadow-lg">
                    <div class="relative">
                        <span class="absolute inset-y-0 left-0 flex items-center pl-3 text-slate-400">
                            <i class="fa-solid fa-magnifying-glass"></i>
                        </span>
                        <input type="text" id="searchInput" placeholder="Buscar por nombre, villa o zona (ej. OV-, R7)..." 
                            class="w-full pl-10 pr-4 py-2.5 bg-slate-900 border border-slate-800 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-blue-600 shadow-sm text-white placeholder-slate-500"
                            oninput="filterDirectory()">
                    </div>

                    <div class="flex gap-2 overflow-x-auto no-scrollbar pb-1 text-xs">
                        <button onclick="filterByZone('all')" id="btn-zone-all" class="px-3 py-1.5 rounded-full font-medium bg-blue-600 text-white shrink-0 shadow-sm transition-colors">Todas</button>
                        <button onclick="filterByZone('Beach & Yacht Club')" id="btn-zone-beach" class="px-3 py-1.5 rounded-full font-medium bg-slate-900 text-slate-300 border border-slate-800 shrink-0 shadow-sm transition-colors">Beach & Yacht Club</button>
                        <button onclick="filterByZone('Residencial')" id="btn-zone-residential" class="px-3 py-1.5 rounded-full font-medium bg-slate-900 text-slate-300 border border-slate-800 shrink-0 shadow-sm transition-colors">Residencial</button>
                    </div>
                </div>

                <div id="residentList" class="grid grid-cols-2 gap-3 p-4">
                    <div class="col-span-2 text-center py-10 text-slate-500 text-sm">Cargando directorio...</div>
                </div>
            </div>

            <!-- VISTA 2: MAPA GPS LEAFLET -->
            <div id="view-map-leaflet" class="hidden absolute top-0 left-0 w-full h-full z-20 bg-slate-950">
                <div id="leaflet-map" class="w-full h-full"></div>
            </div>

            <!-- VISTA 3: MAPA OPERATIVO SVG Y SECTORES (SIN MARCO EXTREMO) -->
            <div id="view-map-svg" class="hidden px-2 pt-2 flex flex-col items-center justify-start pb-32 w-full">
                <div class="w-full max-w-xl flex flex-col items-center">
                    <div class="map-container w-full overflow-hidden relative">
                        <img id="mapImage" src="https://lh3.googleusercontent.com/d/18ATCCK_U9Gm-XyPkiipc7wgZeStHkQs0=w1600" 
                             alt="Mapa Satelital Costa Palmas" class="w-full h-auto block object-contain select-none pointer-events-none"
                             onerror="this.onerror=null; this.src='https://placehold.co/1000x1000/0f172a/38bdf8?text=Costa+Palmas+Mapa';">

                        <svg id="svgOverlay" viewBox="0 0 1000 1000" class="absolute top-0 left-0 w-full h-full">
                            <polygon id="poly-sec-1" points="449,926 321,1019 268,997 250,697 495,450 622,498 546,745 500,740 423,808" class="sector-polygon sec-1" onclick="selectSector('sec-1')"></polygon>
                            <polygon id="poly-sec-2" points="598,161 792,271 996,312 978,438 723,492 657,444 624,265" class="sector-polygon sec-2" onclick="selectSector('sec-2')"></polygon>
                            <polygon id="poly-sec-3" points="575,299 645,341 656,477 614,492 398,424 398,329" class="sector-polygon sec-3" onclick="selectSector('sec-3')"></polygon>
                            <polygon id="poly-sec-4" points="164,372 281,80 569,130 576,298 398,329 399,422" class="sector-polygon sec-4" onclick="selectSector('sec-4')"></polygon>
                            <polygon id="poly-sec-aman" points="63,54 282,80 163,372 15,346 20,200" class="sector-polygon sec-aman" onclick="selectSector('sec-aman')"></polygon>

                            <g id="accessGroup">
                                <g class="access-hitbox" transform="translate(385, 910)" onclick="selectAccess(1)"><circle class="pin-marker" r="16" fill="#22c55e" opacity="0.4"/><circle r="22" fill="transparent"/></g>
                                <g class="access-hitbox" transform="translate(535, 715)" onclick="selectAccess(2)"><circle class="pin-marker" r="16" fill="#22c55e" opacity="0.4"/><circle r="22" fill="transparent"/></g>
                                <g class="access-hitbox" transform="translate(595, 515)" onclick="selectAccess(3)"><circle class="pin-marker" r="16" fill="#22c55e" opacity="0.4"/><circle r="22" fill="transparent"/></g>
                                <g class="access-hitbox" transform="translate(535, 495)" onclick="selectAccess(4)"><circle class="pin-marker" r="16" fill="#22c55e" opacity="0.4"/><circle r="22" fill="transparent"/></g>
                                <g class="access-hitbox" transform="translate(915, 460)" onclick="selectAccess(5)"><circle class="pin-marker" r="16" fill="#3b82f6" opacity="0.4"/><circle r="22" fill="transparent"/></g>
                                <g class="access-hitbox" transform="translate(750, 485)" onclick="selectAccess(6)"><circle class="pin-marker" r="16" fill="#3b82f6" opacity="0.4"/><circle r="22" fill="transparent"/></g>
                                <g class="access-hitbox" transform="translate(620, 475)" onclick="selectAccess(7)"><circle class="pin-marker" r="16" fill="#eab308" opacity="0.4"/><circle r="22" fill="transparent"/></g>
                                <g class="access-hitbox" transform="translate(555, 470)" onclick="selectAccess(8)"><circle class="pin-marker" r="16" fill="#eab308" opacity="0.4"/><circle r="22" fill="transparent"/></g>
                                <g class="access-hitbox" transform="translate(370, 410)" onclick="selectAccess(9)"><circle class="pin-marker" r="16" fill="#f97316" opacity="0.4"/><circle r="22" fill="transparent"/></g>
                                <g class="access-hitbox" transform="translate(160, 380)" onclick="selectAccess(10)"><circle class="pin-marker" r="16" fill="#a855f7" opacity="0.4"/><circle r="22" fill="transparent"/></g>
                                <g class="access-hitbox" transform="translate(20, 360)" onclick="selectAccess(11)"><circle class="pin-marker" r="16" fill="#a855f7" opacity="0.4"/><circle r="22" fill="transparent"/></g>
                            </g>
                        </svg>
                    </div>

                    <div class="mt-2 text-center text-[11px] text-slate-400 flex items-center justify-center gap-1.5">
                        <i class="fa-solid fa-hand-pointer text-amber-400"></i>
                        <span>Toca cualquier sector</span>
                    </div>
                </div>
            </div>

        </main>

        <!-- BARRA INFERIOR FIJA PARA SECTORES, RADIOS Y EMERGENCIA -->
        <div id="sectorBottomBar" class="hidden fixed bottom-16 inset-x-0 bg-slate-900/95 border-t border-slate-800 p-2 z-20 shadow-xl backdrop-blur">
            <div class="max-w-xl mx-auto flex flex-col gap-1.5">
                <div class="grid grid-cols-5 gap-1 text-[11px]">
                    <button onclick="selectSector('sec-1')" class="py-2 px-1 rounded-lg bg-slate-800 hover:bg-slate-700 border border-slate-700 text-slate-200 font-bold text-center truncate">Sec. 1</button>
                    <button onclick="selectSector('sec-2')" class="py-2 px-1 rounded-lg bg-slate-800 hover:bg-slate-700 border border-slate-700 text-slate-200 font-bold text-center truncate">Sec. 2</button>
                    <button onclick="selectSector('sec-3')" class="py-2 px-1 rounded-lg bg-slate-800 hover:bg-slate-700 border border-slate-700 text-slate-200 font-bold text-center truncate">Sec. 3</button>
                    <button onclick="selectSector('sec-4')" class="py-2 px-1 rounded-lg bg-slate-800 hover:bg-slate-700 border border-slate-700 text-slate-200 font-bold text-center truncate">Sec. 4</button>
                    <button onclick="selectSector('sec-aman')" class="py-2 px-1 rounded-lg bg-slate-800 hover:bg-slate-700 border border-slate-700 text-slate-200 font-bold text-center truncate">Amán</button>
                </div>
                <div class="grid grid-cols-2 gap-2">
                    <button onclick="openRadiosModal()" class="w-full py-2.5 px-3 rounded-xl bg-indigo-600 hover:bg-indigo-500 text-white font-bold shadow text-xs flex items-center justify-center gap-2 transition">
                        <i class="fa-solid fa-walkie-talkie text-base"></i>
                        <span>Canales de Radio</span>
                    </button>
                    <button onclick="openEmergencyModal()" class="w-full py-2.5 px-3 rounded-xl bg-rose-600 hover:bg-rose-500 text-white font-bold shadow text-xs flex items-center justify-center gap-2 transition">
                        <i class="fa-solid fa-phone-volume text-base"></i>
                        <span>Emergencias</span>
                    </button>
                </div>
            </div>
        </div>

        <!-- NAVEGACIÓN INFERIOR FIJA -->
        <nav id="appNav" class="bg-slate-900 border-t border-slate-800 h-16 flex justify-around items-center px-2 shrink-0 z-30 w-full">
            <button onclick="switchTab('directory')" id="nav-dir" class="flex flex-col items-center text-blue-500 transition-colors">
                <i class="fa-solid fa-address-book text-base"></i>
                <span class="text-[9px] font-medium mt-1">Directorio</span>
            </button>
            <button onclick="switchTab('map')" id="nav-map" class="flex flex-col items-center text-slate-400 hover:text-slate-200 transition-colors">
                <i class="fa-solid fa-map-location-dot text-base"></i>
                <span class="text-[9px] font-medium mt-1">Mapa GPS</span>
            </button>
            <button onclick="switchTab('svg')" id="nav-svg" class="flex flex-col items-center text-slate-400 hover:text-slate-200 transition-colors">
                <i class="fa-solid fa-map text-base"></i>
                <span class="text-[9px] font-medium mt-1">Sectores</span>
            </button>
        </nav>

    </div>

    <!-- MODAL DE DETALLE DE RESIDENTE -->
    <div id="detailModal" class="fixed inset-0 bg-black/70 z-50 hidden flex items-end justify-center">
        <div id="modalContentBox" class="bg-slate-900 text-white w-full max-w-lg rounded-t-3xl p-4 sm:p-5 flex flex-col gap-3 animate-slide-up max-h-[85vh] overflow-y-auto border-t border-slate-800 shadow-2xl">
            <div class="w-10 h-1 bg-slate-700 rounded-full mx-auto"></div>
            <div class="flex justify-between items-start">
                <div class="flex items-center gap-2.5">
                    <img id="modalImg" src="https://images.unsplash.com/photo-1534528741775-53994a69daeb?w=150&auto=format&fit=crop&q=80" alt="Foto" class="w-14 h-14 rounded-full object-cover border-2 border-blue-600 shadow-md">
                    <div>
                        <h2 id="modalName" class="font-bold text-base leading-tight"></h2>
                        <p id="modalVilla" class="text-[11px] font-semibold text-blue-400 bg-blue-900/40 px-2 py-0.5 rounded-full inline-block mt-1"></p>
                    </div>
                </div>
                <button onclick="closeModal()" class="text-slate-400 hover:text-white p-1.5">
                    <i class="fa-solid fa-xmark text-lg"></i>
                </button>
            </div>

            <div class="border-t border-slate-800 pt-2.5 flex flex-col gap-2 text-xs">
                <div class="flex items-center justify-between bg-amber-950/40 border border-amber-800/60 p-2.5 rounded-xl">
                    <div class="flex items-center gap-2">
                        <i class="fa-solid fa-hotel text-amber-400"></i>
                        <span class="text-[10px] font-bold text-amber-300">Folio Opera (Four Seasons):</span>
                    </div>
                    <span id="modalOpera" class="font-mono font-bold text-amber-200 bg-amber-900/50 px-2 py-0.5 rounded">N/D</span>
                </div>

                <div class="flex items-start gap-2.5 bg-slate-800 p-2.5 rounded-xl border border-slate-700">
                    <i class="fa-solid fa-building text-blue-500 mt-0.5"></i>
                    <div>
                        <span class="text-[10px] font-bold text-slate-400 block">ZONA / TIPO DE MIEMBRO</span>
                        <p id="modalZone" class="text-slate-200"></p>
                    </div>
                </div>

                <div class="flex items-start gap-2.5 bg-slate-800 p-2.5 rounded-xl border border-slate-700">
                    <i class="fa-solid fa-golf-ball-tee text-blue-500 mt-0.5"></i>
                    <div>
                        <span class="text-[10px] font-bold text-slate-400 block">TRANSPORTE</span>
                        <p id="modalTransport" class="text-slate-200"></p>
                    </div>
                </div>

                <div class="flex items-start gap-2.5 bg-slate-800 p-2.5 rounded-xl border border-slate-700">
                    <i class="fa-solid fa-bell-concierge text-blue-500 mt-0.5"></i>
                    <div>
                        <span class="text-[10px] font-bold text-slate-400 block">NOTAS PARA BELLMAN</span>
                        <p id="modalBellman" class="text-slate-200"></p>
                    </div>
                </div>

                <div class="flex items-center gap-2.5 bg-slate-800 p-2.5 rounded-xl border border-slate-700">
                    <i class="fa-solid fa-phone text-blue-500"></i>
                    <div>
                        <span class="text-[10px] font-bold text-slate-400 block">TELÉFONO</span>
                        <p id="modalPhone" class="font-medium text-slate-200"></p>
                    </div>
                </div>
            </div>

            <div class="flex flex-col gap-2 pt-1">
                <button id="goToMapBtn" class="w-full bg-blue-600 text-white py-2.5 rounded-xl font-bold text-xs shadow-md hover:bg-blue-700 transition-colors flex items-center justify-center gap-2">
                    <i class="fa-solid fa-map-pin"></i> Ver ubicación en el Mapa GPS
                </button>

                <button id="goToGpsBtn" class="w-full bg-emerald-600 text-white py-2.5 rounded-xl font-bold text-xs shadow-md hover:bg-emerald-700 transition-colors flex items-center justify-center gap-2">
                    <i class="fa-solid fa-diamond-turn-right"></i> Iniciar Ruta en Google Maps
                </button>

                <button id="copyBtn" onclick="copyResidentDetails()" class="w-full bg-slate-800 text-white py-2.5 rounded-xl font-bold text-xs shadow-md hover:bg-slate-700 transition-colors flex items-center justify-center gap-2">
                    <i class="fa-solid fa-copy"></i> Copiar Datos para WhatsApp
                </button>
            </div>
        </div>
    </div>

    <!-- MODAL DE EMERGENCIAS -->
    <div id="emergencyModal" class="hidden fixed inset-0 bg-slate-950/80 backdrop-blur-sm z-50 flex items-center justify-center p-4">
        <div class="bg-slate-800 border border-slate-700 rounded-2xl p-5 max-w-sm w-full shadow-2xl flex flex-col gap-3">
            <div class="flex items-center justify-between border-b border-slate-700 pb-2">
                <h3 class="font-bold text-rose-400 text-sm flex items-center gap-2">
                    <i class="fa-solid fa-triangle-exclamation"></i> Directorio de Emergencia
                </h3>
                <button onclick="closeEmergencyModal()" class="text-slate-400 hover:text-white p-1">
                    <i class="fa-solid fa-xmark text-lg"></i>
                </button>
            </div>
            <p class="text-[11px] text-slate-300">Toca cualquier número para iniciar una llamada automática:</p>
            
            <div class="space-y-2 text-xs">
                <a href="tel:6241109088" class="flex items-center justify-between p-2.5 rounded-xl bg-slate-900 border border-slate-700 hover:border-rose-500 transition">
                    <div><div class="font-bold text-white">Paramédicos CP</div><div class="text-[11px] text-slate-400 font-mono">624 110 9088</div></div>
                    <i class="fa-solid fa-phone text-emerald-400 text-base"></i>
                </a>
                <a href="tel:6242384124" class="flex items-center justify-between p-2.5 rounded-xl bg-slate-900 border border-slate-700 hover:border-rose-500 transition">
                    <div><div class="font-bold text-white">Bomberos</div><div class="text-[11px] text-slate-400 font-mono">624 238 4124</div></div>
                    <i class="fa-solid fa-phone text-emerald-400 text-base"></i>
                </a>
                <a href="tel:6241781951" class="flex items-center justify-between p-2.5 rounded-xl bg-slate-900 border border-slate-700 hover:border-rose-500 transition">
                    <div><div class="font-bold text-white">Base Seguridad CP</div><div class="text-[11px] text-slate-400 font-mono">624 178 1951</div></div>
                    <i class="fa-solid fa-phone text-emerald-400 text-base"></i>
                </a>
                <a href="tel:6246890477" class="flex items-center justify-between p-2.5 rounded-xl bg-slate-900 border border-slate-700 hover:border-rose-500 transition">
                    <div><div class="font-bold text-white">Seguridad Four Seasons</div><div class="text-[11px] text-slate-400 font-mono">624 689 0477</div></div>
                    <i class="fa-solid fa-phone text-emerald-400 text-base"></i>
                </a>
                <a href="tel:6242371043" class="flex items-center justify-between p-2.5 rounded-xl bg-slate-900 border border-slate-700 hover:border-rose-500 transition">
                    <div><div class="font-bold text-white">Seguridad Amanvari</div><div class="text-[11px] text-slate-400 font-mono">624 237 1043</div></div>
                    <i class="fa-solid fa-phone text-emerald-400 text-base"></i>
                </a>
            </div>

            <button onclick="closeEmergencyModal()" class="w-full py-2 bg-slate-700 hover:bg-slate-600 text-white rounded-xl text-xs font-medium mt-1">Cerrar</button>
        </div>
    </div>

    <!-- MODAL DE CANALES DE RADIO -->
    <div id="radiosModal" class="hidden fixed inset-0 bg-slate-950/80 backdrop-blur-sm z-50 flex items-center justify-center p-4">
        <div class="bg-slate-800 border border-slate-700 rounded-2xl p-5 max-w-sm w-full shadow-2xl flex flex-col gap-3">
            <div class="flex items-center justify-between border-b border-slate-700 pb-2">
                <h3 class="font-bold text-amber-400 text-sm flex items-center gap-2">
                    <i class="fa-solid fa-walkie-talkie"></i> Canales de Radio Oficiales
                </h3>
                <button onclick="closeRadiosModal()" class="text-slate-400 hover:text-white p-1"><i class="fa-solid fa-xmark text-lg"></i></button>
            </div>
            
            <div class="space-y-1.5 text-xs max-h-[50vh] overflow-y-auto pr-1">
                <div class="flex justify-between p-2 rounded-lg bg-slate-900"><span class="font-bold text-amber-400">Canal 1</span><span class="text-slate-300">Seguridad</span></div>
                <div class="flex justify-between p-2 rounded-lg bg-slate-900"><span class="font-bold text-amber-400">Canal 2</span><span class="text-slate-300">Operación Golf</span></div>
                <div class="flex justify-between p-2 rounded-lg bg-slate-900"><span class="font-bold text-amber-400">Canal 3</span><span class="text-slate-300">Mantenimiento Golf</span></div>
                <div class="flex justify-between p-2 rounded-lg bg-slate-900"><span class="font-bold text-amber-400">Canal 4</span><span class="text-slate-300">Mozza</span></div>
                <div class="flex justify-between p-2 rounded-lg bg-slate-900"><span class="font-bold text-amber-400">Canal 5</span><span class="text-slate-300">Beach Club</span></div>
                <div class="flex justify-between p-2 rounded-lg bg-slate-900"><span class="font-bold text-amber-400">Canal 6</span><span class="text-slate-300">Marina / BH / Water Taxi</span></div>
                <div class="flex justify-between p-2 rounded-lg bg-slate-900"><span class="font-bold text-amber-400">Canal 7</span><span class="text-slate-300">Water Taxi</span></div>
                <div class="flex justify-between p-2 rounded-lg bg-slate-900"><span class="font-bold text-amber-400">Canal 8</span><span class="text-slate-300">Delphine</span></div>
                <div class="flex justify-between p-2 rounded-lg bg-slate-900"><span class="font-bold text-amber-400">Canal 9</span><span class="text-slate-300">Aman</span></div>
                <div class="flex justify-between p-2 rounded-lg bg-slate-900"><span class="font-bold text-amber-400">Canal 10</span><span class="text-slate-300">Libre (A&B / BH)</span></div>
                <div class="flex justify-between p-2 rounded-lg bg-slate-900"><span class="font-bold text-amber-400">Canal 11</span><span class="text-slate-300">911</span></div>
            </div>

            <button onclick="closeRadiosModal()" class="w-full py-2 bg-slate-700 hover:bg-slate-600 text-white rounded-xl text-xs font-medium mt-1">Cerrar</button>
        </div>
    </div>

    <!-- BOTTOM SHEET (Hoja Inferior para Sectores) -->
    <div id="bottomSheet" class="bottom-sheet fixed inset-x-0 bottom-0 z-40 bg-slate-800 border-t border-slate-700 rounded-t-3xl shadow-2xl max-h-[80vh] flex flex-col">
        <div class="w-full flex justify-center pt-2.5 pb-1 cursor-pointer" onclick="closeBottomSheet()">
            <div class="w-10 h-1 bg-slate-600 rounded-full"></div>
        </div>

        <div class="px-5 py-2 border-b border-slate-700/80 flex items-start justify-between">
            <div>
                <span id="sheetBadge" class="text-[10px] font-bold uppercase tracking-wider px-2.5 py-0.5 rounded text-slate-900 bg-amber-400 mb-1 inline-block">SECTOR</span>
                <h2 id="sheetTitle" class="text-base font-bold text-white">Título del Sector</h2>
                <p id="sheetSubtitle" class="text-[11px] text-slate-400">Subtítulo Descriptivo</p>
            </div>
            <button onclick="closeBottomSheet()" class="text-slate-400 hover:text-white p-1.5 rounded-xl hover:bg-slate-700 transition">
                <i class="fa-solid fa-xmark text-base"></i>
            </button>
        </div>

        <div class="grid grid-cols-3 border-b border-slate-700/60 bg-slate-900/60 p-1.5 text-xs gap-1">
            <button onclick="switchSectorTab('general')" id="tabBtn-general" class="tab-btn py-2 px-2 font-bold text-amber-400 bg-slate-800 rounded-xl text-center shadow transition">General</button>
            <button onclick="switchSectorTab('infrastructure')" id="tabBtn-infrastructure" class="tab-btn py-2 px-2 font-medium text-slate-300 bg-slate-800/40 rounded-xl text-center shadow transition">Bodegas / Villas</button>
            <button onclick="switchSectorTab('access')" id="tabBtn-access" class="tab-btn py-2 px-2 font-medium text-slate-300 bg-slate-800/40 rounded-xl text-center shadow transition">Accesos</button>
        </div>

        <div class="flex-1 overflow-y-auto p-4 space-y-2.5 text-xs no-scrollbar max-h-[45vh]" id="sheetBodyContent"></div>

        <div class="p-3 bg-slate-900/60 border-t border-slate-700/80 flex gap-2">
            <button onclick="copySectorInfo()" class="w-full py-2 px-3 rounded-xl bg-slate-700 hover:bg-slate-600 border border-slate-600 text-white font-medium text-xs flex items-center justify-center gap-2 transition">
                <i class="fa-regular fa-copy"></i>
                <span>Copiar Datos del Sector</span>
            </button>
        </div>
    </div>

    <!-- Toast Notification -->
    <div id="toast" class="fixed top-4 left-1/2 -translate-x-1/2 bg-emerald-600 text-white px-4 py-2 rounded-xl shadow-2xl flex items-center gap-2 text-xs font-semibold transform -translate-y-20 opacity-0 transition-all duration-300 z-50">
        <i class="fa-solid fa-circle-check text-base"></i>
        <span id="toastMsg">Acción realizada</span>
    </div>

    <!-- Leaflet JS -->
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <script>
        const SHEET_MEMBERS_URL = "https://docs.google.com/spreadsheets/d/e/2PACX-1vR_OhdIaJSzCgeEsYLDNBEtSRREK0oIa1Au8hX44vUp7At94uczbyWOT7NsIt5pHxs0ciVTDAoIbY9V/pub?gid=0&single=true&output=csv";
        const SHEET_COORD_URL   = "https://docs.google.com/spreadsheets/d/e/2PACX-1vR_OhdIaJSzCgeEsYLDNBEtSRREK0oIa1Au8hX44vUp7At94uczbyWOT7NsIt5pHxs0ciVTDAoIbY9V/pub?gid=796440013&single=true&output=csv";
        const SHEET_ORIGIN_URL  = "https://docs.google.com/spreadsheets/d/1vqHG6PWAUPcY5dSpV29Mk47iQfEnm36QY7k7fobL0wI/export?format=csv&gid=1281819744";

        let membersList = [];
        let originDictionary = {};
        let leafletMap = null;
        let mapMarkersMap = {};
        let currentZoneFilter = 'all';

        const OPERA_MAPPING = {
            "R1-11": "4011", "R1-12": "4012", "R1-21": "4021", "R1-22": "4022", "R1-31": "4031", "R1-32": "4032",
            "R2-11": "4211", "R2-12": "4212", "R2-13": "4213", "R2-14": "4214", "R2-15": "4215", "R2-16": "4216",
            "R2-21": "4221", "R2-22": "4222", "R2-23": "4223", "R2-24": "4224", "R2-25": "4225", "R2-26": "4226",
            "R2-31": "4231", "R2-32": "4232", "R2-33": "4233", "R2-34": "4234",
            "R5-11": "4411", "R5-12": "4412", "R5-21": "4421", "R5-22": "4422", "R5-31": "4431", "R5-32": "4432",
            "R6-11": "4611", "R6-12": "4612", "R6-21": "4621", "R6-22": "4622", "R6-31": "4631", "R6-32": "4632",
            "R7-11": "4811", "R7-12": "4812", "R7-21": "4821", "R7-22": "4822", "R7-31": "4831", "R7-32": "4832",
            "BFC-01": "5010", "BFC-02": "5020", "BFC-03": "5030", "BFC-04": "5040", "BFC-05": "5050", "BFC-06": "5060",
            "BFV-00": "5120", "BFV-0": "5130", "BFV-01": "5140", "BFV-02": "5150", "BFV-05": "5180", "BFV-06": "5190",
            "BFV-07": "5200", "BFV-09": "5220", "BFV-11": "5240", "BFV-12": "5250", "EV-14": "5270", "EV-15": "5280", "EV-17": "5300",
            "BCC-02": "5400", "BCC-03": "5410", "BCC-04": "5420", "BCC-05": "5430", "BCC-06": "5440", "BCC-07": "5450", "BCC-08": "5460",
            "OVV18": "5500", "OVV19": "5510", "OVV20": "5520", "OVV21": "5530", "OVV22": "5540", "OVV23": "5550", "OVV24": "5560",
            "SV-09": "5709", "SV-11": "5711", "SV-12": "5712", "SV-13": "5713", "SV-14": "5714", "SV-15": "5715", "SV-16": "5716", "SV-17": "5717", "SV-18": "5718",
            "R3-101": "6101", "R3-102": "6102", "R3-103": "6103", "R3-104": "6104", "R3-105": "6105", "R3-106": "6106", "R3-107": "6107", "R3-108": "6108", "R3-109": "6109", "R3-110": "6110", "R3-111": "6111", "R3-112": "6112",
            "R3-201": "6201", "R3-202": "6202", "R3-203": "6203", "R3-204": "6204", "R3-205": "6205", "R3-206": "6206", "R3-207": "6207", "R3-208": "6208", "R3-209": "6209", "R3-210": "6210", "R3-211": "6211", "R3-212": "6212",
            "R3-301": "6301", "R3-302": "6302", "R3-303": "6303", "R3-304": "6304", "R3-305": "6305", "R3-306": "6306", "R3-307": "6307", "R3-308": "6308", "R3-309": "6309", "R3-310": "6310", "R3-311": "6311", "R3-312": "6312",
            "MV-16": "7316", "MV-17": "7317", "MV-28": "7328"
        };

        function getOperaId(villa) {
            if (!villa) return "N/D";
            const clean = villa.trim().toUpperCase().replace(/[^A-Z0-9-]/g, '');
            for (const [key, val] of Object.entries(OPERA_MAPPING)) {
                if (clean.includes(key) || key.includes(clean)) return val;
            }
            return "N/D";
        }

        async function handleAuth() {
            const userInput = document.getElementById('auth-user').value.trim().toLowerCase();
            const passInput = document.getElementById('auth-pass').value.trim();
            const authBtn = document.getElementById('auth-btn');

            if (!userInput || !passInput) {
                alert('Por favor, ingresa tu usuario y contraseña.');
                return;
            }

            authBtn.innerText = "Verificando en Excel...";
            authBtn.disabled = true;

            try {
                const resOrigin = await fetch(SHEET_ORIGIN_URL);
                if (!resOrigin.ok) throw new Error("Error al conectar con la hoja de accesos.");
                
                const textOrigin = await resOrigin.text();
                const lines = textOrigin.split("\n");
                let authenticated = false;

                for (let i = 1; i < lines.length; i++) {
                    if (!lines[i].trim()) continue;
                    const cols = parseCSVLine(lines[i]);
                    const excelUser = (cols[0] || "").trim().toLowerCase();
                    const excelPass = (cols[1] || "").trim();

                    if (excelUser === userInput && excelPass === passInput) {
                        authenticated = true;
                        break;
                    }
                }

                if (authenticated) {
                    localStorage.setItem('cp_logged_user', userInput);
                    checkSession();
                } else {
                    alert('Usuario o contraseña incorrectos.');
                }
            } catch (error) {
                console.error("Error al validar el acceso:", error);
                alert('⚠️ Error de conexión: No se pudo verificar el acceso con la hoja de cálculo. Comprueba tu red.');
            } finally {
                authBtn.innerText = "Iniciar Sesión";
                authBtn.disabled = false;
            }
        }

        function checkSession() {
            const loggedUser = localStorage.getItem('cp_logged_user');
            const authScreen = document.getElementById('auth-screen');
            const badgeBtn = document.getElementById('userBadgeBtn');

            if (!loggedUser) {
                authScreen.style.display = 'flex';
            } else {
                authScreen.style.display = 'none';
                if (badgeBtn) {
                    badgeBtn.innerText = loggedUser.substring(0, 2).toUpperCase();
                }
                loadSheetData();
            }
        }

        function toggleAdminPanel() {
            const modal = document.getElementById('admin-modal');
            const isHidden = modal.classList.contains('hidden');
            
            if (isHidden) {
                modal.classList.remove('hidden');
                modal.classList.add('flex');
                
                const currentUser = localStorage.getItem('cp_logged_user') || 'operador';
                document.getElementById('profile-user').value = currentUser;

                const savedProfile = JSON.parse(localStorage.getItem('cp_profile_' + currentUser) || '{}');
                document.getElementById('profile-name').value = savedProfile.name || (currentUser.toUpperCase() + ' — Bellman & Security');
            } else {
                modal.classList.remove('flex');
                modal.classList.add('hidden');
            }
        }

        function saveProfileData() {
            const currentUser = localStorage.getItem('cp_logged_user');
            const name = document.getElementById('profile-name').value.trim();
            localStorage.setItem('cp_profile_' + currentUser, JSON.stringify({ name }));
            alert('¡Datos guardados exitosamente!');
            toggleAdminPanel();
        }

        function clearActiveSession() {
            if (confirm('¿Deseas cerrar la sesión activa?')) {
                localStorage.removeItem('cp_logged_user');
                location.reload();
            }
        }

        function cleanKey(str) {
            if (!str) return "";
            return str.toString().trim().toUpperCase().replace(/[^A-Z0-9]/g, '');
        }

        async function loadSheetData() {
            const syncEl = document.getElementById('syncStatus');
            const syncDot = document.getElementById('syncIndicatorDot');

            try {
                let coordsDictionary = {};
                try {
                    const resCoords = await fetch(SHEET_COORD_URL);
                    if (resCoords.ok) {
                        const textCoords = await resCoords.text();
                        coordsDictionary = parseCoordsCSV(textCoords);
                    }
                } catch(e) {
                    console.warn("Advertencia al cargar coordenadas.");
                }

                try {
                    const resOrigin = await fetch(SHEET_ORIGIN_URL);
                    if (resOrigin.ok) {
                        const textOrigin = await resOrigin.text();
                        originDictionary = parseOriginCSV(textOrigin);
                    }
                } catch(e) {
                    console.warn("Advertencia al cargar orígenes.");
                }

                const resMembers = await fetch(SHEET_MEMBERS_URL);
                if (!resMembers.ok) throw new Error("Error en respuesta de red de miembros");

                const textMembers = await resMembers.text();
                parseCSV(textMembers, coordsDictionary);

                const now = new Date();
                const timeString = now.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
                if (syncEl) syncEl.innerText = `Sincronizado (${timeString})`;
                if (syncDot) syncDot.className = "w-2 h-2 rounded-full bg-green-400";
            } catch (error) {
                console.error("Error al cargar datos:", error);
                const listEl = document.getElementById('residentList');
                if (listEl) listEl.innerHTML = `<div class="col-span-2 text-center py-10 text-red-400 text-sm">⚠️ Error al conectar con las hojas de cálculo. Verifique su conexión.</div>`;
                if (syncEl) syncEl.innerText = `Error de conexión`;
                if (syncDot) syncDot.className = "w-2 h-2 rounded-full bg-rose-500 animate-ping";
            }
        }

        function parseOriginCSV(text) {
            const lines = text.split("\n");
            const originMap = {};
            for (let i = 1; i < lines.length; i++) {
                if (!lines[i].trim()) continue;
                const cols = parseCSVLine(lines[i]);
                const rawName = cols[0] || "";
                const originVal = cols[1] || "";
                const key = cleanKey(rawName);
                if (key) originMap[key] = originVal;
            }
            return originMap;
        }

        function parseCoordsCSV(text) {
            const lines = text.split("\n");
            const coordsMap = {};
            for (let i = 1; i < lines.length; i++) {
                if (!lines[i].trim()) continue;
                const cols = parseCSVLine(lines[i]);
                const latVal = parseFloat(cols[1]);
                const lngVal = parseFloat(cols[2]);
                const rawName = cols[cols.length - 1] || cols[4] || "";
                const key = cleanKey(rawName);
                if (key && !isNaN(latVal) && !isNaN(lngVal)) {
                    coordsMap[key] = { lat: latVal, lng: lngVal };
                }
            }
            return coordsMap;
        }

        function findLocation(rawVilla, coordsMap) {
            if (!rawVilla) return null;
            const key = cleanKey(rawVilla);
            if (key.includes('MC10') || key.includes('M10L') || key.includes('M15L')) return null;
            if (key.startsWith('R1') || key.startsWith('R2') || key.startsWith('R3')) {
                return { lat: 23.6089788290, lng: -109.588890271 };
            }
            if (coordsMap[key]) return coordsMap[key];
            const buildingMatch = key.match(/^([A-Z]+\d+)/);
            if (buildingMatch && coordsMap[buildingMatch[1]]) return coordsMap[buildingMatch[1]];
            return null;
        }

        function parseCSV(text, coordsMap) {
            const lines = text.split("\n");
            membersList = [];
            const defaultLat = 23.609724386;
            const defaultLng = -109.5943928;

            for (let i = 1; i < lines.length; i++) {
                if (!lines[i].trim()) continue;
                const cols = parseCSVLine(lines[i]);

                if (cols.length >= 2 && cols[1]) {
                    const rawVilla = cols[3] || "";
                    let latVal = parseFloat(cols[5]);
                    let lngVal = parseFloat(cols[6]);

                    if (isNaN(latVal) || isNaN(lngVal)) {
                        const mapLoc = findLocation(rawVilla, coordsMap);
                        latVal = mapLoc ? mapLoc.lat : defaultLat;
                        lngVal = mapLoc ? mapLoc.lng : defaultLng;
                    }

                    const memberName = cols[1];
                    const memberKey = cleanKey(memberName);
                    const userOrigin = originDictionary[memberKey] || "N/D";

                    membersList.push({
                        id: cols[0] || i,
                        name: memberName,
                        zone: cols[2] || "Beach & Yacht Club",
                        villa: rawVilla || "N/D",
                        photo: cols[4] && cols[4].startsWith('http') ? cols[4] : "https://images.unsplash.com/photo-1534528741775-53994a69daeb?w=150&auto=format&fit=crop&q=80",
                        lat: latVal,
                        lng: lngVal,
                        transport: cols[7] || "Sin preferencias especiales de transporte.", 
                        bellman: cols[8] || "Sin notas específicas para bellman.",          
                        phone: cols[9] || "N/A",
                        origin: userOrigin,
                        opera: getOperaId(rawVilla)
                    });
                }
            }
            applyFilters();
        }

        function parseCSVLine(line) {
            const cols = [];
            let inQuotes = false;
            let current = '';
            for (let char of line) {
                if (char === '"') {
                    inQuotes = !inQuotes;
                } else if (char === ',' && !inQuotes) {
                    cols.push(current.trim().replace(/^"(.*)"$/, '$1'));
                    current = '';
                } else {
                    current += char;
                }
            }
            cols.push(current.trim().replace(/^"(.*)"$/, '$1'));
            return cols;
        }

        function renderDirectory(list) {
            const container = document.getElementById('residentList');
            if(!container) return;
            container.innerHTML = '';

            if(list.length === 0) {
                container.innerHTML = `<div class="col-span-2 text-center py-10 text-slate-500 text-sm">No se encontraron registros</div>`;
                return;
            }

            list.forEach(member => {
                const card = document.createElement('div');
                card.className = "p-3 rounded-2xl border shadow-sm flex flex-col items-center text-center cursor-pointer transition-all bg-slate-900 border-slate-800 hover:border-blue-500 text-slate-100 gap-2";
                card.onclick = () => openModal(member);

                card.innerHTML = `
                    <img src="${member.photo}" alt="${member.name}" class="w-16 h-16 rounded-full object-cover border-2 border-slate-700 shadow-md mt-1">
                    <div class="flex-1 flex flex-col items-center w-full">
                        <h3 class="font-bold text-xs line-clamp-1 w-full text-white">${member.name}</h3>
                        <span onclick="event.stopPropagation(); focusOnMember('${member.id}')" class="text-[11px] text-blue-400 font-semibold hover:underline inline-flex items-center gap-1 mt-0.5">
                            <i class="fa-solid fa-map-pin text-[9px]"></i> ${member.villa}
                        </span>
                        <span class="text-[9px] text-slate-400 bg-slate-800 px-2 py-0.5 rounded-md mt-1.5 truncate max-w-full">${member.zone}</span>
                    </div>
                `;
                container.appendChild(card);
            });
        }

        function filterByZone(zone) {
            currentZoneFilter = zone;
            const btnMap = {
                'all': 'btn-zone-all',
                'Beach & Yacht Club': 'btn-zone-beach',
                'Residencial': 'btn-zone-residential'
            };

            Object.keys(btnMap).forEach(z => {
                const btn = document.getElementById(btnMap[z]);
                if(btn) {
                    btn.className = (z === zone)
                        ? "px-3 py-1.5 rounded-full font-medium bg-blue-600 text-white shrink-0 shadow-sm transition-colors"
                        : "px-3 py-1.5 rounded-full font-medium bg-slate-900 text-slate-300 border border-slate-800 shrink-0 shadow-sm transition-colors";
                }
            });
            applyFilters();
        }

        function filterDirectory() {
            applyFilters();
        }

        function applyFilters() {
            const searchEl = document.getElementById('searchInput');
            const query = searchEl ? searchEl.value.toLowerCase() : '';
            
            const filtered = membersList.filter(m => {
                const matchesSearch = m.name.toLowerCase().includes(query) || 
                                      m.villa.toLowerCase().includes(query) || 
                                      m.zone.toLowerCase().includes(query);
                
                let matchesZone = true;
                if (currentZoneFilter !== 'all') {
                    matchesZone = m.zone.trim().toLowerCase() === currentZoneFilter.trim().toLowerCase();
                }
                return matchesSearch && matchesZone;
            });
            renderDirectory(filtered);
            if (leafletMap && document.getElementById('view-map-leaflet').classList.contains('hidden') === false) {
                updateLeafletMarkers(filtered);
            }
        }

        function focusOnMember(memberId) {
            const member = membersList.find(m => m.id == memberId);
            if (!member) return;

            closeModal();
            switchTab('map');

            setTimeout(() => {
                if (leafletMap) {
                    leafletMap.invalidateSize();
                    leafletMap.setView([member.lat, member.lng], 18);
                    updateLeafletMarkers([member], true);
                }
            }, 250);
        }

        function switchTab(tab) {
            const dirView = document.getElementById('view-directory');
            const mapView = document.getElementById('view-map-leaflet');
            const svgView = document.getElementById('view-map-svg');
            const sectorBar = document.getElementById('sectorBottomBar');
            
            const navDir = document.getElementById('nav-dir');
            const navMap = document.getElementById('nav-map');
            const navSvg = document.getElementById('nav-svg');

            if(dirView) dirView.classList.add('hidden');
            if(mapView) mapView.classList.add('hidden');
            if(svgView) svgView.classList.add('hidden');
            if(sectorBar) sectorBar.classList.add('hidden');

            const inactiveColor = "flex flex-col items-center text-slate-400 hover:text-slate-200 transition-colors";
            if(navDir) navDir.className = inactiveColor;
            if(navMap) navMap.className = inactiveColor;
            if(navSvg) navSvg.className = inactiveColor;

            if(tab === 'directory') {
                if(dirView) dirView.classList.remove('hidden');
                if(navDir) navDir.className = "flex flex-col items-center text-blue-500 transition-colors";
            } else if(tab === 'map') {
                if(mapView) mapView.classList.remove('hidden');
                if(navMap) navMap.className = "flex flex-col items-center text-blue-500 transition-colors";
                
                setTimeout(() => {
                    if(!leafletMap) {
                        initLeafletMap();
                    } else {
                        leafletMap.invalidateSize();
                        updateLeafletMarkers(membersList);
                    }
                }, 100);
            } else if(tab === 'svg') {
                if(svgView) svgView.classList.remove('hidden');
                if(sectorBar) sectorBar.classList.remove('hidden');
                if(navSvg) navSvg.className = "flex flex-col items-center text-blue-500 transition-colors";
            }
        }

        function initLeafletMap() {
            const mapContainer = document.getElementById('leaflet-map');
            if(!mapContainer) return;

            const standardLayer = L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
                maxZoom: 19,
                attribution: '&copy; OpenStreetMap'
            });

            const satelliteLayer = L.tileLayer('https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}', {
                maxZoom: 19,
                attribution: 'Esri'
            });

            leafletMap = L.map('leaflet-map', {
                center: [23.609724386, -109.5943928],
                zoom: 16,
                layers: [satelliteLayer]
            });

            L.control.layers({ "Estándar": standardLayer, "Satelital": satelliteLayer }).addTo(leafletMap);
            updateLeafletMarkers(membersList);
            
            setTimeout(() => { leafletMap.invalidateSize(); }, 200);
        }

        function updateLeafletMarkers(list) {
            if (!leafletMap) return;
            Object.values(mapMarkersMap).forEach(m => leafletMap.removeLayer(m));
            mapMarkersMap = {};

            list.forEach(member => {
                const marker = L.marker([member.lat, member.lng]).addTo(leafletMap);
                marker.bindPopup(`<b>${member.name}</b><br>Villa: ${member.villa}<br>Folio Opera: ${member.opera}<br>Origen: ${member.origin}<br>${member.zone}`);
                marker.on('click', () => openModal(member));
                mapMarkersMap[member.id] = marker;
            });
        }

        const CostaPalmasData = {
            'sec-1': {
                name: "Sector 1 — Infraestructura y Servicios",
                subtitle: "Área de Servicios Generales, Mantenimiento y Planta",
                badge: "SECTOR 1 (VERDE)",
                badgeClass: "bg-emerald-500 text-slate-950",
                accesos: [
                    { num: 1, desc: "Acceso 1: Caseta de Entrada Principal / Control General." },
                    { num: 2, desc: "Acceso 2: Filtro Secundario e Ingreso a Áreas Operativas." },
                    { num: 3, desc: "Acceso 3: Punto de Control Sector Servicios / Mantenimiento." },
                    { num: 4, desc: "Acceso 4: Cruce de Distribución Interna a Zona Central." }
                ],
                infrastructure: [
                    { title: "Lucha Libre (Outlet)", desc: "Punto operativo y gastronómico oficial." },
                    { title: "Agua Planta de Tratamiento (PTAR)", desc: "Infraestructura de servicios de agua y reciclaje." },
                    { title: "Bodega CP Homes & CP", desc: "Almacenaje de bienes inmuebles y suministros." },
                    { title: "Taller Habilitado HDOS & Beach Club", desc: "Talleres especializados de carpintería y playa." },
                    { title: "Bodega Península & Fourseasons", desc: "Resguardo logístico y herramientas." },
                    { title: "Talleres Golf y Jardinería", desc: "Mantenimiento de áreas verdes y maquinaria." },
                    { title: "Cuarto de Bombas - Garza", desc: "Sistema hidráulico principal." }
                ]
            },
            'sec-2': {
                name: "Sector 2 — Marina Villas & Yacht Club",
                subtitle: "Residencias de Marina y Zona Náutica Exclusiva",
                badge: "SECTOR 2 (AZUL)",
                badgeClass: "bg-blue-500 text-white",
                accesos: [
                    { num: 5, desc: "Acceso 5: Entrada Norte a Zona Marina Villas." },
                    { num: 6, desc: "Acceso 6: Acceso a Docks y Muelles Principales." }
                ],
                infrastructure: [
                    { title: "Delphine (Outlet)", desc: "Restaurante y punto de interés náutico principal." },
                    { title: "Beach Front Yacht Villas (Manzana 1)", desc: "Villas M1 con capacidad de amarre exclusivo." },
                    { title: "Marina Villas (Manzana 8)", desc: "MV8-00 hasta MV8-28 frente a la dársena." },
                    { title: "Marina Villas (Manzana 9)", desc: "MV09-08 hasta MV09-34." }
                ]
            },
            'sec-3': {
                name: "Sector 3 — Marina, Mozza & Casa Blake",
                subtitle: "Centro Urbano, Gastronomía y Club Náutico",
                badge: "SECTOR 3 (AMARILLO)",
                badgeClass: "bg-yellow-500 text-slate-950",
                accesos: [
                    { num: 7, desc: "Acceso 7: Punto de Control Casa Blake & Centro Comercial." },
                    { num: 8, desc: "Acceso 8: Ingreso a Restaurante Mozza / Docks Turísticos." }
                ],
                infrastructure: [
                    { title: "Restaurante Mozza (Outlet)", desc: "Área gastronómica italiana y espejos de agua." },
                    { title: "Vivero", desc: "Zona de producción y resguardo botánico." },
                    { title: "Casa Blake", desc: "Condominios y Boutique de estilo de vida." },
                    { title: "Muelles A, B, C y Muelle Mozza", desc: "Infraestructura para embarcaciones turísticas y privadas." }
                ]
            },
            'sec-4': {
                name: "Sector 4 — Four Seasons Resort & Residences",
                subtitle: "Zona Hotelera, Golf y Residencias de Lujo",
                badge: "SECTOR 4 (NARANJA)",
                badgeClass: "bg-orange-500 text-slate-950",
                accesos: [
                    { num: 9, desc: "Acceso 9: Acceso Principal al Lobby y Vialidad Copa de Oro." }
                ],
                infrastructure: [
                    { title: "Jimmy's & Bouchies (Outlets)", desc: "Cafetería y puntos gastronómicos exclusivos." },
                    { title: "The Beach House Pavilion (Outlet)", desc: "Centro de playa y amenidades principales." },
                    { title: "Lobby FS, Limón & Casa de Braza", desc: "Zona hotelera central y restaurantes insignia." },
                    { title: "Milos", desc: "Experiencia culinaria de alta gama." },
                    { title: "Ocean View & Estuary Villas", desc: "Manzanas residenciales panorámicas." }
                ]
            },
            'sec-aman': {
                name: "Sector Amán — Amanvari Resort & Residences",
                subtitle: "Zona Hotelera y Desarrollo Amanvari",
                badge: "ZONA AMÁN (PÚRPURA)",
                badgeClass: "bg-purple-500 text-white",
                accesos: [
                    { num: 10, desc: "Acceso 10: Ingreso a Obras y Módulo de Control Amanvari." },
                    { num: 11, desc: "Acceso 11: Acceso Poniente de Servicios Aman." }
                ],
                infrastructure: [
                    { title: "Luma & Arva", desc: "Restaurantes insignia de alta cocina en Amanvari." },
                    { title: "Sesui / Sensui & The Lounge", desc: "Espacios de alta coctelería y gastronomía asiática." },
                    { title: "Amanvari Resort & Pavilion", desc: "Desarrollo turístico de ultra-lujo." },
                    { title: "Puente Amán - Beach Club", desc: "Conexión vial principal." }
                ]
            }
        };

        const AccessDirectory = {
            1: { sector: 'sec-1', name: "Acceso 1" }, 2: { sector: 'sec-1', name: "Acceso 2" },
            3: { sector: 'sec-1', name: "Acceso 3" }, 4: { sector: 'sec-1', name: "Acceso 4" },
            5: { sector: 'sec-2', name: "Acceso 5" }, 6: { sector: 'sec-2', name: "Acceso 6" },
            7: { sector: 'sec-3', name: "Acceso 7" }, 8: { sector: 'sec-3', name: "Acceso 8" },
            9: { sector: 'sec-4', name: "Acceso 9" }, 10: { sector: 'sec-aman', name: "Acceso 10" },
            11: { sector: 'sec-aman', name: "Acceso 11" }
        };

        let currentActiveSec = null;
        let activeSectorTab = 'general';

        function selectSector(secId) {
            currentActiveSec = secId;
            const data = CostaPalmasData[secId];
            if (!data) return;

            document.querySelectorAll('.sector-polygon').forEach(p => p.classList.remove('active'));
            const poly = document.getElementById(`poly-${secId}`);
            if (poly) poly.classList.add('active');

            document.getElementById('sheetBadge').textContent = data.badge;
            document.getElementById('sheetBadge').className = `text-[10px] font-bold uppercase tracking-wider px-2.5 py-0.5 rounded inline-block ${data.badgeClass}`;
            document.getElementById('sheetTitle').textContent = data.name;
            document.getElementById('sheetSubtitle').textContent = data.subtitle;

            switchSectorTab('general');
            openBottomSheet();
        }

        function selectAccess(num) {
            const acc = AccessDirectory[num];
            if (!acc) return;
            selectSector(acc.sector);
            showToast(`Seleccionado ${acc.name}`);
        }

        function switchSectorTab(tabName) {
            activeSectorTab = tabName;
            
            ['general', 'infrastructure', 'access'].forEach(t => {
                const btn = document.getElementById(`tabBtn-${t}`);
                if (btn) {
                    if (t === tabName) {
                        btn.className = "tab-btn py-2 px-2 font-bold text-amber-400 bg-slate-800 rounded-xl text-center shadow transition";
                    } else {
                        btn.className = "tab-btn py-2 px-2 font-medium text-slate-300 bg-slate-800/40 rounded-xl text-center shadow transition";
                    }
                }
            });

            renderSectorTabContent();
        }

        function renderSectorTabContent() {
            if (!currentActiveSec) return;
            const data = CostaPalmasData[currentActiveSec];
            const container = document.getElementById('sheetBodyContent');

            if (activeSectorTab === 'general') {
                container.innerHTML = `
                    <div class="p-3 rounded-xl bg-slate-900/80 border border-slate-700/70">
                        <h4 class="font-bold text-amber-400 text-xs mb-1">Información General</h4>
                        <p class="text-slate-300 text-xs leading-relaxed">${data.subtitle}</p>
                    </div>
                    <div class="p-3 rounded-xl bg-slate-900/80 border border-slate-700/70">
                        <h4 class="font-bold text-amber-400 text-xs mb-1">Accesos Asignados</h4>
                        <p class="text-slate-300 text-xs">Este sector cuenta con ${data.accesos.length} puntos de control registrados.</p>
                    </div>
                `;
            } else if (activeSectorTab === 'infrastructure') {
                container.innerHTML = data.infrastructure.map(item => `
                    <div class="p-2.5 rounded-xl bg-slate-900/80 border border-slate-700/70">
                        <h5 class="font-bold text-amber-400 text-xs">${item.title}</h5>
                        <p class="text-slate-300 text-[11px] mt-0.5">${item.desc}</p>
                    </div>
                `).join('');
            } else if (activeSectorTab === 'access') {
                container.innerHTML = data.accesos.map(acc => `
                    <div class="p-2.5 rounded-xl bg-slate-900/80 border border-slate-700/70 flex items-center gap-2.5">
                        <div class="w-7 h-7 rounded-lg bg-amber-500/20 text-amber-400 font-bold flex items-center justify-center text-xs shrink-0 border border-amber-500/30">
                            <i class="fa-solid fa-shield-halved text-[10px]"></i>
                        </div>
                        <p class="text-slate-300 text-[11px]">${acc.desc}</p>
                    </div>
                `).join('');
            }
        }

        function openBottomSheet() { document.getElementById('bottomSheet').classList.add('open'); }
        function closeBottomSheet() {
            document.getElementById('bottomSheet').classList.remove('open');
            document.querySelectorAll('.sector-polygon').forEach(p => p.classList.remove('active'));
            currentActiveSec = null;
        }

        function openEmergencyModal() { document.getElementById('emergencyModal').classList.remove('hidden'); }
        function closeEmergencyModal() { document.getElementById('emergencyModal').classList.add('hidden'); }
        function openRadiosModal() { document.getElementById('radiosModal').classList.remove('hidden'); }
        function closeRadiosModal() { document.getElementById('radiosModal').classList.add('hidden'); }

        function copySectorInfo() {
            if (!currentActiveSec) return;
            const data = CostaPalmasData[currentActiveSec];
            let text = `${data.name}\n${data.subtitle}\n\nInfraestructura:\n`;
            data.infrastructure.forEach(i => text += `- ${i.title}: ${i.desc}\n`);

            navigator.clipboard.writeText(text).then(() => {
                showToast("Información copiada al portapapeles");
            });
        }

        function openModal(member) {
            const modal = document.getElementById('detailModal');
            if(!modal) return;

            document.getElementById('modalImg').src = member.photo;
            document.getElementById('modalName').innerText = member.name;
            document.getElementById('modalVilla').innerText = "Villa: " + member.villa + " | Origen: " + member.origin;
            document.getElementById('modalOpera').innerText = member.opera;
            document.getElementById('modalZone').innerText = member.zone;
            document.getElementById('modalTransport').innerText = member.transport;
            document.getElementById('modalBellman').innerText = member.bellman;
            document.getElementById('modalPhone').innerText = member.phone;
            
            document.getElementById('goToMapBtn').onclick = () => focusOnMember(member.id);
            document.getElementById('goToGpsBtn').onclick = () => {
                window.open(`https://www.google.com/maps/search/?api=1&query=${member.lat},${member.lng}`, '_blank');
            };

            modal.classList.remove('hidden');
        }

        function closeModal() {
            const modal = document.getElementById('detailModal');
            if(modal) modal.classList.add('hidden');
        }

        function copyResidentDetails() {
            const name = document.getElementById('modalName').innerText;
            const villa = document.getElementById('modalVilla').innerText;
            const opera = document.getElementById('modalOpera').innerText;
            const transport = document.getElementById('modalTransport').innerText;
            const bellman = document.getElementById('modalBellman').innerText;
            const phone = document.getElementById('modalPhone').innerText;

            const reportText = `[COSTA PALMAS — OPERACIÓN]\nResidente: ${name}\n${villa} | Folio Opera: ${opera}\nTransporte: ${transport}\nNotas Bellman: ${bellman}\nTel: ${phone}`;

            navigator.clipboard.writeText(reportText).then(() => {
                showToast("¡Datos copiados para WhatsApp!");
            });
        }

        function showToast(msg) {
            const toast = document.getElementById('toast');
            document.getElementById('toastMsg').textContent = msg;
            toast.classList.remove('-translate-y-20', 'opacity-0');
            setTimeout(() => {
                toast.classList.add('-translate-y-20', 'opacity-0');
            }, 2500);
        }

        document.addEventListener('DOMContentLoaded', () => {
            checkSession();
        });
    </script>
</body>
</html>
