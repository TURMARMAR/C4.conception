<!DOCTYPE html>
<html lang="fr">
    
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Split the G - Commande</title>
    <style>
        :root {
            --cyan: #00e5ff;
            --bg-color: #0b1319;
            --panel-bg: #14222c;
            --danger: #ff4757;
            --success: #2ecc71;
        }
        
      body {
            font-family: 'Courier New', Courier, monospace;
            background-color: var(--bg-color);
            color: var(--cyan);
            margin: 0;
            padding: 15px;
            display: flex;
            flex-direction: column;
            height: 100vh;
            box-sizing: border-box;
            overscroll-behavior-y: none; 
        }
        
     h1 {
            text-align: center;
            font-size: 1.2rem;
            margin-top: 0;
            text-transform: uppercase;
            letter-spacing: 2px;
        }
        
     /* --- Télémétrie --- */
        .telemetry {
            display: flex;
            justify-content: space-around;
            background: var(--panel-bg);
            padding: 15px;
            border-radius: 12px;
            border: 1px solid rgba(0, 229, 255, 0.2);
            text-align: center;
        }
        
     .data-label { font-size: 0.8rem; opacity: 0.8; }
     .data-value { font-size: 1.8rem; font-weight: bold; margin-top: 5px; }
     
       /* --- Visualisation du bateau --- */
      .radar-view {
            flex-grow: 1;
            display: flex;
            justify-content: center;
            align-items: center;
            position: relative;
            min-height: 120px;
        }
        
       .hull {
            width: 30px;
            height: 80px;
            border: 2px solid var(--cyan);
            border-radius: 50% 50% 10px 10px;
            position: absolute;
        }
        
     .rudder {
            width: 4px;
            height: 40px;
            background-color: var(--cyan);
            position: absolute;
            top: 50%;
            transform-origin: top center;
            transform: rotate(0deg);
            transition: transform 0.2s ease-out;
        }
        
       /* --- Grille de Contrôle --- */
        .controls {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 15px;
            margin-bottom: 10px;
        }
        
     .control-col {
            display: flex;
            flex-direction: column;
            gap: 15px;
        }
        
        button {
            background: var(--panel-bg);
            color: var(--cyan);
            border: 1px solid var(--cyan);
            border-radius: 12px;
            padding: 25px 10px;
            font-size: 1.1rem;
            font-family: inherit;
            font-weight: bold;
            touch-action: manipulation; 
            user-select: none;
            cursor: pointer;
            transition: background 0.1s, transform 0.1s;
        }
        
       button:active {
            background: var(--cyan);
            color: var(--bg-color);
            transform: scale(0.95);
        }
        
        .btn-stop {
            grid-column: span 2;
            background: rgba(255, 71, 87, 0.1);
            border-color: var(--danger);
            color: var(--danger);
            padding: 20px;
            font-size: 1.2rem;
        }
        
     .btn-stop:active {
            background: var(--danger);
            color: white;
        }
        
       .trim-controls {
            grid-column: span 2;
            display: flex;
            justify-content: space-between;
            align-items: center;
            background: var(--panel-bg);
            border-radius: 12px;
            padding: 5px;
        }
        
        .trim-controls button {
            padding: 15px 20px;
            font-size: 1rem;
            border: none;
        }
        
    </style>
    
</head>
<body>

  <h1>Split the G - Télécommande</h1>
    <button id="btn-connect" class="btn-stop" style="background: rgba(0, 229, 255, 0.1); border-color: var(--cyan); color: var(--cyan); margin-bottom: 15px;">🔌 CONNECTER LE BATEAU</button>
    
  <div class="telemetry">
        <div>
            <div class="data-label">VITESSE</div>
            <div class="data-value" id="ui-vitesse">0</div>
        </div>
        <div>
            <div class="data-label">BARRE</div>
            <div class="data-value" id="ui-barre">0°</div>
        </div>
    </div>
    <div class="radar-view">
        <div class="hull"></div>
        <div class="rudder" id="ui-safran"></div>
    </div>
    <div class="controls">
        <div class="control-col">
            <button onclick="moteur(1)">AVANT</button>
            <button onclick="moteur(-1)">ARRIÈRE</button>
        </div>
             <div class="control-col">
            <button onclick="barre(-1)">BÂBORD</button>
            <button onclick="barre(1)">TRIBORD</button>
        </div>
     <button class="btn-stop" onclick="arretUrgence()">ARRÊT D'URGENCE</button>
    <div class="trim-controls">
            <button onclick="ajusterTrim(-1)">TRIM -</button>
            <div>TRIM: <span id="ui-trim">0</span></div>
            <button onclick="ajusterTrim(1)">TRIM +</button>
        </div>
    </div>

  <script>
        const ANGLE_MAX = 45;
        const ANGLE_MIN = -45;
        const INC_BARRE = 5;
        const INC_MOTEUR = 30;
        const VITESSE_MAX = 255;

        let angleBarre = 0;
        let vitesse = 0;
        let trim = 0;

        const UART_SERVICE_UUID = "6e400001-b5a3-f393-e0a9-e50e24dcca9e";
        const UART_RX_CHAR_UUID = "6e400003-b5a3-f393-e0a9-e50e24dcca9e";
        let txCharacteristic;

        document.getElementById('btn-connect').addEventListener('click', async () => {
            try {
                const device = await navigator.bluetooth.requestDevice({
                    filters: [{ namePrefix: 'BBC micro:bit' }],
                    optionalServices: [UART_SERVICE_UUID]
                });

                const server = await device.gatt.connect();
                const service = await server.getPrimaryService(UART_SERVICE_UUID);
                txCharacteristic = await service.getCharacteristic(UART_RX_CHAR_UUID);
                
                const btnConnect = document.getElementById('btn-connect');
                btnConnect.innerText = "✅ NAVIRE CONNECTÉ";
                btnConnect.style.color = "var(--success)";
                btnConnect.style.borderColor = "var(--success)";
                btnConnect.style.background = "rgba(46, 204, 113, 0.1)";
                
                transmettreTrames();

            } catch (error) {
                console.error("Erreur Bluetooth :", error);
                alert("Impossible de se connecter : " + error);
            }
        });

        // --- CORRECTION APPLIQUÉE ICI : Utilisation du Dièse (#) ---
        async function envoyerBluetooth(texte) {
            if (!txCharacteristic) return;
            const encoder = new TextEncoder();
            try {
                // On ajoute un "#" à la fin du message pour que la carte le comprenne
                await txCharacteristic.writeValue(encoder.encode(texte + "#"));
            } catch (e) {
                console.log("Erreur de transmission :", e);
            }
        }

        async function transmettreTrames() {
    let angleFinal = angleBarre + trim;
    // On attend que l'angle soit bien envoyé
    await envoyerBluetooth("B" + angleFinal);
    // Optionnel mais recommandé : petite pause de respiration pour le Micro:bit
    await new Promise(resolve => setTimeout(resolve, 20)); 
    // On envoie la vitesse
    await envoyerBluetooth("V" + vitesse);
}
        
        function actualiserAffichage() {
            document.getElementById('ui-vitesse').innerText = vitesse;
            document.getElementById('ui-barre').innerText = angleBarre + '°';
            document.getElementById('ui-trim').innerText = trim;

            let rotationTotale = (angleBarre + trim) * -1;
            document.getElementById('ui-safran').style.transform = `rotate(${rotationTotale}deg)`;
            
            transmettreTrames();
        }
        
   function barre(direction) {
            angleBarre += (direction * INC_BARRE);
            if (angleBarre > ANGLE_MAX) angleBarre = ANGLE_MAX;
            if (angleBarre < ANGLE_MIN) angleBarre = ANGLE_MIN;
            actualiserAffichage();
  }

   function moteur(direction) {
            vitesse += (direction * INC_MOTEUR);
            if (vitesse > VITESSE_MAX) vitesse = VITESSE_MAX;
            if (vitesse < -VITESSE_MAX) vitesse = -VITESSE_MAX;
            actualiserAffichage();
   }

   function ajusterTrim(direction) {
            trim += direction;
            actualiserAffichage();
   }

   function arretUrgence() {
            vitesse = 0;
            angleBarre = 0;
            actualiserAffichage();
            envoyerBluetooth("S");
  }
    // Envoie les commandes toutes les secondes pour nourrir le chien de garde du Micro:bit
  setInterval(() => {
        if (txCharacteristic) {
        transmettreTrames();
        }
}, 1000);
        document.addEventListener('keydown', function(e) {
            if (e.key === 'ArrowUp') moteur(1);
            if (e.key === 'ArrowDown') moteur(-1);
            if (e.key === 'ArrowLeft') barre(-1);
            if (e.key === 'ArrowRight') barre(1);
            if (e.key === ' ') arretUrgence();
        });

        actualiserAffichage();
    </script>
</body>
</html>
