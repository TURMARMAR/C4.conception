<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Split the G - Télécommande</title>
    <style>
        :root { --fond: #0b1319; --carte: #14222c; --cyan: #00e5ff; --vert: #2ecc71; --rouge: #ff4757; }
        * { box-sizing: border-box; }
        body {
            font-family: monospace; background: var(--fond); color: var(--cyan);
            margin: 0; padding: 20px; min-height: 100vh;
            display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 16px;
            touch-action: manipulation; overscroll-behavior-y: none; user-select: none;
        }
        h2 { margin: 0; letter-spacing: 2px; text-transform: uppercase; font-size: 1.3rem; }
        .bloc { width: 100%; max-width: 420px; }
        .carte { background: var(--carte); border: 2px solid var(--cyan); border-radius: 12px; padding: 14px 16px; }
        .entete, .reperes { display: flex; justify-content: space-between; align-items: center; }
        .entete { letter-spacing: 2px; margin-bottom: 4px; }
        .entete strong { font-size: 1.3rem; }
        .reperes { font-size: .75rem; opacity: .8; }

        button {
            background: var(--carte); color: var(--cyan); border: 2px solid var(--cyan); border-radius: 12px;
            padding: 16px 10px; font: bold 1.05rem monospace; cursor: pointer;
        }
        button:active:not(:disabled) { background: var(--cyan); color: var(--fond); }
        button:disabled { opacity: .4; }
        #btn-connect.ok { color: var(--vert); border-color: var(--vert); background: rgba(46, 204, 113, .1); opacity: 1; }
        #btn-centre { padding: 8px 14px; font-size: .8rem; }
        #btn-stop { padding: 24px 10px; color: var(--rouge); border-color: var(--rouge); background: rgba(255, 71, 87, .1); }
        #btn-stop:active { background: var(--rouge); color: #fff; }

        /* Curseurs : grosse poignée pour le pouce */
        input[type=range] { -webkit-appearance: none; appearance: none; width: 100%; height: 48px; margin: 0; background: transparent; }
        input[type=range]:disabled { opacity: .4; }
        input[type=range]::-webkit-slider-runnable-track {
            height: 14px; border-radius: 7px; background: var(--fond); box-shadow: inset 0 0 0 2px var(--cyan);
        }
        input[type=range]::-webkit-slider-thumb {
            -webkit-appearance: none; width: 44px; height: 44px; margin-top: -15px;
            border-radius: 50%; background: var(--cyan); border: 4px solid var(--fond);
        }
    </style>
</head>
<body>

    <h2>Split the G</h2>
    <button id="btn-connect" class="bloc">🔌 CONNECTER</button>

    <div class="bloc carte">
        <div class="entete"><span>DIRECTION</span><strong id="aff-direction">Centre</strong></div>
        <input type="range" id="curseur-direction" min="45" max="135" step="5" value="90" disabled>
        <div class="reperes"><span>◄ BÂBORD</span><button id="btn-centre" disabled>CENTRER</button><span>TRIBORD ►</span></div>
    </div>

    <div class="bloc carte">
        <div class="entete"><span>VITESSE</span><strong id="aff-vitesse">0 %</strong></div>
        <input type="range" id="curseur-vitesse" min="0" max="100" step="5" value="0" disabled>
        <div class="reperes"><span>0</span><span>100 %</span></div>
    </div>

    <button id="btn-stop" class="bloc">ARRÊT D'URGENCE</button>

    <script>
        const UART_SERVICE_UUID = "6e400001-b5a3-f393-e0a9-e50e24dcca9e";
        const UART_RX_CHAR_UUID = "6e400002-b5a3-f393-e0a9-e50e24dcca9e";
        const ANGLE_CENTRE = 90;  // même valeur que dans le code micro:bit
        const PAUSE_MS = 40;      // délai mini entre deux envois Bluetooth

        const $ = id => document.getElementById(id);
        const btnConnect = $('btn-connect'), btnCentre = $('btn-centre'), btnStop = $('btn-stop');
        const curseurDirection = $('curseur-direction'), curseurVitesse = $('curseur-vitesse');
        let txCharacteristic = null;

        // --- ENVOI BLUETOOTH ---
        // Un seul message à la fois, et seule la DERNIÈRE valeur de chaque curseur est envoyée :
        // on peut glisser le doigt vite sans saturer la liaison.
        // Protocole : "D<angle>" direction, "V<pourcentage>" vitesse, "S" arrêt.
        const aEnvoyer = new Map();   // canal ('S', 'V', 'D') -> dernier message en attente
        let envoiEnCours = false;

        function envoyer(canal, message) {
            aEnvoyer.set(canal, message);
            viderFile();
        }

        async function viderFile() {
            if (envoiEnCours || !txCharacteristic) return;
            envoiEnCours = true;
            try {
                while (aEnvoyer.size > 0 && txCharacteristic) {
                    const canal = ['S', 'V', 'D'].find(c => aEnvoyer.has(c));  // l'arrêt passe toujours en premier
                    const message = aEnvoyer.get(canal);
                    aEnvoyer.delete(canal);
                    await txCharacteristic.writeValue(new TextEncoder().encode(message + "\n"));
                    await new Promise(r => setTimeout(r, PAUSE_MS));
                }
            } catch (e) {
                console.log("Erreur d'envoi :", e);
            } finally {
                envoiEnCours = false;
            }
        }

        // --- AFFICHAGE ---
        function majAffichage() {
            const ecart = Number(curseurDirection.value) - ANGLE_CENTRE;
            $('aff-direction').textContent = ecart === 0 ? "Centre" : (ecart < 0 ? "Bâbord " : "Tribord ") + Math.abs(ecart) + "°";
            $('aff-vitesse').textContent = curseurVitesse.value + " %";
        }

        function etatConnexion(connecte) {
            btnConnect.disabled = connecte;
            btnConnect.classList.toggle('ok', connecte);
            btnConnect.textContent = connecte ? "✅ CONNECTÉ" : "🔌 CONNECTER";
            curseurDirection.disabled = curseurVitesse.disabled = btnCentre.disabled = !connecte;
        }

        function remiseAZero() {
            curseurDirection.value = ANGLE_CENTRE;
            curseurVitesse.value = 0;
            majAffichage();
        }

        // --- COMMANDES ---
        curseurDirection.addEventListener('input', () => {
            majAffichage();
            envoyer('D', 'D' + curseurDirection.value);
        });

        curseurVitesse.addEventListener('input', () => {
            majAffichage();
            envoyer('V', 'V' + curseurVitesse.value);
        });

        btnCentre.addEventListener('click', () => {
            curseurDirection.value = ANGLE_CENTRE;
            majAffichage();
            envoyer('D', 'D' + ANGLE_CENTRE);
        });

        // pointerdown : réagit dès le contact du doigt, sans attendre le relâchement
        btnStop.addEventListener('pointerdown', () => {
            curseurVitesse.value = 0;
            majAffichage();
            aEnvoyer.delete('V');   // inutile d'envoyer une ancienne vitesse après l'arrêt
            envoyer('S', 'S');
        });

        // --- CONNEXION ---
        btnConnect.addEventListener('click', async () => {
            try {
                const appareil = await navigator.bluetooth.requestDevice({
                    filters: [{ namePrefix: 'BBC micro:bit' }],
                    optionalServices: [UART_SERVICE_UUID]
                });
                appareil.addEventListener('gattserverdisconnected', () => {
                    txCharacteristic = null;
                    aEnvoyer.clear();
                    remiseAZero();
                    etatConnexion(false);
                });
                const server = await appareil.gatt.connect();
                const service = await server.getPrimaryService(UART_SERVICE_UUID);
                txCharacteristic = await service.getCharacteristic(UART_RX_CHAR_UUID);

                remiseAZero();
                etatConnexion(true);
                envoyer('V', 'V0');                       // micro:bit et curseurs repartent d'accord
                envoyer('D', 'D' + ANGLE_CENTRE);
            } catch (error) {
                if (error.name !== 'NotFoundError') alert("Erreur de connexion : " + error);  // NotFoundError = choix annulé
            }
        });

        majAffichage();
    </script>
</body>
</html>
