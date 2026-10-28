<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Split the G - Dashboard</title>
    
    <!-- Tailwind CSS pour le style rapide et responsive -->
    <script src="https://cdn.tailwindcss.com"></script>
    
    <!-- Google Fonts pour un look technique -->
    <link href="https://fonts.googleapis.com/css2?family=Chakra+Petch:wght@400;600;700&family=JetBrains+Mono:wght@400;700&display=swap" rel="stylesheet">
    
    <!-- Icônes Phosphor -->
    <script src="https://unpkg.com/@phosphor-icons/web"></script>

    <script>
        tailwind.config = {
            theme: {
                extend: {
                    fontFamily: {
                        sans: ['Chakra Petch', 'sans-serif'],
                        mono: ['JetBrains Mono', 'monospace'],
                    },
                    colors: {
                        naval: {
                            900: '#0a192f', // Fond très sombre
                            800: '#112240', // Panneaux
                            700: '#233554', // Boutons
                            accent: '#64ffda', // Cyan technique
                            danger: '#ff3366', // Rouge urgence
                        }
                    }
                }
            }
        }
    </script>

    <style>
        /* Optimisations pour l'usage tactile (télécommande) */
        body {
            touch-action: manipulation;
            user-select: none;
            -webkit-user-select: none;
        }
        
        .panel {
            box-shadow: 0 10px 30px -10px rgba(0,0,0,0.5);
            border: 1px solid rgba(100, 255, 218, 0.1);
        }

        .btn-control {
            transition: all 0.1s ease;
            box-shadow: 0 4px 6px rgba(0,0,0,0.3);
        }
        
        .btn-control:active {
            transform: translateY(2px);
            box-shadow: 0 1px 2px rgba(0,0,0,0.3);
        }

        /* Jauge de vitesse */
        .speed-bar-container {
            height: 150px;
            width: 20px;
            background: #233554;
            border-radius: 10px;
            position: relative;
            overflow: hidden;
        }
        
        .speed-bar-fill {
            position: absolute;
            width: 100%;
            background: #64ffda;
            transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
        }

        /* Animation du gouvernail */
        #rudder-blade {
            transform-origin: top center;
            transition: transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
        }
    </style>
</head>
<body class="bg-naval-900 text-slate-300 min-h-screen flex flex-col">

    <!-- Header -->
    <header class="bg-naval-800 border-b border-naval-700 p-4 flex justify-between items-center z-10">
        <div class="flex items-center gap-3">
            <i class="ph-fill ph-anchor text-naval-accent text-3xl"></i>
            <div>
                <h1 class="text-xl font-bold text-white uppercase tracking-wider">Split the G</h1>
                <p class="text-xs text-naval-accent font-mono">DASHBOARD DE CONTRÔLE</p>
            </div>
        </div>
        <div class="flex items-center gap-2">
            <span class="relative flex h-3 w-3">
              <span class="animate-ping absolute inline-flex h-full w-full rounded-full bg-naval-accent opacity-75"></span>
              <span class="relative inline-flex rounded-full h-3 w-3 bg-naval-accent"></span>
            </span>
            <span class="text-xs font-mono text-naval-accent uppercase">Simulateur</span>
        </div>
    </header>

    <!-- Main Content -->
    <main class="flex-grow p-4 flex flex-col md:flex-row gap-4 max-w-6xl mx-auto w-full">
        
        <!-- TELEMETRY PANEL (Left) -->
        <section class="panel bg-naval-800 rounded-2xl p-6 flex-1 flex flex-col">
            <h2 class="text-sm font-bold text-slate-400 uppercase tracking-widest mb-6 border-b border-naval-700 pb-2">
                <i class="ph ph-activity mr-2"></i>Télémétrie
            </h2>
            
            <div class="flex justify-around items-center flex-grow">
                
                <!-- Speed Gauge -->
                <div class="flex flex-col items-center gap-4">
                    <div class="text-center">
                        <div class="text-xs text-slate-400 font-bold mb-1">MOTEUR</div>
                        <div id="speed-display" class="font-mono text-3xl text-white">0</div>
                    </div>
                    
                    <div class="flex gap-4 items-center">
                        <div class="text-xs text-slate-500 font-mono">AV<br>255</div>
                        <div class="speed-bar-container relative">
                            <!-- Ligne du zéro -->
                            <div class="absolute w-full h-[2px] bg-slate-400 top-1/2 left-0 z-10"></div>
                            <!-- Remplissage dynamique -->
                            <div id="speed-bar" class="speed-bar-fill top-1/2 h-0"></div>
                        </div>
                        <div class="text-xs text-slate-500 font-mono">AR<br>-255</div>
                    </div>
                </div>

                <!-- Direction Gauge -->
                <div class="flex flex-col items-center gap-4">
                    <div class="text-center">
                        <div class="text-xs text-slate-400 font-bold mb-1">CAP (BARRE)</div>
                        <div class="font-mono text-3xl text-white flex items-center justify-center gap-1">
                            <span id="angle-display">0</span><span class="text-lg text-naval-accent">°</span>
                        </div>
                    </div>
                    
                    <!-- Top-down boat & rudder SVG -->
                    <div class="relative w-32 h-32 flex items-center justify-center bg-naval-900 rounded-full border border-naval-700">
                        <svg width="60" height="100" viewBox="0 0 60 100" class="overflow-visible">
                            <!-- Coque du bateau -->
                            <path d="M30 10 C45 20, 50 40, 50 70 L10 70 C10 40, 15 20, 30 10 Z" fill="none" stroke="#233554" stroke-width="3"/>
                            <circle cx="30" cy="70" r="3" fill="#64ffda" />
                            <!-- Axe du gouvernail -->
                            <g id="rudder-blade" style="transform: rotate(0deg);">
                                <rect x="28" y="70" width="4" height="30" fill="#64ffda" rx="2" />
                            </g>
                        </svg>
                    </div>
                    
                    <div class="text-center">
                        <div class="text-xs text-slate-400 font-bold">TRIM</div>
                        <div id="trim-display" class="font-mono text-lg text-slate-300">0</div>
                    </div>
                </div>
            </div>
        </section>

        <!-- COMMANDS PANEL (Right) -->
        <section class="panel bg-naval-800 rounded-2xl p-6 flex-1 flex flex-col gap-4">
            <h2 class="text-sm font-bold text-slate-400 uppercase tracking-widest mb-2 border-b border-naval-700 pb-2">
                <i class="ph ph-game-controller mr-2"></i>Contrôles
            </h2>

            <!-- Moteur Controls -->
            <div class="bg-naval-900 p-4 rounded-xl border border-naval-700">
                <div class="text-xs text-center text-slate-400 mb-3">CONTRÔLE MOTEUR (±30)</div>
                <div class="flex justify-center gap-4">
                    <button onclick="throttle(-1)" class="btn-control bg-naval-700 hover:bg-slate-600 text-white w-20 h-16 rounded-xl flex items-center justify-center text-2xl">
                        <i class="ph-bold ph-caret-down"></i>
                    </button>
                    <button onclick="throttle(1)" class="btn-control bg-naval-accent text-naval-900 font-bold w-20 h-16 rounded-xl flex items-center justify-center text-2xl">
                        <i class="ph-bold ph-caret-up"></i>
                    </button>
                </div>
            </div>

            <!-- Barre & Trim Controls -->
            <div class="bg-naval-900 p-4 rounded-xl border border-naval-700">
                <div class="text-xs text-center text-slate-400 mb-3">DIRECTION (±5°) & TRIM (±1)</div>
                
                <div class="grid grid-cols-2 gap-4">
                    <!-- Bâbord (Left) -->
                    <div class="flex flex-col items-center gap-2">
                        <span class="text-[10px] text-red-400 font-bold uppercase tracking-wider">Bâbord</span>
                        <button onclick="steer(-1)" class="btn-control bg-naval-700 hover:bg-slate-600 text-white w-full h-16 rounded-xl flex items-center justify-center text-2xl">
                            <i class="ph-bold ph-caret-left"></i>
                        </button>
                        <button onclick="adjustTrim(-1)" class="btn-control bg-transparent border border-naval-700 text-slate-400 w-full py-1 rounded-lg text-xs uppercase hover:bg-naval-700">
                            Trim -
                        </button>
                    </div>

                    <!-- Tribord (Right) -->
                    <div class="flex flex-col items-center gap-2">
                        <span class="text-[10px] text-green-400 font-bold uppercase tracking-wider">Tribord</span>
                        <button onclick="steer(1)" class="btn-control bg-naval-700 hover:bg-slate-600 text-white w-full h-16 rounded-xl flex items-center justify-center text-2xl">
                            <i class="ph-bold ph-caret-right"></i>
                        </button>
                        <button onclick="adjustTrim(1)" class="btn-control bg-transparent border border-naval-700 text-slate-400 w-full py-1 rounded-lg text-xs uppercase hover:bg-naval-700">
                            Trim +
                        </button>
                    </div>
                </div>
            </div>

            <!-- Emergency Stop -->
            <button onclick="emergencyStop()" class="btn-control mt-auto bg-naval-danger hover:bg-red-500 text-white w-full py-4 rounded-xl font-bold uppercase tracking-widest flex items-center justify-center gap-2">
                <i class="ph-bold ph-warning-octagon text-2xl"></i>
                Arrêt d'urgence
            </button>

        </section>
    </main>

    <!-- JavaScript Logic (Mirrors micro:bit logic) -->
    <script>
        // --- CONSTANTES ---
        const ANGLE_MAX = 45;
        const ANGLE_MIN = -45;
        const INCREMENT_BARRE = 5;
        const INCREMENT_MOTEUR = 30;
        const VITESSE_MAX = 255;
        
        // --- ETAT DU BATEAU ---
        let state = {
            angleBarre: 0,
            vitesse: 0,
            trim: 0
        };

        // --- FONCTIONS UTILITAIRES ---
        function clamp(val, min, max) {
            return Math.min(Math.max(val, min), max);
        }

        // --- ACTIONS ---
        function steer(direction) {
            // direction = 1 (Tribord/Right) ou -1 (Bâbord/Left)
            state.angleBarre += (direction * INCREMENT_BARRE);
            state.angleBarre = clamp(state.angleBarre, ANGLE_MIN, ANGLE_MAX);
            updateUI();
        }

        function throttle(direction) {
            // direction = 1 (Accélérer) ou -1 (Décélérer/Reculer)
            state.vitesse += (direction * INCREMENT_MOTEUR);
            state.vitesse = clamp(state.vitesse, -VITESSE_MAX, VITESSE_MAX);
            updateUI();
        }

        function adjustTrim(direction) {
            state.trim += direction;
            updateUI();
        }

        function emergencyStop() {
            state.vitesse = 0;
            state.angleBarre = 0;
            updateUI();
            
            // Petit effet visuel
            document.body.style.backgroundColor = '#300';
            setTimeout(() => { document.body.style.backgroundColor = ''; }, 150);
        }

        // --- MISE A JOUR DE L'INTERFACE VISUELLE ---
        function updateUI() {
            // 1. Textes
            document.getElementById('angle-display').textContent = state.angleBarre;
            document.getElementById('speed-display').textContent = state.vitesse;
            document.getElementById('trim-display').textContent = state.trim > 0 ? `+${state.trim}` : state.trim;

            // 2. Animation du gouvernail (incluant le trim visuellement)
            const angleTotal = state.angleBarre + state.trim;
            const rudder = document.getElementById('rudder-blade');
            // On inverse le signe visuellement car tourner la barre à gauche (bâbord) 
            // fait pivoter le safran vers la gauche visuellement sur l'écran
            rudder.style.transform = `rotate(${-angleTotal}deg)`;

            // 3. Jauge de vitesse (Barre verticale)
            const speedBar = document.getElementById('speed-bar');
            
            // La jauge fait 150px de haut. Le zéro est au milieu (50%).
            if (state.vitesse >= 0) {
                // Marche avant (remplit vers le haut)
                // 255 = 50% de la jauge (la moitié haute)
                const percentage = (state.vitesse / VITESSE_MAX) * 50; 
                speedBar.style.top = `${50 - percentage}%`;
                speedBar.style.height = `${percentage}%`;
                speedBar.style.backgroundColor = '#64ffda'; // Cyan
            } else {
                // Marche arrière (remplit vers le bas)
                const percentage = (Math.abs(state.vitesse) / VITESSE_MAX) * 50;
                speedBar.style.top = '50%';
                speedBar.style.height = `${percentage}%`;
                speedBar.style.backgroundColor = '#ff3366'; // Rouge
            }
        }

        // Initialisation au chargement
        window.onload = function() {
            updateUI();
        };

        // Support clavier pour tester sur PC
        document.addEventListener('keydown', (e) => {
            if (e.key === 'ArrowUp') throttle(1);
            if (e.key === 'ArrowDown') throttle(-1);
            if (e.key === 'ArrowLeft') steer(-1);
            if (e.key === 'ArrowRight') steer(1);
            if (e.key === ' ') emergencyStop(); // Espace pour l'arrêt d'urgence
        });
    </script>
</body>
</html>
